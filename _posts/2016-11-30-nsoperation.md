---
layout: post
title: iOS多线程之NSOperation
categories: iOS NSOperation 
date: 2016-11-30 00:49:58
pid: 20161130-004958
---

**NSOperation**是Apple基于GCD的封装，面向对象，抽象层次更高，使用简单。我们只需根据任务类型创建合适的NSOperation，配置好相应的参数，丢到**NSOperationQueue**中，剩下的工作就交给系统调度处理，我们无需关心。是不是很诱人？

本篇文章主要内容：

- **NSOperation和NSOperationQueue简介**
- **创建操作对象的三种方式 **
- **设置操作的优先级**
- **设置操作的依赖**
- **获取操作的状态**
- **取消操作**
- **暂停和恢复操作队列**
- **操作完成的回调**

## NSOperation和NSOperationQueue简介

NSOperation和NSOperationQueue理解起来很容易，我们可以参照GCD相关内容：NSOperation相当于GCD中的任务块，而NSOperationQueue相当于GCD中的并发队列。使用NSOperation编程用到的类主要有**NSOperation**、**NSBlockOperation**、**NSInvocationOperation**和**NSOperationQueue**。NSBlockOperation和NSInvocationOperation是NSOperation的子类，我们不会使用NSOperation这个抽象基类，而是使用NSBlockOperation和NSInvocationOperation创建我们的任务。下面的内容会具体介绍这些类的使用。[代码地址](https://github.com/neebel/NBLNSOperationDemo)

## 创建操作对象的三种方式

### 1.NSInvocationOperation

初始化方法：
>initWithTarget:(id)target selector:(SEL)sel object:(nullable id)arg;

代码示例：

```
NSInvocationOperation *operation = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(sayHello) object:nil];
```

```
- (void)sayHello
{
    NSLog(@"Hello World");
}
```

如果我们创建了一个NSInvocationOperation对象，不放入队列中执行，而是直接调用start方法会是怎样的呢？

```
NSInvocationOperation *downloadOperation = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(downloadPic) object:nil];
    [downloadOperation start];
    NSLog(@"complete");
```

```
- (void)downloadPic
{
    NSLog(@"start download");
    [NSThread sleepForTimeInterval:2.0];
}
```

运行后发现：打印 "start download" 后，过了2秒才打印 "complete"，可以看出没有放入队列而直接调用start运行的操作，会阻塞当前线程，是同步的，只有添加到队列后，操作才会异步执行。下面的NSBlockOperation同理。

### 2.NSBlockOperation

初始化方法：
> blockOperationWithBlock:(void (^)(void))block；

添加操作：
> addExecutionBlock:(void (^)(void))block;

代码示例：

```
NSBlockOperation *blockOperation = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"start download ------ %@", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:2.0];
    }];
    
    [blockOperation addExecutionBlock:^{
        NSLog(@"hello1 ------ %@", [NSThread currentThread]);
    }];
    
    [blockOperation addExecutionBlock:^{
        NSLog(@"hello2 ------ %@", [NSThread currentThread]);
    }];
    
    [blockOperation addExecutionBlock:^{
        NSLog(@"hello3 ------ %@", [NSThread currentThread]);
    }];
    
    [blockOperation start];
    
    NSLog(@"complete------ %@", [NSThread currentThread]);

```

从运行结果可以看出，通过blockOperationWithBlock创建的操作永远在主线程执行，addExecutionBlock添加的其他操作会分发到子线程执行。

### 3.继承自NSOperation

有些情况下，前面两种方式不能满足我们的需求，就需要自定义NSOperation了。我们可以创建两种类型的NSOperation：非并发的和并发的。本人能力有限，恐介绍不全，想要彻底吃透这块需要一定时间，建议大家直接参照Apple提供的[官方文档及示例代码](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationObjects/OperationObjects.html#//apple_ref/doc/uid/TP40008091-CH101-SW16)。

##设置操作的优先级

有时候我们想提前或推迟一些操作的执行，就可以通过设置操作的优先级来达到目的。操作的优先级分为以下几个级别：

```
typedef NS_ENUM(NSInteger, NSOperationQueuePriority) {
	NSOperationQueuePriorityVeryLow = -8L,
	NSOperationQueuePriorityLow = -4L,
	NSOperationQueuePriorityNormal = 0,
	NSOperationQueuePriorityHigh = 4,
	NSOperationQueuePriorityVeryHigh = 8
};
```

我们需要注意的是：操作的优先级是相对于**同一个操作队列**中的其他操作而言的，不同操作队列中的操作的优先级没有可比性。例如：操作队列1中的操作A的优先级为NSOperationQueuePriorityVeryHigh，操作队列2中的操作B的优先级为NSOperationQueuePriorityVeryLow，至于A和B谁先执行，完全不确定。

代码示例：
```
NSBlockOperation *downloadPicOperation = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"start download picture ------ %@", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:2.0];//模拟下载操作
    }];
    [downloadPicOperation setQueuePriority:NSOperationQueuePriorityVeryLow];
    
    
    NSBlockOperation *downloadMusicOperation = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"start download music ------ %@", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:4.0];//模拟下载操作
    }];
    [downloadMusicOperation setQueuePriority:NSOperationQueuePriorityVeryHigh];
    
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];
    [queue addOperation:downloadPicOperation];
    [queue addOperation:downloadMusicOperation];
```

多次运行代码，我们发现优先级高的不一定会早于优先级低的任务执行，这就是我们需要注意的另外一点：**优先级高的操作先执行的概率大，但并不表示必然先执行**。设置优先级的代码要在操作放入操作队列之前，否则是不起作用的或者说会产生不利的影响。

##设置操作的依赖

在并发编程中，如果任务之间没有执行的先后关系，那么它们并发执行是没问题的，但还有一种情况是某个任务的执行需要其他任务的执行结果，这时候就要通过设置操作的依赖来达到目的，类似于GCD的同步。

代码示例：
```
- (void)startAction
{
    NSBlockOperation *downloadPicOperation = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"start download picture ------ %@", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:2.0];//模拟下载操作
    }];
    
    NSBlockOperation *prepareOperation = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"prepare download picture ------ %@", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:1.0];//模拟下载前的准备工作
    }];
    
    [downloadPicOperation addDependency:prepareOperation];
    
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];
    [queue addOperation:downloadPicOperation];
    [queue addOperation:prepareOperation];
}

```

如代码所示，我们对下载图片操作添加了依赖，无论运行多少次，下载操作都是等到准备操作执行完成后才执行的。当然，对操作添加依赖也可以发生在不同队列中，需要注意的是不能形成循环依赖关系，会导致死锁。设置依赖的代码要在操作放入操作队列之前，否则是不起作用的或者说会产生不利的影响。与添加依赖对应的操作是移除依赖：

```
- (void)removeDependency:(NSOperation *)op;
```

##获取操作的状态

操作的生命周期可以表示为：ready  —>  excute —> finish ，我们将操作放入操作队列后，操作的isReady状态取决于其依赖是否都已执行完成，操作的isExecuting状态表示操作正在执行，操作的isFinished状态表示操作已经执行完成或者被cancel掉了。操作同一时刻只可能是以上三种状态中的一种。

##取消操作

取消单个操作，我们可以调用cancel方法；取消操作队列中的所有操作，我们可以调用cancelAllOperations方法。我们最好是在确定不需要某个操作的时候才取消它，因为一旦取消，这个操作就被作为finished处理。

##暂停和恢复操作队列

暂停操作队列：
```
[queue setSuspended:YES];
```

恢复操作队列：
```
[queue setSuspended:NO];
```

当我们暂停某个操作队列时，操作队列就会停止调度新的操作执行，而正在执行的操作不会被停止。

##操作完成的回调
代码示例：
```
- (void)startAction
{
    NSBlockOperation *downloadPicOperation = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"start download picture ------ %@", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:2.0];//模拟下载操作
    }];
    
    [downloadPicOperation setCompletionBlock:^{
        NSLog(@"download picture complete");
    }];
    
    NSBlockOperation *downloadMusicOperation = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"start download music ------ %@", [NSThread currentThread]);
        [NSThread sleepForTimeInterval:4.0];//模拟下载操作
    }];
    
    [downloadMusicOperation setCompletionBlock:^{
         NSLog(@"download music complete");
    }];
    
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];
    [queue addOperation:downloadPicOperation];
    [queue addOperation:downloadMusicOperation];
}

```
操作完成的回调可以用于通知主线程任务完成，注意：如果在回调中更新UI，需要派发到主线程执行。

至此，iOS多线程之NSOperation相关内容就总结完了，加上之前的两篇：iOS多线程之GCD、iOS多线程之pthread和NSThread，iOS并发编程基本就覆盖全面了。学以致用，希望自己在以后的实践中不断地总结提升。