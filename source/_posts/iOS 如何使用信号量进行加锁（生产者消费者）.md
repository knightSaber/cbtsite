---
title: iOS 如何使用信号量进行加锁（生产者消费者）
date: 2017-07-25
categories:

- 多线程

tags: 
- 生产者消费者
- 锁
- 信号量

---


#引子：
上一篇文章讲到了队列组，从而引出信号量。下面来结合一个实际的例子来玩玩信号量。
<!--more-->

#文章概要
1.  生产者，消费者
2. 如何使用信号量加锁

#生产者，消费者

我对于生产者和消费者的理解是：需要有一个缓存池，生产者和消费者需要在不同的线程中去分别操作缓存池，这时候就特别容易产生并发问题。

下面讲解如何去写一个消费者和生产者

缓存池：其实需要一个可变的容器，所有在oc里面可变数组即可
```
- (NSMutableArray *)array{
    if (!_array) {
        _array = [NSMutableArray array];
    }
    return  _array;
}
```
生产者：需要在子线程去生产
```
//生产者
- (void)producerFunc{
    
    __block int count = 0;
    
    //生产者生成数据
    dispatch_queue_t t = dispatch_queue_create("222222", DISPATCH_QUEUE_CONCURRENT);
    
    dispatch_async(t, ^{
        while (YES) {
            count++;
            int t = random()%10;
            [self.array addObject:[NSString stringWithFormat:@"%zd",t]];
            NSLog(@"生产了%zd",count);
        }
    });
}
```

消费者：需要在子线程去消费
```
//消费者
- (void)consumerFunc{
    
    __block int count = 0;
    
    //消费者消费数据
    
    dispatch_queue_t t1 = dispatch_queue_create("11111", DISPATCH_QUEUE_CONCURRENT);
    
    dispatch_async(t1, ^{
        
        while (YES) {
            if (self.array.count > 0) {
                count++;
                [self.array removeLastObject];
                NSLog(@"消费了%zd",count);
            }
        }
    }); 
}
```

上面就是关于消费者，生产者的写法，但是这种写法会带来一个很严重的问题，就是当两个线程同时去竞争一个资源的时候，程序容易出现问题。所以这时候我们需要加锁去解决这个问题。

```
- (dispatch_semaphore_t)semaphore{
    if (!_semaphore) {
        _semaphore = dispatch_semaphore_create(1);
    }
    return _semaphore;
}
```

信号量为负数的时候，对当前线程进行阻塞，我们这里设置为1，是提供给两个线程操作的时候，放第一个线程去工作，阻塞第二个线程。

```
//生产者
- (void)producerFunc{
    
    __block int count = 0;
    
    //生产者生成数据
    dispatch_queue_t t = dispatch_queue_create("222222", DISPATCH_QUEUE_CONCURRENT);
    
    dispatch_async(t, ^{
        while (YES) {
            count++;
            int t = random()%10;
            dispatch_semaphore_wait(self.semaphore, DISPATCH_TIME_FOREVER);
            [self.array addObject:[NSString stringWithFormat:@"%zd",t]];
            dispatch_semaphore_signal(self.semaphore);
            NSLog(@"生产了%zd",count);
            
        }
    });
}

//消费者
- (void)consumerFunc{
    
    __block int count = 0;
    
    //消费者消费数据
    
    dispatch_queue_t t1 = dispatch_queue_create("11111", DISPATCH_QUEUE_CONCURRENT);
    
    dispatch_async(t1, ^{
        
        while (YES) {
            if (self.array.count > 0) {
                count++;
                dispatch_semaphore_wait(self.semaphore, DISPATCH_TIME_FOREVER);
                [self.array removeLastObject];
                dispatch_semaphore_signal(self.semaphore);
                NSLog(@"消费了%zd",count);
            }
        }
    });
}
```

上面当生产者执行
 ```dispatch_semaphore_wait(self.semaphore, DISPATCH_TIME_FOREVER);``` 的时候，这时候信号量进行--为0.继续执行下面操作，当消费者执行 ```dispatch_semaphore_wait(self.semaphore, DISPATCH_TIME_FOREVER);```，信号量为-1了，阻塞当前线程，只有当生产者执行完毕这时候信号量++为0，消费者继续执行，性质完毕之后信号量++为1。之后也是按照这个模式进行循环。
[文章demo](https://github.com/knightSaber/CBTLockDemo)

如果解决文章对你有帮助，请给我的github  star 一下，不胜感激