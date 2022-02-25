## Problems

1. Dealloc 中使用 weak_self
2. 线程与计时器问题



#### 一、Dealloc 中使用 weak_self

```objective-c
- (void)dealloc{
    __weak __typeof(self) weak_self = self;
    NSLog(@"%@",weak_self);
}
```

1. 会 crash , 并报错误为 `"Cannot form weak reference to instance (%p) of " "class %s. It is possible that this object was " "over-released, or is in the process of deallocation."`

2. 原因是这么写会在 `dealloc` 的时候会调用 `storeWeak` , 进行注册弱引用表, 此时会判断是否是正在释放阶段并且如果在在释放就 crash。

   ```objective-c
   /** 
    * Registers a new (object, weak pointer) pair. Creates a new weak (注册一个新的(对象，弱指针)对。创建一个新的弱点)
    * object entry if it does not exist.
    * 
    * @param weak_table The global weak table. (全局弱引用表)
    * @param referent The object pointed to by the weak reference. (弱引用所指向的对象)
    * @param referrer The weak pointer address. (弱指针地址)
    */
   id 
   weak_register_no_lock(weak_table_t *weak_table, id referent_id, 
                         id *referrer_id, bool crashIfDeallocating) {
       objc_object *referent = (objc_object *)referent_id;
       objc_object **referrer = (objc_object **)referrer_id;
   
       if (!referent  ||  referent->isTaggedPointer()) return referent_id;
   
       // ensure that the referenced object is viable （确保被引用的对象是可行的）
       bool deallocating;
       if (!referent->ISA()->hasCustomRR()) {
           deallocating = referent->rootIsDeallocating();
       }
       else {
           BOOL (*allowsWeakReference)(objc_object *, SEL) = 
               (BOOL(*)(objc_object *, SEL))
               object_getMethodImplementation((id)referent, 
                                              SEL_allowsWeakReference);
           if ((IMP)allowsWeakReference == _objc_msgForward) {
               return nil;
           }
           deallocating =
               ! (*allowsWeakReference)(referent, SEL_allowsWeakReference);
       }
   
       if (deallocating) {
           if (crashIfDeallocating) {
               _objc_fatal("Cannot form weak reference to instance (%p) of "
                           "class %s. It is possible that this object was "
                           "over-released, or is in the process of deallocation.",
                           (void*)referent, object_getClassName((id)referent));
           } else {
               return nil;
           }
       }
   
       // now remember it and where it is being stored （现在记住它和它被存储在哪里）
       weak_entry_t *entry;
       if ((entry = weak_entry_for_referent(weak_table, referent))) {
           append_referrer(entry, referrer);
       } 
       else {
           weak_entry_t new_entry(referent, referrer);
           weak_grow_maybe(weak_table);
           weak_entry_insert(weak_table, &new_entry);
       }
   
       // Do not set *referrer. objc_storeWeak() requires that the 
       // value not change.
   
       return referent_id;
   }
   ```

   

#### 二、线程与计时器问题

##### 1.

```objective-c
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    dispatch_queue_t queue = dispatch_queue_create(0, 0);
    dispatch_async(queue, ^{
        NSLog(@"1");
        [self performSelector:@selector(test) withObject:nil afterDelay:0];
        NSLog(@"3");
    });
}

- (void)test{
    NSLog(@"2");
}

// 打印结果：1  3

#pragma mark - 去掉 afterDelay:0
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    dispatch_queue_t queue = dispatch_queue_create(0, 0);
    dispatch_async(queue, ^{
        NSLog(@"1");
        [self performSelector:@selector(test) withObject:nil];
        NSLog(@"3");
    });
}

/*
   打印结果：1  2  3
   
   首先我们要知道[self performSelector:@selector(test2) withObject:nil];这里的执行本质就是objc_msgSend(self,@selector(test2)),再说白点就是[self test2];(这里可以查看objc源码的实现,直接查找performSelector的方法实现就能找到).
   
   而[self performSelector:@selector(test2) withObject:nil afterDelay:.0];就有点不一样的,大家可以把上面的异步线程的里面的代码全部拿到主线程来操作,你会发现都是有输出的.会输出1,2,3(顺序可能不一样,但都会输出).这时候你可以点击进去看里面的定义,你会发现它是属于RunLoop里面的模块,而且在objc里面你是找不到源码的.这时候我们就可以参照上个博客说的GNUstep的源码
 */

// 我们看源码如下:
- (void)performSelector:(SEL)aSelector 
             withObject:(id)argument 
             afterDelay:(NSTimeInterval)seconds{
    NSRunLoop *loop = [NSRunLoop currentRunLoop];
    GSTimedPerformer *item = [[GSTimedPerformer alloc] initWithSelector:aSelector 
                                                                target:argument
                                                                 delay:seconds];
    [[loop _timedPerformers] addObject:item];
    RELEASE(item);
    [loop addTimer: item->timer forModel:NSDefaultRunLoopMode];
}

/*
   所以很明显,它的底层是用了NSTimer定时器.比如时间我写2.0就是2秒以后做事情.而我们知道定时器是要加在Runloop里面.这下就很清楚了,为什么主线程可以,因为我们知道主线程的Runloop是默认开启的,子线程是默认没有Runloop的,所以会出现这种情况.定时器没法工作,它那段代码是往runloop中添加了一个定时器任务.如果想让它工作,我们只要添加runloop即可.
*/
```

##### 2.

```objective-c
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    NSThread *thread = [[NSThread alloc] initWithBlock:^{
        NSLog(@"1");
    }];
    [thread start];
    [self performSelector:@selector(test) onThread:thread withObject:nil waitUntilDone:YES];
}

- (void)test{
    NSLog(@"2");
}

/*
 * 输出结果：1   然后崩溃
   输出1以后,执行performSelector任务的时候,直接崩溃了,因为我们知道,调用start,它就会执行block任务,执行完以后,线程的任务就算完成了,线程自动会退出,这时候你去让一个已经退出的线程安排任务,肯定会出问题.
 */
```

##### 3.

```objective-c
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    self.thread = [[NSThread alloc] initWithBlock:^{
        NSLog(@"1");
    }];
    [self.thread start];
    [self performSelector:@selector(test) onThread:self.thread withObject:nil waitUntilDone:YES];
}

- (void)test{
    NSLog(@"2");
}

/*
 * 输出结果：1   然后崩溃
 
   第三道也可以说是第二道的升华,因为有些人可能会想,既然thread会退出,那我就用一个强指针指着,不让它退出,不就行了吗?真的可以吗?
   
   这是因为,强指针只是让它不销毁,让这个线程对象还是在内存中,但是它不能做事情了,它已经没有用了,而只有添加RunLoop才是保证线程处于激活的状态,保证它的生命周期不结束,还能继续在这个线程中执行线程任务.所以必须使用RunLoop才能保住线程的生命周期.
 */


- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    self.thread = [[NSThread alloc] initWithBlock:^{
        NSLog(@"1");
        [[NSRunLoop currentRunLoop] addPort:[NSPort new] forMode:NSDefaultRunLoopMode];
        [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
    }];
    [self.thread start];
    [self performSelector:@selector(test) onThread:self.thread withObject:nil waitUntilDone:YES];
}

- (void)test{
    NSLog(@"2");
}

/*
 * 输出结果：1   2
 */


- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    NSThread *thread1 = [[NSThread alloc] initWithBlock:^{
        NSLog(@"1");
        [[NSRunLoop currentRunLoop] addPort:[NSPort new] forMode:NSDefaultRunLoopMode];
        [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
    }];
    [thread1 start];
    [self performSelector:@selector(test) onThread:thread1 withObject:nil waitUntilDone:YES];
}

- (void)test{
    NSLog(@"2");
}

/*
 * 输出结果：1   2
 */


- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    NSThread *thread1 = [[NSThread alloc] initWithBlock:^{
        NSLog(@"1");
        [[NSRunLoop currentRunLoop] addPort:[NSPort new] forMode:NSDefaultRunLoopMode];
        [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
    }];
    [thread1 start];
    [self performSelector:@selector(test) onThread:thread1 withObject:nil waitUntilDone:YES];
    [self performSelector:@selector(test) onThread:thread1 withObject:nil waitUntilDone:YES];
}

- (void)test{
    NSLog(@"2");
}
/*
 * 输出结果：1   2   然后崩溃
 
   为什么会有上面的结果呢?原因它还是关于Runloop的一个线程保活的问题(也就是控制线程的生命周期)
 */


- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    self.thread = [[NSThread alloc] initWithBlock:^{
        NSLog(@"1");
        [[NSRunLoop currentRunLoop] addPort:[NSPort new] forMode:NSDefaultRunLoopMode];
        [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
    }];
    [self.thread start];
    [self performSelector:@selector(test) onThread:self.thread withObject:nil waitUntilDone:YES];
    [self performSelector:@selector(test) onThread:self.thread withObject:nil waitUntilDone:YES];
}

- (void)test{
    NSLog(@"2");
}

/*
 * 输出结果：1   2                                                                                                
 */
```

