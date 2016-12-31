---
title: Learning Cocoa with Objc reading notes
date: 2015-11-13 23:48:59
tags:
- iOS
---
Learning Cocoa with Objective-C 这本书我差不多读了一半，感觉这本书比较像一本参考书，要做某个功能可以参考下了解一下基本的东西。跳过了一些章节，把自己之前不知道的一些东西记一下。

#### NSData
NSData contains bytes with no assumptions of the type of data.
```objc
//load file into NSData
NSData *loadedData = [NSData dataWithContentsOfFile:<filePath>];
```
NSData itself is useless
```objc
NSString *loadedString = [[NSString alloc] initWithData:loadedData
                                        encoding:NSUTF8Encoding];
//encoding:NSUTF8Encoding tells NSString how to interrprate the bytes in NSData
```

#### NSCoding
Serialization and Deserialization in Cocoa
```objc
// confirming to NSCoding
- (void)encodeWithCoder:(NSKeyedArchiver *)aCoder{
    [aCoder encodeObject:<obj> forKey:<key>];
}

- (void)initWithCoder:(NSKeyedArchiver *)aCoder{
    ...
    _obj = [aDecoder decodeObjectForKey:<key>]
    return self;
}
```
Working with NSData
```objc
// store
NSData *storedData = [NSKeyedArchiver archivedDataWithRootObject:myObject];
[storedData writeToFile:<filePath> atomically:YES];

// retrive
SomeObj *obj = [NSKeyedUnarchiver unarchiveObjectWithData:storedData];
```

#### Using NSBundle to find resources
```objc
NSString *path = [[NSBundle mainBundle] pathForResource:@"SomeImage" ofType:@"png"];
```

#### MultiTasking on iOS
```objc
- (void)applicationDidEnterBacktround:(UIApplication *)application
```
Any task like saving to disk needs to be performed at this moment. Background app can be killed at anytime and no code can be executed at while being suspended.

#### Request to run in background
App can request to run in background for no more 10 mins
```objc
- (void)applicationDidEnterBackground:(UIApplication *)application
{
    // beginBackgroundTaskWithExpirationHandler returns a unqie ident for the task
    bgTask = [application beginBackgroundTaskWithExpirationHandler:^{
        [application endBackgroundTask:bgTask];
        bgTask = UIBackgroundTaskInvalid;
    }

    dispatch_async(dispatch_get_global_queue(DISPTACH_QUEU_PRIORITY_DEFAULT, 0), ^{
        ....bgTasks... 
        [application endBackgroundTask:bgTask];
        bgTask = UIBackgroundTaskInvalid;
    })
}
```
Tasks that can run for a longer time
1. playing audio
2. tracking location
3. VoIP

#### Nib
Nib – “freeze-drying” objects and storing them in a serialized form inside the file.
When displaying, loads the nib file, “rehydrates” the stored bojects, and presents them to the user.

#### How Nib Files are Loaded
1. re-create every objects in the nib
2. connect evey outlet
3. every single object loaded receives awakeFromNib: message

#### Blocks
```objc
<return type>(^ <variable name>) (<params>)
```
Blocks exist in stack when created, thus then assign to like instance variable, the block needs to be copied to the heap.
```objc
_var = [bomeBlock copy];
```
Blocks keep strong ref to variables captured.
To modify captured variables, add __block modifier.

```objc
__block int i = 0;
```

#### KVC
1. accessing key that doesn’t exist causes exception
2. is able to access even private vriables
```objc
// determine the varibale to set/get at run time
for (NSString * key in dict) {
    NSObject *value = [dict objectForKey:key];
    [aProduct setValue:value forKey:key];
}
```

#### KVO
1. synthesized property auto notify changes when setter is called
2. overried setter -> needs to fire notifiction manually
```objc
- (void)setName:(NSString *)name {
    [self willChangeValueForKey:@"name"];
    _name = name;
    [self didChangeValueForKey:@"name"];
}
```

#### NSNotification
broadcast notifications, typically one singleton notification center. 
