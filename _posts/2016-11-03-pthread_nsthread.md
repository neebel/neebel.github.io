---
layout: post
title: iOS多线程之pthread和NSThread
categories: iOS pthread NSThread 
date: 2016-11-03 18:08:15
pid: 20161103-180815
---

iOS开发中，多线程相关的知识点主要包括**pthread**、**NSThread**、**NSOperation**和**GCD**，我们经常用到的就数**NSOperation**和**GCD**了。学习了一段时间后，觉得有必要总结巩固一下，对自己也是一种提高。**pthread**和**NSThread**内容不多，所以放在同一篇，**NSOperation**和**GCD**各一篇，总共三篇。

本篇文章主要内容：

- **简单介绍pthread**
- **NSThread的使用**

-------------------

## pthread

> POSIX线程（POSIX threads），简称Pthreads，是线程的POSIX标准。该标准定义了创建和操纵线程的一整套API。在类Unix操作系统（Unix、Linux、Mac OS X等）中，都使用Pthreads作为操作系统的线程。


简单来说就是操作系统级别使用的线程，基于c语言实现，我们的OC代码中很少用到，并且不便于管理。在pthread.h中，我们可以看到很多操作线程的方法：

>pthread_create( ) : 创建一个线程
  pthread_exit ( )   :  退出当前线程
  pthread_main_np ( ) : 获取主线程
  ......

这些方法具体怎么用，本篇文章不再关注，有兴趣的童鞋可自行研究，这里看下互斥锁的相关内容。**互斥锁**的作用是防止多个线程同时访问临界区引起的脏数据或者数据损坏问题。pthread_mutex的用法如下：

> pthread_mutex_t   _lock;
   pthread_mutex_init(&_lock, NULL);  //初始化一个互斥锁
pthread_mutex_lock(&_lock); //加锁，线程进入临界区，其他线程在外面等待
......   //执行临界区代码
pthread_mutex_unlock(&_lock); //解锁，线程离开临界区，其他线程进入临界区执行
pthread_mutex_destroy(&_lock); //最后销毁互斥锁


**互斥锁保证了临界区代码在某个时刻只有一个线程在执行。**iOS开发中还有其他类型的锁，以后会再写一篇文章单独介绍。

## NSThread的使用

**创建**


> 类方法
   detachNewThreadSelector:(SEL)selector toTarget:(id)target withObject:(nullable id)argument;
  detachNewThreadWithBlock:(void (^)(void))block; //iOS10新增方法，以块的形式执行
  
  >实例方法
  initWithTarget:(id)target selector:(SEL)selector object:(nullable id)argument;
 initWithBlock:(void (^)(void))block; //iOS10新增方法，以块的形式执行

**执行**

>对于类方法创建的线程会自动执行，而实例方法创建的线程需要调用start才能执行

**配置线程**

>设置线程名称：[[NSThread currentThread] setName:@"xxxx"]
设置线程优先级：(BOOL)setThreadPriority:(double)p
......

**操作线程**
>cancel //取消线程
  exit  //停止线程
  sleepUntilDate:(NSDate *)date //线程休眠到某个时间点
  sleepForTimeInterval:(NSTimeInterval)ti //线程休眠一段时间

**获取线程信息**
> (double)threadPriority;//获取线程优先级
   BOOL isMainThread; //当前线程是否是主线程，开发中比较有用
   BOOL executing;//是否正在执行
   BOOL finished; //是否已经结束
   BOOL cancelled;//是否被取消了

**线程间通信**

>//在主线程执行方法
performSelectorOnMainThread:(SEL)aSelector withObject:(nullable id)arg waitUntilDone:(BOOL)wait;

>//在某个线程执行方法
 performSelector:(SEL)aSelector onThread:(NSThread *)thr withObject:(nullable id)arg waitUntilDone:(BOOL)wait
 
 >//在后台线程执行方法
 performSelectorInBackground:(SEL)aSelector withObject:(nullable id)arg

部分实例代码如下，[完整链接点这里](https://github.com/neebel/NBLNSThreadDemo)：


```
@implementation ViewController

#pragma mark - LifeCycle

- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self initUI];
    
    //以block形式执行,类方法
    [NSThread detachNewThreadWithBlock:^{
        [[NSThread currentThread] setName:@"block线程"];
        [NSThread sleepForTimeInterval:0.5];
        NSString *info = [NSString stringWithFormat:@"detach新线程执行Block,thread info:%@", [NSThread currentThread]];
        [self performSelectorOnMainThread:@selector(fillLabel:) withObject:info waitUntilDone:NO];
    }];
    
    //以方法形式执行，类方法
    [NSThread detachNewThreadSelector:@selector(detachThreadExcuteMethod) toTarget:self withObject:nil];
}

#pragma mark - Getter

- (UIButton *)startButton
{
    if (!_startButton) {
        UIButton *startButton = [[UIButton alloc] initWithFrame:CGRectMake(100, 280, 150, 30)];
        [startButton setTitle:@"点击开始下载图片" forState:UIControlStateNormal];
        [startButton setTitleColor:[UIColor blueColor] forState:UIControlStateNormal];
        [startButton addTarget:self action:@selector(startDownload) forControlEvents:UIControlEventTouchUpInside];
        _startButton = startButton;
    }

    return _startButton;
}

- (NSThread *)downloadThread
{
    if (!_downloadThread) {
        //以方法形式执行，实例方法,需要手动开始
        NSThread *downloadThread = [[NSThread alloc] initWithTarget:self selector:@selector(downloadPicture) object:nil];
        [downloadThread setName:@"download thread"];
        _downloadThread = downloadThread;
    }
    
    return _downloadThread;
}

#pragma mark - Private

- (void)initUI
{
    [self.view addSubview:self.threadInfoTextView];
    [self.view addSubview:self.startButton];
    [self.view addSubview:self.imageView];
}

#pragma mark - Action

- (void)detachThreadExcuteMethod
{
    [[NSThread currentThread] setName:@"method线程"];
    [NSThread sleepForTimeInterval:0.5];
    NSString *info = [NSString stringWithFormat:@"detach新线程执行方法，thread info:%@",[NSThread currentThread]];
    NSMutableString *str = [NSMutableString stringWithString:info];
    for (NSInteger i = 0; i < 5; i++) {
        [str appendString:[NSString stringWithFormat:@"\n第%@次循环", [NSNumber numberWithInteger:i].stringValue]];
    }
    [self performSelectorOnMainThread:@selector(fillLabel:) withObject:str waitUntilDone:NO];
}


- (void)downloadPicture
{
    NSError *error;
    NSData *imageData = [[NSData alloc] initWithContentsOfURL:
                         [NSURL URLWithString:@"https://www.baidu.com/img/bd_logo1.png"]
                                                      options:0 error:&error];
    if(imageData == nil) {
        NSLog(@"Error: %@", error);
    } else {
        UIImage *image = [[UIImage alloc] initWithData:imageData];
        [self performSelectorOnMainThread:@selector(fillPicture:) withObject:image waitUntilDone:NO];
    }
}


- (void)startDownload
{
    //线程执行完成后会死掉，如果再次调用其start方法会crash
    //线程正在执行中，如果再次调用其start方法也会crash
    if ([self.downloadThread isFinished] || [self.downloadThread isExecuting]) {
        return;
    }
    
    [self.downloadThread start];
}


- (void)fillPicture:(UIImage *)image
{
    self.imageView.image = image;
}


- (void)fillLabel:(NSString *)info
{
    NSMutableString *str = [NSMutableString stringWithString:self.threadInfoTextView.text];
    [str appendString:@"\n\n"];
    [str appendString:info];
    self.threadInfoTextView.text = str;
}

@end
```

另外一点，pthread和NSThread是一一对应的关系，例如两者都提供了获取主线程和当前线程的方法。

至此，pthread和NSThread的介绍就差不多完了，两者在开发中用途不是很多，所以也没有做特别深入的研究，例如线程顺序执行、线程同步等问题。这是本人写的第一篇博客，肯定有不正确或者不恰当的地方，希望通过以后更多的写作实践得以改善。
