---
title: Core Data (Part 1)
date: 2015-10-19 23:11:57
tags:
- Core Data
- iOS
---

现在在读Marcus S. Zarra的[Core Data](https://pragprog.com/book/mzcd2/core-data)。自己也写了一些code来尝试core data。所以在这里总结一下。不对的地方请指正:)

#### 基础
Core Data 对几个部分：NSManagedObjectModel, NSPersistentStoreCoordinator, NSPersistentStore, NSManagedObjectContext, NSManagedObject

#### NSManagedObjectModel
NSManagedObjectModel 就是你在xcode里面创建的.xcdatamodel 编译过后得到的.mom 的binary file。 它是我们创建的data model。
加载NSManagedObjectModel就是用这个.mom file 来 init 一个NSManagedObjectModel:
```objc
NSURL *modelURL = [[NSBundle mainBundle] URLForResource:@"ModelName" withExtension:@"momd"];

ZAssert(modelURL, @"Failed to find model URL");

NSManagedObjectModel *mom = nil;
mom = [[NSManagedObjectModel alloc] initWithContentsOfURL:modelURL];

ZAssert(mom, @"Failed to initialize model");
```
顺便安利一下ZAssert，详见[MY CURRENT PREFIX.PCH FILE by Marcus Zarra ](http://www.cimgf.com/2010/05/02/my-current-prefix-pch-file/)

#### NSPersistentStoreCoordinator
NSPersistentStoreCoordinator 是Core Data的核心，负责loading, caching data 还有持久化。而NSPersistentStore 就是持久化的目的地，比如本地的SQlite Database。因为加载NSPersistentStore的时间可能很长（比如从iCould)，所以应当异步执行。
```objc
NSPersistentStoreCoordinator *psc = nil;
psc = [[NSPersistentStoreCoordinator alloc] initWithManagedObjectModel:mom];

dispatch_queue_t queue = NULL;
queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_async(queue, ^{
    NSFileManager *fm = [NSFileManager defaultManager];
    NSArray *dictArr = [fm URLsForDirectory:NSDocumentationDirectory inDomains:NSUserDomainMask];
    
    NSURL *storeURL = nil;
    NSError *error = nil;
    
    storeURL = [dictArr lastObject];
    if (![fm fileExistsAtPath:[storeURL absoluteString]]) {
        [fm createDirectoryAtURL:storeURL withIntermediateDirectories:YES attributes:nil error:&error];
        if (error) {
            ALog(@"Error creating store dictionary %@\n%@", [error localizedDescription], [error userInfo]);
        }
    }
    
    storeURL = [storeURL URLByAppendingPathComponent:@"TableForDummies.sqlite"];
    
    NSPersistentStore *store = nil;
    
    store = [psc addPersistentStoreWithType:NSSQLiteStoreType configuration:nil URL:storeURL options:nil error:&error];
    
    ZAssert(store, @"Error adding persistent store to coordinator %@\n%@", [error localizedDescription], [error userInfo]);
});
```

#### NSManagedObjectContext
上面几个在init 之后基本都不会再碰了，而NSManagedObjectContext将会被用来保存和读取数据。
NSManagedObjectContext 只需要NSPersistentStore 就可以init 了，为了确保程序其他地方可以及时调用到NSManagedContext，NSManagedContext应该在NSPersistentStore init 之后init
```objc
NSManagedObjectContext *moc = nil;
NSManagedObjectContextConcurrencyType ccType = NSMainQueueConcurrencyType;
moc = [[NSManagedObjectContext alloc] initWithConcurrencyType:ccType];
[moc setPersistentStoreCoordinator:psc];
```
同时由于NSManagedContext 不是线程安全的，这里用NSMainQueueConcurrencyType，之后对于NSManagedContext 的调用都应该在主线程里。

#### Fetching & Inserting
##### Fetch
Fetch 的时候要用NSFetchRequest，会得到一条NSManagedObject 的array
```objc
NSFetchRequest *request = [[NSFetchRequest alloc] init];
[request setEntity:[NSEntityDescription entityForName:@"EntityName" inManagedObjectContext:managedObjectContext]];
NSError *error = nil;
NSArray *results = [moc executeFetchRequest:request error:&error];
```
为fetchRequest 加上filter 和 sorter (否则取回来的东西顺序是随机的)
```objc
// filter
NSPredicate *predicate = [NSPredicate predicateWithFormat:@"salary > %i", basicSalary];
[request setPredicate:predicate];

// sorter
NSMutableArrary *sortArray = [NSMutableArray array] // 可能有多个sorter
[sortArray addObject:[[NSSortDescriptor alloc] initWithKey:@"name" ascending:YES]];
[request setSortDescriptor:sortArray];
```
NSManagedObject 拿到之后就可以用KVC的方法来set/get 里面的值（包括relationship, one-to-many的会得到一个装了相应mangedObject的NSSet）
```objc
[managedObject setValue:value forKey:@"key"];
[managedObject valueForKey:@"key"];
[managedObject valueForKey:@"someOneToManyRelationship"]; // return NSSet filled with cooresponding NSMangedObject on the other side of the relationship
// 如果需要改变（添加／删除）这个这个Set里面的一些Object
[managedObject mutableSetValueForKey:@"someOneToManyRelationship"]; // return a mutableSet allowing to add/delete object in the relationship
```
##### Insert
Insert 则是如下创建一个NSManagedObject 然后set 相应的值就好了
```objc
NSManagedObject *obj = [NSEntityDescription
                            insertNewObjectForEntityForName: @"ObjName"
                            inManagedObjectContext:context];
```

##### Save
最后Save 所有的changes, 就将数据持久化了
```objc
[context save:&error];
if (error) {
    ALog(...)
}
```

#### 更简单的用法
##### subclass NSMangedObject
用KVC来设置NSManagedObject 显然需要一大段code。而更简单的方法就是直接subclass NSManagedObject, 例如
```objc
@interface MyMO: NSManagedObject

@property (strong) NSString *name;
@property (strong) NSSet *someOneToManyRelationship; // relationship也一样
....
```
别忘了要设成dynamic, 因为这些properity 都是在运行时create 的
```objc
@dynamic name;
@dynamic someOneToManyRelationship;
...
```
然后就可以像直接取用了
```objc
MyMO obj = [...]
obj.name = "xxx"
...
```
超方便是不是👍

##### stored fetch request
而fetch request 也是可以存起来的，如下图添加一个fetch request，可以设置fetch的条件
![](/images/core-data-part-1-stored-fetch-request-pic.png)
用的时候..
```objc
NSFetchRequest *request = [[[moc persistentStoreCoordinator] managedObjectModel] fetchRequestTemplateForName:@"allDummiesNamedZert"];
```
