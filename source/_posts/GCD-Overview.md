---
title: GCD Overview
date: 2016-07-18 12:56:55
tags:
- GCD
- Objective-C
- iOS
---
总结下 GCD.
#### Basics
CGD 是对线程的抽象。将线程管理交给系统实现，程序员只需要将想执行的任务加入到相应的队列中。

只有global queue 会被调度运行。当创建自己的queue 的时候，实际上会被set target 到global queue 去，custome queue 将要执行的 block 交给 global queue 执行。(存疑，只在一本书上看到这样的描述)

Concurrent Disptach Queue 会由系统来决定应当生成的线程数。而 Serial Dispatch Queue 每创建一条都会创建一条新线程。（大量生成 Serial Queue 会消耗大量内存）

iOS 6.0 之后 GCD 已经加入 ARC, 无需手动释放。

#### Set target
```objc
dispatch_set_target(queueA, queueB);
```
queueA 将 block 传给 queueB 执行

#### Suspend & Resume
```objc
dispatch_suspend(queue);
dispatch_resume(queue);
```
queue 被 suspend 之后将停止执行队列中的block. (当前block会继续执行到结束)

#### dispatch_barrier_async
```objc
dispatch_async(queue, ^{
	// read blocks one
});

dispatch_barrier_async(queue, ^{
	// write block
});

dispatch_async(queue, ^{
	// read blocks two
});
```
dispatch_barrier_async 加入block 之前的 blocks 执行完毕后， 执行 barrier 加入的 block, 完成之后再执行之后加入的 block

case: read block 用普通的 dispatch_async, write block 用 dispatch_barrier_async。比如一个array，可以用这样的方法来保证其线程安全。

#### dispatch group
将几个block group 起来，全部执行完后通知用户
```objc
dispatch_group_t group = dispatch_group_create();

dispatch_group_async(group, queue, blockA);
dispatch_group_async(group, queue, blockB);

// blockC will be executed after blockA and block B finishes
dispatch_group_notify(group, queue, blockC);
```
dispatch_group_wait 设置等待时间，到时间直接返回，若为0 则全部完成。设置 DISPATCH_TIME_FOREVER 永远等待，DISPTACH_TIME_NOW 立即返回。
```objc
long result = dispatch_group_wait(group, time);
if (result == 0) {
	// all blocks finished executing
} else {
	// some blocks are not finished yet
}
```
#### dispatch_apply
dispatch_sync 的复数版本
```objc
// execute the block on queue 10 times, 
// will block current thread until all blocks are finished
dispatch_apply(10, queue, ^(size_t index){
	// index of the executing times
});
```

#### dispatch semaphore
计数信号
```objc
// create a semphore with initial count of 10
dispatch_semaphore_t semaphore = dispatch_semaphore_create(10);

// wait for the chance to reduce 1 to the semaphore
long result = dispatch_semaphore_wait(semaphore, time);

// perform some tasks...

// increacse one to semaphore
dispatch_semaphore_signal(semaphore);
```

#### dispatch_once
略

#### dispatch source
##### dispatch source data add

```objc
dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_DATA_ADD,
							0, 0, queue);

// add handler to the source 			
__block long totalComplete = 0;
dispatch_source_set_event_handler(source, ^{
	long value = dispatch_source_get_data(source);
	totalComplete += value;
})

// created cource is in suspended status
// needs to be resumed
dispatch_resume(source);

// sending data to the source
dispatch_async(queue, ^{
	for (int i = 0 ; i <= 100; i++) {
		dispatch_source_merge_data(source, 1);
		usleep(200000);
	}
})
```
注意不可发送0 或者 复数

##### dispatch timer
```objc
void dispatch_source_set_timer( dispatch_source_t source, dispatch_time_t start, uint64_t interval, uint64_t leeway);
```
start - start after the start interval
interval - execution interval. the handler will be called with after the first execution repeately with the interval. set to DISPATCH_TIME_FOREVER if you want to perform it only once. (use dispatch_after if you want to perform some task after a delay only once)
leeway - the time allowed for delay. used a hint to the system
```objc
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0);
    dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
    dispatch_source_set_timer(timer, dispatch_time(DISPATCH_TIME_NOW, 5 * NSEC_PER_SEC), 1 * NSEC_PER_SEC, 1 * NSEC_PER_SEC);
    dispatch_source_set_event_handler(timer, ^{
        // perform some tasks
    });
    
    
    // to cancel the timer after 10 seconds
    dispatch_queue_t q = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0);
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(10 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        dispatch_cancel(timer);
    });
```
##### Queue sepcific data
Similiar to the Associated Object. Assign a key-value pair to a queue. A release function is required.
```objc
static char key;
CFStringRef *value = CFSTR("some string");
dispatch_queue_set_specific(queue,  
                              &key,  
                              (void*)value,  
                              (dispatch_function_t)CFRelease); 
                              
   CFStringRef *str = dispatch_get_specific(&key);
```
如果找不到key, 会向target queue 查询。

例子
```objc
dispatch_queue_set_specific(q, &key, (__bridge void*)q, NULL);
```
因为dispatch_get_current_queue 可能会造成死锁，如下，get current queue 检查为当前queue 为queue B, 所以调用 dispatch_sync(queueA, …), 造成死锁（因为最外面还套一个 dispatch_sync(queueA, …)）
```objc
// a 'not' safe sync
void dispatch_sync_safe(dispatch_queue_t queue, dispatch_block_t block) {
	if (dispatch_get_current_queue() == queue) {
		block();
	} else {
		dispatch_sync(queue, block);
	}
}

dispatch_sync(queueA, ^{
	dispatch_sync(queueB, ^{
		 dispatch_sync_safe(queueA, ^{
		 	// some tasks...
		 })
	});
});
```
上面的dispatch_sync_safe 重入，导致deadlock

解决方法：
```objc
static const char sQueueTagKey;

void RNQueueTag(dispatch_queue_t q) {
  // Make q point to itself by assignment.
  // This doesn't retain, but it can never dangle.
  dispatch_queue_set_specific(q, &sQueueTagKey, (__bridge void*)q, NULL);
}

dispatch_queue_t RNQueueCreateTagged(const char *label,
                                     dispatch_queue_attr_t attr) {
  dispatch_queue_t q = dispatch_queue_create(label, attr);
  RNQueueTag(q);
  return q;
}

dispatch_queue_t RNQueueGetCurrentTagged() {
  return (__bridge dispatch_queue_t)dispatch_get_specific(&sQueueTagKey);
}

BOOL RNQueueCurrentIsTaggedQueue(dispatch_queue_t q) {
  return (RNQueueGetCurrentTagged() == q);
}

BOOL RNQueueCurrentIsMainQueue() {
  return [NSThread isMainThread];
}

void RNQueueSafeDispatchSync(dispatch_queue_t q, dispatch_block_t block) {
  if (RNQueueCurrentIsTaggedQueue(q)) {
    block();
  }
  else {
    dispatch_sync(q, block);
  }
}
```
上面利用Queue sepcific data 会像target queue询问的特性来避免死锁。
不过只能应对这种只有两条queue 的情况，而且只能给最外面的queue 加上标签。（所以有啥用啊。。。）

#### References
1. iOS 7 Programming: Pushing the limit
2. Objective-C 高级编程
