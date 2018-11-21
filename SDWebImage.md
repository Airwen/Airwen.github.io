# SDWebImage

这是一片SDWebImage源码阅读笔记，我前前后后读了3～5遍SDWebImage，但都不得要领，我先一步一步地记录下我的阅读理解，最后在写总结：

从通常使用的图片缓存API开始一步一步深入到框架的核心：

## UIView (WebCacheOperation)

为UIView挂了(关联了Associated)一个“名字”为loadOperationKey的NSMapTable对象：

```
/*
 * Map<Key(Strong, appointed operationKey?:ClassName), Value(Weak, id<SDWebImageOperation>)>
 * value 实际是SDWebImageCombinedOperation，可以通过URL找到相应的operation，然后cancel
 * 这也是每次对同一个试图对象反复设置download operation时，先回cancel 掉前一个operation
 * key is copy, value is weak because operation instance is retained by SDWebImageManager's runningOperations property
 * WebCacheOperation Category是用来管理View 对象上 image operation
 * operation的设置`setObject:froKey`和移除都`removeObjectForKey:`是用了`@synchronized(self){ /* handler operation */}`来整线程安全
 * 如果开始新的image operation，sd api想查询这个view对象有没有key相应的operation，如果有cancel并remove它
 * 具体看看器setter和getter方法逻辑
 */
(retain)NSMapTable<NSString *, id<SDWebImageOperation>> *loadOperationKey;
```

> 横向要理解`NSMapTable`的特性。


## UIView (WebCache)

为UIView挂了(关联了Associated)一些属性：

```
(retain)NSProgress *sd_imageProgress
(retain)NSString *imageURLKey

(retain)NSNumber(BOOL) *TAG_ACTIVITY_SHOW
(retain)NSNumber(UIActivityIndicatorViewStyle) *TAG_ACTIVITY_STYLE
(retain)UIActivityIndicatorView *TAG_ACTIVITY_INDICATOR
```

### 下载Image的核心方法时 -[SDWebImageManager loadImageWithURL:options:progress:completed:] 代码流程

先是从通过url生成相应的cache key(`NSString *key = [self cacheKeyForURL:url];`)，然后从缓存中查询：

`- [SDImageCache queryCacheOperationForKey:options:done:]`，代码大噶看流程:

```
// First check the in-memory cache...
if shouldQueryMemoryOnly
   在 SDMemoryCache 以key查询缓存图片
   回调 doneBlock (SDCacheQueryCompletedBlock)
   return nil
/*
 * 新建一个NSOperation对象 —— 一个空任务
 * 用于在queryDiskBlock执行时根据捕获的operation的isCancelled来判断是否继续执行磁盘缓存
 * 前提是queryDiskBlock没有被执行
 */
new operation 
if Sync
   在当前线程执行queryDiskBlock
else 
   在串行的ioQueue中异步(dispatch_async)执行queryDiskBlock任务
return operation
/* return
 * 这个任务返回赋值给 SDWebImageCombinedOperation 对象的cacheOperation中，
 * 这样在cancel SDWebImageCombinedOperation 时也会cancel相应的磁盘缓存
 * 前提是queryDiskBlock没有被执行
 */
```
在查询cache Image的回调`done`Block中：


downloader 调用序列:

![sd downloader image](https://qcloud.coding.net/api/project/3905697/files/4458481/imagePreview)


引用关系图

![refrences](https://dn-coding-net-production-file.codehub.cn/6f838fe0-ed5b-11e8-ad0e-e38b74016731.png?e=1542789476&token=goE9CtaiT5YaIP6ZQ1nAafd_C1Z_H2gVP8AwuC-5:e12d0f1f90lNsthJI5b3sqOfeec=)

> SDWebImageCombinedOperation : NSObject <SDWebImageOperation> //内部
> 每个视图类都有管理自己的任务管理队列在`loadOperationKey`
> SDWebImageDownloaderOperation 是NSOperation的子类，是image下载任务类，
> SDWebImageDownloader 管理下载的operationQueue

创建下载Operation，并肩Operation添加到`downloadQueue`中的调用栈:

![refrences](https://qcloud.coding.net/api/project/3905697/files/4458481/imagePreview)

`SDWebImageManager`的对象结构

```
SDWebImageManager
    |_ SDImageCache *imageCache
    |_ SDWebImageDownloader *imageDownloader
    |_ NSMutableSet<NSURL *> *failedURLs
    |_ dispatch_semaphore_t failedURLsLock 
    |    // init with `dispatch_semaphore_create(1)`
    |_ NSMutableSet<SDWebImageCombinedOperation *> *runningOperations
    |_ dispatch_semaphore_t runningOperationsLock 
          // init with `dispatch_semaphore_create(1)`
``` 
   
`SDImageCache`的对象结构

```
SDImageCache 
   |_ SDMemoryCache *memCache;
   |_ dispatch_queue_t ioQueue; 
   |      //串行队列：`dispatch_queue_create("com.hackemist.SDWebImageCache",
   |                                                 DISPATCH_QUEUE_SERIAL);`
   |_ NSString *diskCachePath;  
        //default cacheing path `NSCachesDirectory/default/MD5(cacheKey)` 
        `-[SDImageCache cachedFileNameForKey:]`
```

`SDWebImageDownloader`的对象结构

```
SDWebImageDownloader<NSURLSessionTaskDelegate, NSURLSessionDataDelegate>
	|_ NSTimeInterval downloadTimeout // 15 seconds //public
	|_ NSOperationQueue(SDWebImageDownloaderOperation) *downloadQueue 
	|  // maxConcurrentOperationCount(6), name(com.hackemist.SDWebImageDownloader)
	|_ (weak) NSOperation *lastAddedOperation;
	|_ NSMutableDictionary<NSURL *, SDWebImageDownloaderOperation *> *URLOperations
	|_ NSMutableDictionary<NSString *, NSString *> *HTTPHeaders 
	| //@{@"Accept": @"image/webp,image/*;q=0.8"} or @{@"Accept": @"image/*;q=0.8"}
	|_ NSURLSession *session 
	   //Configuration(defaultSessionConfiguration) 
	   //delegate(SDWebImageDownloader) delegateQueue(nil) 
```

`SDWebImageDownloadToken`的对象结构

```
SDWebImageDownloadToken<SDWebImageOperation>
    |_ NSURL *url;
    |_ id downloadOperationCancelToken; 
    |  //assgined by `-[SDWebImageDownloaderOperation 
    |                       addHandlersForProgress:completed:]`
    |_ (weak)NSOperation<SDWebImageDownloaderOperationInterface> *downloadOperation;
```

`SDWebImageDownloaderOperation`的对象结构

```
SDWebImageDownloaderOperation : NSOperation <SDWebImageDownloaderOperationInterface, SDWebImageOperation, NSURLSessionTaskDelegate, NSURLSessionDataDelegate>
    |_ (public,strong)NSURLRequest *request;
    |_ (readonly,strong)NSURLSessionTask *dataTask;
    |_ (private,strong)NSMutableData *imageData;
    |_ (private,copy)NSData *cachedData; 
    |          // for `SDWebImageDownloaderIgnoreCachedResponse`
    |_ (private,strong)dispatch_queue_t coderQueue; 
    |                   // the queue to do image decoding, serial Queue
    |_ (public,strog)NSURLCredential *credential;
```

`SDMemoryCache`对象结构

```
// A memry cache which auto purge the cache on memory warning and support weak cache.
## SDMemoryCache <KeyType, ObjectType> : NSCache <KeyType, ObjectType>
     |_ NSMapTable<KeyType, ObjectType> *weakCache; // Key(strong): Value(weak) cache
```

一些对我来说有意思的代码片段：

```
-[SDWebImagePrefetcher prefetchURLs]// Invoking this method without a completedBlock is pointless(prefetch: transfer (data) from main memory to temporary storage in readiness for later use: this model prefetches instructions before they need to be executed.)
```

```
  SDWebImageDownloaderOperation.credential = [NSURLCredential credentialWithUser:sself.username password:sself.password persistence:NSURLCredentialPersistenceForSession];
```

```
// Prevents app crashing on argument type error like sending NSNull instead of NSURL
    if (![url isKindOfClass:NSURL.class]) {
        url = nil;
    }
```

```
dispatch_semaphore_t lock = dispatch_semaphore_create(1)
#define LOCK(lock) dispatch_semaphore_wait(lock, DISPATCH_TIME_FOREVER);
#define UNLOCK(lock) dispatch_semaphore_signal(lock);
```

```
if (sself.executionOrder == SDWebImageDownloaderLIFOExecutionOrder) {
            // Emulate LIFO execution order by systematically adding new operations as last operation's dependency
            [sself.lastAddedOperation addDependency:operation];
            sself.lastAddedOperation = operation;
        }
```

```
//在设置新的Session之前会
-[invalidateAndCancel invalidateAndCancel]
```

生成 NSMutableURLRequest :

```
// In order to prevent from potential duplicate caching (NSURLCache + SDImageCache) we disable the cache for image requests if told otherwise
        NSURLRequestCachePolicy cachePolicy = options & SDWebImageDownloaderUseNSURLCache ? NSURLRequestUseProtocolCachePolicy : NSURLRequestReloadIgnoringLocalCacheData;
        NSMutableURLRequest *request = [[NSMutableURLRequest alloc] initWithURL:url
                                                                    cachePolicy:cachePolicy
                                                                timeoutInterval:timeoutInterval];
        
        request.HTTPShouldHandleCookies = (options & SDWebImageDownloaderHandleCookies);
        request.HTTPShouldUsePipelining = YES;
        if (sself.headersFilter) {
            request.allHTTPHeaderFields = sself.headersFilter(url, [sself allHTTPHeaderFields]);
        }
        else {
            request.allHTTPHeaderFields = [sself allHTTPHeaderFields];
        }
```

// ?
downloadedImage = [SDWebImageManager scaledImageForKey:key image:downloadedImage];

