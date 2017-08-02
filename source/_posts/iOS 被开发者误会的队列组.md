---
title: iOS 被开发者误会的队列组
date: 2017-07-16
categories:

- 多线程

tags: 
- 队列组
- 信号量

---

#引子：
>了解过GCD的同学可能对队列组不会陌生，但是我认识的程序员里面有1/3的认为这个队列组能够解决线程同步问题。还有1/3的程序员不知道不知道。
dispatch_group_enter,dispatch_group_leave函数是什么。

<!--more-->


这就是我想要写这篇文章的目的。

#文章要点：
1. 队列组如何实现多个子线程的同步。
2. 队列组的几种写法。
3. 信号量的几种玩法。

#阅读下面代码，是否看这很熟悉，我相信不少人是从这段代码开始学习GCD的用法的。

```
// 创建一个队列组!
    dispatch_group_t group = dispatch_group_create();
    __block UIImage *image1 ,*image2;
    // 下载第一张图片
    dispatch_group_async(group, dispatch_get_global_queue(0, 0), ^{
        image1 = [self downloadImageWithUrlString:@"http://g.hiphotos.baidu.com/image/pic/item/95eef01f3a292df54e0e7e08be315c6035a873da.jpg"];
    });
    
    // 下载第二张图片
    dispatch_group_async(group, dispatch_get_global_queue(0, 0), ^{
        
        image2 = [self downloadImageWithUrlString:@"http://e.hiphotos.baidu.com/image/pic/item/cc11728b4710b912d4bb69ffc1fdfc03924522bc.jpg"];
        
    });
    
    // 合并图片并且显示
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        
       // NSLog(@"显示图片! %@",[NSThread currentThread]);
        
        // 合并图片
        UIImage *image = [self bingImageWithImage1:image1 Image2:image2];
        
        // 显示合并之后的图片!
        self.imageView.image = image;
    });

```

上面的代码很有误导性，下载一张图片和第二张图片的方法都是在子线程里面去执行的，然后最后合并的时候能够拿到合并在一起的图片。当看到最终演示的效果的时候，大家心满意足的觉得自己已经学会了队列组。

但是我们忽略了一个问题就是下面这个封装的代码是同步的还是异步的，如果这段代码是同步的，那么队列组就毫无意义
```
[self downloadImageWithUrlString:@"http://e.hiphotos.baidu.com/image/pic/item/cc11728b4710b912d4bb69ffc1fdfc03924522bc.jpg"];
```
我们看到下面的方法，在我断点调试的时候发现下面代码全部都是在主线程里面执行的
![C9EB368D-AFDE-4FF7-AD4C-4ACBABA5DE77.png](http://upload-images.jianshu.io/upload_images/1743443-7536af929eabb1e3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![8B968DE8-3F57-4400-8CF9-EF8C53EFE2C2.png](http://upload-images.jianshu.io/upload_images/1743443-5e94d792f6164412.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


于是我将队列组的代码全部干掉。
```
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{

    __block UIImage *image1 ,*image2;
    
    // 下载第一张图片
    image1 = [self downloadImageWithUrlString:@"http://g.hiphotos.baidu.com/image/pic/item/95eef01f3a292df54e0e7e08be315c6035a873da.jpg"];
    
    
    // 下载第二张图片
    
    image2 = [self downloadImageWithUrlString:@"http://e.hiphotos.baidu.com/image/pic/item/cc11728b4710b912d4bb69ffc1fdfc03924522bc.jpg"];
    
    
    // 合并图片并且显示
    
    // 合并图片
    UIImage *image = [self bingImageWithImage1:image1 Image2:image2];
    
    // 显示合并之后的图片!
    self.imageView.image = image;
}
```
![WechatIMG98.jpeg](http://upload-images.jianshu.io/upload_images/1743443-f5d66a54833d23f1.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

效果一摸一样，两张图片拼接出来了一张图片，所以说上面这种队列一点用都没有。

#实际开发
实际开发中遇到了上传多张图片（上传操作全部是异步操作）之后进行一个回调操作的时候，假如你按照上面的写法就会大骂一句坑爹，然后会使用很low的方式，比如说在回调里面i++，然后在最终方法里面统计这个i++是否合图片数组的count一致，然后执行最终方法。

#两种多线程的同步解法（[附上demo](https://github.com/knightSaber/CBTQueueGroupDemo)）``` enter，leave``` or ```信号量```

1.  ``` enter，leave```

```
    // 创建一个队列组!
    dispatch_group_t group = dispatch_group_create();
    
    __block UIImage *image1 ,*image2;
    
    // 下载第一张图片
    dispatch_group_enter(group);//在需要进入队列组的子线程外面进入队列组
    [self downloadImageWithUrlString:@"http://g.hiphotos.baidu.com/image/pic/item/95eef01f3a292df54e0e7e08be315c6035a873da.jpg" SuccessBlock:^(UIImage *image) {
        
        image1 = image;
        dispatch_group_leave(group);//在成功回调的时候离开队列组
        
    } failBlock:^(id error) {
        
    }];
    
    
    // 下载第二张图片
    dispatch_group_enter(group);
    [self downloadImageWithUrlString:@"http://e.hiphotos.baidu.com/image/pic/item/cc11728b4710b912d4bb69ffc1fdfc03924522bc.jpg" SuccessBlock:^(UIImage *image) {
        
        image2 = image;
        dispatch_group_leave(group);
        
    } failBlock:^(id error) {
        
    }];
    
    // 合并图片并且显示
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        
        // NSLog(@"显示图片! %@",[NSThread currentThread]);
        
        // 合并图片
        UIImage *image = [self bingImageWithImage1:image1 Image2:image2];
        
        // 显示合并之后的图片!
        self.imageView.image = image;
        
    });
```

2.  信号量

```
    // 创建一个队列组!
    dispatch_group_t group = dispatch_group_create();
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    
    __block UIImage *image1 ,*image2;
    
    /*
     需要将dispatch_semaphore_wait放在后面的原因是，程序先执行了下载图片代码,进行wait--，然后下载完成的回调signal++，这时候程序可以继续
     */
    
    dispatch_group_async(group, dispatch_queue_create("1111", DISPATCH_QUEUE_CONCURRENT), ^{
       
        // 下载第一张图片
        [self downloadImageWithUrlString:@"http://g.hiphotos.baidu.com/image/pic/item/95eef01f3a292df54e0e7e08be315c6035a873da.jpg" SuccessBlock:^(UIImage *image) {
            
            image1 = image;
            
            dispatch_semaphore_signal(semaphore);//信号量++，继续
            
        } failBlock:^(id error) {
            
        }];
        
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);//信号量--，阻塞
        
    });
    
    
    dispatch_group_async(group, dispatch_queue_create("22222", DISPATCH_QUEUE_CONCURRENT), ^{
        
        // 下载第二张图片
        [self downloadImageWithUrlString:@"http://e.hiphotos.baidu.com/image/pic/item/cc11728b4710b912d4bb69ffc1fdfc03924522bc.jpg" SuccessBlock:^(UIImage *image) {
            
            image2 = image;
            
            dispatch_semaphore_signal(semaphore);//信号量++，继续
            
        } failBlock:^(id error) {
            
        }];
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);//信号量--，阻塞
        
    });
    
    
    // 合并图片并且显示
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        
        // 合并图片
        UIImage *image = [self bingImageWithImage1:image1 Image2:image2];
        
        NSLog(@"%@",[NSThread currentThread]);
        
        // 显示合并之后的图片!
        self.imageView.image = image;
    });
```

#信号量
信号量是解决多线程同步的的API,我最先接触的时候，是从锁的角度去看待信号量的。下面是各种锁的博客，第二个就是信号量的讲解，看下面这个就好了。这里就不班门弄斧了。
[信号量的详解](http://www.cocoachina.com/ios/20160707/16957.html)

#信号量的其他玩法

队列组讲到这里，信号量的加锁将在我的另外一个简书文章里面进行讲解
[信号量的加锁](http://www.jianshu.com/p/ea1985006e1a)

#一点感悟

现在很多程序员圈子里面真的是有趣的灵魂太少，大家似乎看博客就是看博客，而不会自己去写demo去验证一下博主的对错。
“知行合一”，是几百年前王守仁提出来的思想，我觉得用在开发中不错。希望大家以后能够带着挑剔的眼光以及怀疑的心态去学知识。

[附上demo](https://github.com/knightSaber/CBTQueueGroupDemo)

@godl

我发现我的写法有点问题，现在已经改了写法，这样就不会阻塞主线程了。