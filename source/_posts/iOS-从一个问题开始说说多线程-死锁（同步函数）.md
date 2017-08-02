---
title: iOS 从一个问题开始说说多线程+死锁（同步函数）
date: 2017-07-22
categories:

- 多线程

tags: 
- 多线程
- 死锁
- 同步函数

---

#引子：
>如果我现在问你GCD里，队列+执行函数的组合怎么产生子线程。
你的回答是异步函数+并行队列吗？

<!--more-->
#目的
写这篇文章不是说我对多线程+加锁理解的有多透彻，我只是喜欢对国内博客里面讲到的一些东西进行一些验证。不知道大家有没有一个感触，百度每个技术点的时候，往往你看到的博客全部是一个模子，全部是抄来抄去，很多作者写东西的时候往往不加验证，这样导致了对于一些知识点的误解。比如说上面那个问题。

#验证

说到GCD，最基本的就是四个队列，两个函数
####四种队列：
1. 串行队列
```
dispatch_queue_t q = dispatch_queue_create("aaaaa", DISPATCH_QUEUE_SERIAL);
```
2. 并行队列
```
dispatch_queue_t q = dispatch_queue_create("cccccc", DISPATCH_QUEUE_CONCURRENT);
```
3. 全局并行队列
```
dispatch_get_global_queue(0, 0)
```
4. 主队列
```
dispatch_get_main_queue()
```
但实际上这四个队列可以归结为串行+并行，因为主队列就是一种串行队列（关于这一点我待会回来进行验证），全局并行队列实际上是并行队列。

####两个函数：
1. 同步函数

```
    dispatch_sync(q, ^{
    });
```

2. 异步函数

```
    dispatch_async(q, ^{
    });
```

下面我会将队列和函数进行两两组合，然后进行调试

```
//同步派遣+串行队列
- (void)func1{
    dispatch_queue_t q = dispatch_queue_create("aaaaa", DISPATCH_QUEUE_SERIAL);    
    dispatch_sync(q, ^{
        NSLog(@"1111");
    });
    NSLog(@"22222");
}

//异步派遣+串行队列
- (void)func2{
    dispatch_queue_t q = dispatch_queue_create("bbbbbb", DISPATCH_QUEUE_SERIAL);
    dispatch_async(q, ^{
        NSLog(@"1111");
    });
    NSLog(@"22222");
}

//同步派遣+并行队列
- (void)func3{
    dispatch_queue_t q = dispatch_queue_create("cccccc", DISPATCH_QUEUE_CONCURRENT);
    dispatch_sync(q, ^{
        NSLog(@"1111");
    });
    NSLog(@"22222");
}

//异步派遣+并行队列
- (void)func4{
    dispatch_queue_t q = dispatch_queue_create("ddddddd", DISPATCH_QUEUE_CONCURRENT);
    dispatch_async(q, ^{
        NSLog(@"1111");
    });
    NSLog(@"22222");
}
```

#结论
####1. 主队列是串行队列
现在程序运行在主线程中，这个主线程在主队列中
![AC451239-799E-44AD-8FFB-C3700808B0FF.png](http://upload-images.jianshu.io/upload_images/1743443-9b46ce2d2efe6854.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
Queue:com.apple.main-thred(serial)
```

```com.apple.main-thred``` 实际上是队列的别名
```serial``` 是串行队列的意思

####2.  同步派遣+串行队列，不产生子线程

这个截图侧面的佐证了1的结论，而且我们观察到这时候，程序还是运行在Thread1中。这里要说明的是不一定Thread1就是主线程，我们需要观察调用栈的方法，如截图所示```UIApplication,UIViewController，UIWindow```的各种初始化方法都在这个线程里面执行，所以我们判断出这里是主线程，其他的所有线程都是子线程。程序还在主线程中，所以这里不产生子线程。

![186052A6-74E5-4305-A625-63529D9F54C2.png](http://upload-images.jianshu.io/upload_images/1743443-f61508cb6a98322d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

####3.  异步派遣+串行队列，产生子线程
这个结论就是文章最初的提问的关键所在，很多人认为这里不会产生子线程，但是实际上是产生了子线程的。
如下面所示，这时候程序员已经来到了Thread3。

![113F1EC4-09CE-4672-B1FC-CE898F9409F7.png](http://upload-images.jianshu.io/upload_images/1743443-c88d6044f7606738.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

####4.  同步派遣+并行队列，没有产生子线程

![07A6F515-800E-4F7E-BE15-7170670CDEF4.png](http://upload-images.jianshu.io/upload_images/1743443-cf496b57809661fb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

####5.  异步派遣+并行队列，产生子线程

![162F6773-024E-4D38-B53F-104A7C3689CD.png](http://upload-images.jianshu.io/upload_images/1743443-d262efb66c08a687.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

####6. 总结一下
单独谈论队列和同步异步函数都不合适，只有组合的时候，才有讨论的意义。
通过上面的例子，是否产生子线程很大程度取决于执行函数。
异步派遣能够将队列调度到别的子线程上面去
同步派遣能够将队列调度到当前线程（注意是当前线程，不一定是主线程）

所以上面的问题的答案是：异步函数+并行队列／异步函数+串行队列 都会产生子线程。

#另外一个问题：死锁
问下面函数的执行结果
![3B9AF8C9-6D33-44A5-9AA0-11064ED80AE2.png](http://upload-images.jianshu.io/upload_images/1743443-5c566edbe333aa87.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![3171DA0A-DF9E-43D8-B433-4570DCBA3396.png](http://upload-images.jianshu.io/upload_images/1743443-924ae17b35825f57.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里如果我将22222上面的更改为``` dispatch_sync(q, ^{``` ，请问执行结果是什么？

![C9A71300-A1E3-462A-96D9-623DBA93CAEA.png](http://upload-images.jianshu.io/upload_images/1743443-30257eead41f2c84.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

答案是执行到这里崩溃了，我们来分析一下，为什么这里会崩溃？很多博客上面都有说会崩溃
但是理由很牵强。大概是下面截图这个意思。

![75E517E2-24C2-4B11-AFB5-ABA1D1334560.png](http://upload-images.jianshu.io/upload_images/1743443-71ebc9a5e55e23a7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

老实说，这个说法很牵强。看这段我自己都不能说服我自己，下面是唯一能说服我的帖子，是从源码的角度来说的。
[一个曲高和寡语言水平很捉急的大神给出的源码](http://www.jianshu.com/p/3684f40c9172)
但是他的分析太难看懂了，主要是因为语文水平很捉急。这里我帮大家总结一下。

实际上GCD里面关于实现同步队列使用到的是信号量，模拟出来大概是这样子的
```
@synchronized (self) {
                if (self.array.count <= 0) {
                    continue;
                }
                
                NSNumber *number = [self.array lastObject];
                if ([number boolValue]) {
                    
                    //同步事件.  同步派遣的事件->synsc block里面需要被执行的
                    //一个同步事件可以分为两部分.  第一wait queue,  第二, 派发事件.
                    //此处为了方便,  两行代码位置是反的.  但是由于async 派遣, 本身就拼到了队列末尾.  所以
                    //从实际执行角度,   顺序是没有问题的.   完整模拟了.   dispatch源码中对事件的执行模式和 同步派遣到本身队列的死锁问题.
                    dispatch_async(queue, ^{
                        dispatch_semaphore_signal(semaphore);
                    });
                    
                    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
                    
                    [self.array removeLastObject];
                    NSLog(@"\n");
                    NSLog(@"消费了  同步!!!!!!事件");
                    NSLog(@"\n");
                    syncEvent = NO;
                    
                }else{
                    //普通消费.
                    dispatch_async(dispatch_get_global_queue(0, 0), ^{
                        @synchronized (self) {
                            [self.array removeLastObject];
                            NSLog(@"消费了一个异步事件.");
                        }
                    });
                }
            }

```

想要看懂上面这段代码，你需要反复理解下面几句话，我当时想了一下午才想通。
1. 串行队列里面只可能有一个线程，并行队列里面可能有多个。
2.  队列里面可能没有线程，线程总是跑来跑去的。
3.  不管是同步函数或者是异步函数，都会将block里面的内容派遣到对应的队列的最下面。
4. 同步函数里面维护了一套信号量，信号量的single操作被套在异步函数里面
```
 dispatch_async(queue, ^{
        dispatch_semaphore_signal(semaphore);
 });
```
5. 同步函数的的其他操作在同步函数所处的外面的队列里面去执行，只有
```
        dispatch_semaphore_signal(semaphore);
```
在同步函数锁包裹的队列里面去执行。

综上所述，这行代码将信号量的++事件放到了queue队列的最后。如果同步函数外面没有对queue做派遣动作，不会死锁。

我们将这些理论放到实际例子里面去解释
```
- (void)viewDidLoad {
    [super viewDidLoad];
    dispatch_sync(dispatch_get_main_queue(), ^{
        NSLog(@"1111");
    });

    NSLog(@"222");
}
```
外面是主队列，主队列里面是主线程，同步函数将``` NSLog(@"1111");``` 压倒主队列的底部了。里外都是主队列，事件顺序执行。
同步函数底层按照顺序异步函数将```dispatch_semaphore_signal(semaphore);```
压在queue（主队列）的最下面

```
 dispatch_async(queue, ^{
        dispatch_semaphore_signal(semaphore);
 });
```
然后执行
```
dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
```
信号量--为-1，阻塞当前线程

，这样程序员永远都执行不到```dispatch_semaphore_signal(semaphore)```

#思考
这个例子不知道大家看懂了没有，是不是感觉这样分析的话，只要使用同步函数都会被死锁。
其实同步函数的底层下面这个函数能够执行到线程就不会被锁住
```
 dispatch_async(queue, ^{
        dispatch_semaphore_signal(semaphore);
 });
```
如何能执行```dispatch_semaphore_signal(semaphore);```
信号量的++操作被压在queue的最下面了，只要同步函数的执行的queue和外面的queue不一致，这里就会被执行了。

看这个例子，我们将同步函数派遣的主队列换成一个新的串行队列
```
- (void)viewDidLoad {
    [super viewDidLoad];
    dispatch_sync(dispatch_queue_create("1111", DISPATCH_QUEUE_SERIAL), ^{
        NSLog(@"1111");
    });

    NSLog(@"222");
}
```

这样同步函数的底层在主队列里面执行下面信号量--
```
dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
```
主队列里面执行异步函数+串行队列，将信号量++压在串行队列的底部，串行队列唯一事件为信号量++函数，于是执行解锁。
```
 dispatch_async(queue, ^{
        dispatch_semaphore_signal(semaphore);
 });
```
这样就不会造成死锁了。

#再来一个例子
```
    dispatch_queue_t q = dispatch_queue_create("1111111", DISPATCH_QUEUE_SERIAL);
    
    dispatch_sync(q, ^{
        
        NSLog(@"11111");
        dispatch_sync(q, ^{
            NSLog(@"22222");
        });
        NSLog(@"33333");
    });
    
    NSLog(@"44444");
    NSLog(@"5555");
```

第一层同步函数处在主队列里面，
同步函数的底层主队列里面执行
```
dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
```
在新的串行队列里面执行
```
 dispatch_async(queue, ^{
        dispatch_semaphore_signal(semaphore);
 });
```
这两步操作分开在不同的队列里面，都能执行，所以不会死锁

第二层同步函数处在q队列里面
同步函数的底层q串行队列里面执行，锁住了q队列，q队列无法执行其他事件
```
dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
```
在q串行队列里面执行，被压入队列事件中，但是因为信号量锁住线程gg
```
 dispatch_async(queue, ^{
        dispatch_semaphore_signal(semaphore);
 });
```

真的很绕很绕，但是是从一个源码的角度来看的。当然你们想看源码，会更蒙
![2263264-5b92f9b728dfa869.png](http://upload-images.jianshu.io/upload_images/1743443-f4ab76fe7b6f2ee9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

[一个曲高和寡的哥们写的源码分析](http://www.jianshu.com/p/3684f40c9172)

[demo地址](https://github.com/knightSaber/CBTMultithreadingDemo)