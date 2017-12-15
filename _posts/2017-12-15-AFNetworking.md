---
layout: post
title: AFNetworking网络请求框架的运行流程
categories: iOS
date: 2017-12-15 10:59:00
pid: 20171215-105900
---

最近仔细阅读了AFNetworking的源码，现总结下AFNetworking网络请求框架的运行流程，以备查用。

主要流程包括：

1. NSURLSession的创建

2. NSURLRequest的创建、配置和序列化

3. NSURLSessionTask的创建

4. NSURLSessionTask启动后相关代理回调

5. AFURLSessionManagerTaskDelegate代理

6. 响应的解析（反序列化）


## NSURLSession创建

（1）使用NSURLSessionConfiguration创建NSURLSession

（2）设置NSURLSession的代理和代理回调所在的队列

（3）配置序列化和反序列化对象

（4）配置安全策略

（5）创建全局字典（该字典维护NSURLSessionTask和其对应的AFURLSessionManagerTaskDelegate的一对一关系）

（6）清除NSURLSession的所有task的代理


## NSURLRequest的创建、配置和序列化

（1）生成NSURLRequest

（2）设置NSURLRequest的相关属性，例如cachePolicy、HTTPShouldHandleCookies、HTTPShouldUsePipelining等（通过RequestSerializer设置）

（3）设置NSURLRequest的HTTP Header属性（也是通过RequestSerializer设置）

（4）通过传入的参数构建query字符串

（5）判断请求方法，如果是GET、HEAD、DELETE，将query字符串拼接到url后面，如果是其他，则放在HTTP Body中

## NSURLSessionTask的创建

 (1) iOS8以上直接创建，iOS8以下在AF的串行队列里面同步创建，保证taskid的唯一性（taskid作为key映射到delegate的，所以必须唯一）

（2）为SessionTask设置AFURLSessionManagerTaskDelegate代理，传入的成功失败进度等回调参数也交给AFURLSessionManagerTaskDelegate管理

（3）AFURLSessionManagerTaskDelegate管理、监听SessionTask的进度，其内部维护的NSProgress与SessionTask的状态关联起来，实时获取请求的进度，通过传入的回调参数交给调用者

（4）NSURLSessionTask启动 

## NSURLSessionTask启动后相关代理回调

（1）- (void)URLSession:(NSURLSession *)session didBecomeInvalidWithError:(NSError *)error
     
     NSURLSession失效时回调调用者的block，并发送Session失效通知

（2）- (void)URLSession:(NSURLSession *)session 
    didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
    completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential *credential))completionHandler   
   
    用于HTTPS认证的代理，让客户端提供认证策略和证书，然后再去做系统认证
 
（3）- (void)URLSession:(NSURLSession *)session
    task:(NSURLSessionTask *)task
    willPerformHTTPRedirection:(NSHTTPURLResponse *)response
    newRequest:(NSURLRequest *)request
    completionHandler:(void (^)(NSURLRequest *))completionHandler

    当某个请求被服务端重定向时回调该方法，用于重新生成一个重定向请求  

（4）- (void)URLSession:(NSURLSession *)session
    task:(NSURLSessionTask *)task
    didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
    completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential *credential))completionHandler

    针对某个task的HTTPS认证

（5）- (void)URLSession:(NSURLSession *)session
    task:(NSURLSessionTask *)task
    needNewBodyStream:(void (^)(NSInputStream *bodyStream))completionHandler   

    session task 需要发送一个新的bodyStream到服务器时调用

（6）- (void)URLSession:(NSURLSession *)session
    task:(NSURLSessionTask *)task
    didSendBodyData:(int64_t)bytesSent
    totalBytesSent:(int64_t)totalBytesSent
    totalBytesExpectedToSend:(int64_t)totalBytesExpectedToSend   

    每次发送数据后回调该代理，告知调用者实时的进度

（7）- (void)URLSession:(NSURLSession *)session
    task:(NSURLSessionTask *)task
    didCompleteWithError:(NSError *)error  
    
    某个请求完成后调用，转发给AFURLSessionManagerTaskDelegate处理，AFURLSessionManagerTaskDelegate负责移除和代理相关的监听、通知

（8）- (void)URLSession:(NSURLSession *)session
    dataTask:(NSURLSessionDataTask *)dataTask
    didReceiveResponse:(NSURLResponse *)response
    completionHandler:(void (^)(NSURLSessionResponseDisposition disposition))completionHandler

    收到服务器的响应时调用，并决定后续的行为：

    NSURLSessionResponseCancel = 0,                                    
    NSURLSessionResponseAllow = 1,                                     
    NSURLSessionResponseBecomeDownload = 2,                           
    NSURLSessionResponseBecomeStream

 (9) - (void)URLSession:(NSURLSession *)session
     dataTask:(NSURLSessionDataTask *)dataTask
     didBecomeDownloadTask:(NSURLSessionDownloadTask *)downloadTask

     当一个请求转变成下载请求时调用，重新生成一个download task， delegate与老的task接除关联，与新的task建立关联

（10）- (void)URLSession:(NSURLSession *)session
      dataTask:(NSURLSessionDataTask *)dataTask
      didReceiveData:(NSData *)data    

      收到数据后转发给AFURLSessionManagerTaskDelegate处理

（11） - (void)URLSession:(NSURLSession *)session
      dataTask:(NSURLSessionDataTask *)dataTask
      willCacheResponse:(NSCachedURLResponse *)proposedResponse
      completionHandler:(void (^)(NSCachedURLResponse *cachedResponse))completionHandler

      请求完成后给用户做缓存使用的

（12） - (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
      didFinishDownloadingToURL:(NSURL *)location

      任务下载完成后调用，从用户自定义的block中拿到文件保存的目的地址，然后将文件移动到目的地址，这里用户设置的目的地址是针对全局任务的        

 (13)  - (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
      didWriteData:(int64_t)bytesWritten
      totalBytesWritten:(int64_t)totalBytesWritten
      totalBytesExpectedToWrite:(int64_t)totalBytesExpectedToWrite

      实时通知下载进度

 (14) - (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
      didResumeAtOffset:(int64_t)fileOffset
      expectedTotalBytes:(int64_t)expectedTotalBytes 

      下载任务重新开始时调用，可以用来做断点续传

## AFURLSessionManagerTaskDelegate代理

 (1) - (void)URLSession:(__unused NSURLSession *)session
     task:(NSURLSessionTask *)task
     didCompleteWithError:(NSError *)error
    
    请求完成，处理数据，回调给调用者，发送通知

 (2) - (void)URLSession:(__unused NSURLSession *)session
     dataTask:(__unused NSURLSessionDataTask *)dataTask
     didReceiveData:(NSData *)data  
 
    拼接数据

 (3) - (void)URLSession:(NSURLSession *)session
     downloadTask:(NSURLSessionDownloadTask *)downloadTask
     didFinishDownloadingToURL:(NSURL *)location    
   
    设置某个下载任务的保存地址

## 响应的解析（反序列化）

   1.判断响应状态码、acceptableContentTypes是否正确匹配

   2.将data解析成对应的数据类型，像Json、XML等

   3.回调结果

