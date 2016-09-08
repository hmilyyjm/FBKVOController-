#FBKVOController源码分析
##背景
Key-Value Observing ，通过在特定对象之间监听一个特定的 keypath 的改变进行事件内省。例如：一个 ProgressView 可以观察 网络请求的 numberOfBytesRead 来更新它自己的 progress 属性。

这个方案已经被明确定义，获得框架级支持，可以方便地采用。开发人员不需要添加任何代码，不需要设计自己的观察者模型，直接可以在工程里使用。其次，KVO的架构非常的强大，可以很容易的支持多个观察者观察同一个属性，以及相关的值。

但是：
Apple原生KVO也有一些显而易见的缺点。

1. 添加和移除观察者要配对出现。移除一个未添加的观察者，程序会crash；重复添加观察者会造成回调方法多次调用，给程序造成逻辑上的混乱。
2. 添加观察者，移除观察者，通知回调，三块儿代码过于分散。

那么，有没有改良版的KVO呢？Facebook出品的FBKVOController是同类中我觉得最好用，且源码简单，设计感好。

ps:有人估计会提到ReactiveCocoa，我个人并不建议在项目中使用这么庞大一个类库，一是学习成本太大，不利于开展团队协作。二是一旦出现bug，那就哭吧。
##FBKVOController
简单来说，FBKVOController是对KVO机制的一层封装，同时提供了线程安全的特性和并对如下这个臭名昭著的函数进行了封装，提供了干净的block的回调，避免了处理这个函数的逻辑散落的到处都是。

```
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
```

##先看用法，再看源码


```
// create KVO controller with observer
FBKVOController *KVOController = [FBKVOController controllerWithObserver:self];
self.KVOController = KVOController;

// observe clock date property
[self.KVOController observe:clock keyPath:@"date" options:NSKeyValueObservingOptionInitial|NSKeyValueObservingOptionNew block:^(ClockView *clockView, Clock *clock, NSDictionary *change) {

  // update clock view with new value
  clockView.date = change[NSKeyValueChangeNewKey];
}];
```
使用非常简单，提供了block回调。而且，并不需要考虑remove observer的事情。FBKVOController实现了“自释放”。关于“自释放”的概念会在文后予以详细解释。


##FBKVOController源码


从使用者调用API看起。

```
@implementation FBKVOController
{
  NSMapTable<id, NSMutableSet<_FBKVOInfo *> *> *_objectInfosMap;
  pthread_mutex_t _lock;
}


+ (instancetype)controllerWithObserver:(nullable id)observer
{
  return [[self alloc] initWithObserver:observer];
}

- (instancetype)initWithObserver:(nullable id)observer retainObserved:(BOOL)retainObserved
{
  self = [super init];
  if (nil != self) {
    _observer = observer;
    NSPointerFunctionsOptions keyOptions = retainObserved ? NSPointerFunctionsStrongMemory|NSPointerFunctionsObjectPointerPersonality : NSPointerFunctionsWeakMemory|NSPointerFunctionsObjectPointerPersonality;
    _objectInfosMap = [[NSMapTable alloc] initWithKeyOptions:keyOptions valueOptions:NSPointerFunctionsStrongMemory|NSPointerFunctionsObjectPersonality capacity:0];
    pthread_mutex_init(&_lock, NULL);
  }
  return self;
}

- (instancetype)initWithObserver:(nullable id)observer
{
  return [self initWithObserver:observer retainObserved:YES];
}
```

1. 首先我们看到，这个对象持有一个pthread_mutex_t及一个NSMapTable。其中pthread_mutex_t即为互斥锁，互斥锁通过确保一次只有一个线程执行代码的临界段来同步多个线程。互斥锁还可以保护单线程代码。。NSMapTable可能大家接触不是很多，我在后文会详细介绍，这里大家可以先理解为一个高级的NSDictionary。
2. 在构造函数中，首先将传入的observer进行weak持有，这主要为了避免Retain Cycle。
3. 这一段的内容可能大家不太熟悉，NSPointerFunctionsOptions简单来说就是定义NSMapTable中的key和value采用何种内存管理策略，包括strong强引用，weak弱引用以及copy（要支持NSCopying协议）
4. 初始化互斥锁

然后是监听API.

```
- (void)observe:(nullable id)object keyPath:(NSString *)keyPath options:(NSKeyValueObservingOptions)options block:(FBKVONotificationBlock)block
{
  NSAssert(0 != keyPath.length && NULL != block, @"missing required parameters observe:%@ keyPath:%@ block:%p", object, keyPath, block);
  if (nil == object || 0 == keyPath.length || NULL == block) {
    return;
  }

  // create info
  _FBKVOInfo *info = [[_FBKVOInfo alloc] initWithController:self keyPath:keyPath options:options block:block];

  // observe object with info
  [self _observe:object info:info];
}
```

1. 对于传入的参数，构建一个内部的FBKVOInfo数据结构
2. 调用[self _observe:object info:info];

接下来，我们来跟踪一下[self _observe:object info:info];，内容如下：

```
- (void)_observe:(id)object info:(_FBKVOInfo *)info
{
  // lock
  pthread_mutex_lock(&_lock);

  NSMutableSet *infos = [_objectInfosMap objectForKey:object];

  // check for info existence
  _FBKVOInfo *existingInfo = [infos member:info];
  if (nil != existingInfo) {
    // observation info already exists; do not observe it again

    // unlock and return
    pthread_mutex_unlock(&_lock);
    return;
  }

  // lazilly create set of infos
  if (nil == infos) {
    infos = [NSMutableSet set];
    [_objectInfosMap setObject:infos forKey:object];
  }

  // add info and oberve
  [infos addObject:info];

  // unlock prior to callout
  pthread_mutex_unlock(&_lock);

  [[_FBKVOSharedController sharedController] observe:object info:info];
}
```
1. 根据被观察的object获取其对应的infos set。这个主要作用在于避免多次对同一个keyPath添加多次观察。因为每调用一次addObserverForKeyPath就要有一个对应的removeObserverForKey。

2. 从infos set判断是不是已经有了与此次info相同的观察。

3. 如果以上都顺利通过，将观察的信息及关系注册到_FBKVOSharedController中。

至此，FBKVOController的任务基本都结束，unObserve相关的任务逻辑大同小异，不再赘述。

##FBKVOSharedController

上述代码中有一个比较有意识的地方，[[_FBKVOSharedController sharedController] observe:object info:info];有一个单例类
_FBKVOSharedController。
将所有的观察信息统一交由一个FBKVOSharedController的单例进行维护。

```

+ (instancetype)sharedController
{
  static _FBKVOSharedController *_controller = nil;
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    _controller = [[_FBKVOSharedController alloc] init];
  });
  return _controller;
}

- (instancetype)init
{
  self = [super init];
  if (nil != self) {
    NSHashTable *infos = [NSHashTable alloc];
#ifdef __IPHONE_OS_VERSION_MIN_REQUIRED
    _infos = [infos initWithOptions:NSPointerFunctionsWeakMemory|NSPointerFunctionsObjectPointerPersonality capacity:0];
#elif defined(__MAC_OS_X_VERSION_MIN_REQUIRED)
    if ([NSHashTable respondsToSelector:@selector(weakObjectsHashTable)]) {
      _infos = [infos initWithOptions:NSPointerFunctionsWeakMemory|NSPointerFunctionsObjectPointerPersonality capacity:0];
    } else {
      // silence deprecated warnings
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wdeprecated-declarations"
      _infos = [infos initWithOptions:NSPointerFunctionsZeroingWeakMemory|NSPointerFunctionsObjectPointerPersonality capacity:0];
#pragma clang diagnostic pop
    }

#endif
    pthread_mutex_init(&_mutex, NULL);
  }
  return self;
}
```
1. 在单例的初始化方法中有一个NSHashTable，同NSMapTable一样，可能大家接触不是很多，我在后文会详细介绍，这里大家可以先理解为一个高级的NSSet。
2. NSPointerFunctionsZeroingWeakMemory简单来说就是定义NSHashTable中的元素采用何种内存管理策略


于是，通过如下方法，我们像使用注册表一样将对KVOInfo注册。


```
- (void)observe:(id)object info:(nullable _FBKVOInfo *)info
{
  if (nil == info) {
    return;
  }

  // register info
  pthread_mutex_lock(&_mutex);
  [_infos addObject:info];
  pthread_mutex_unlock(&_mutex);

  // add observer
  [object addObserver:self forKeyPath:info->_keyPath options:info->_options context:(void *)info];

  if (info->_state == _FBKVOInfoStateInitial) {
    info->_state = _FBKVOInfoStateObserving;
  } else if (info->_state == _FBKVOInfoStateNotObserving) {
    // this could happen when `NSKeyValueObservingOptionInitial` is one of the NSKeyValueObservingOptions,
    // and the observer is unregistered within the callback block.
    // at this time the object has been registered as an observer (in Foundation KVO),
    // so we can safely unobserve it.
    [object removeObserver:self forKeyPath:info->_keyPath context:(void *)info];
  }
}
```

1. 代表所有的观察信息都首先由FBKVOSharedController进行接受，随后进行转发。
2. 对应的添加代码 有一个 移除代码，设计的相当细心啊

实现observeValueForKeyPath:ofObject:Change:context
来接收通知。

```
- (void)observeValueForKeyPath:(nullable NSString *)keyPath
                      ofObject:(nullable id)object
                        change:(nullable NSDictionary<NSString *, id> *)change
                       context:(nullable void *)context
{
  NSAssert(context, @"missing context keyPath:%@ object:%@ change:%@", keyPath, object, change);

  _FBKVOInfo *info;

  {
    // lookup context in registered infos, taking out a strong reference only if it exists
    pthread_mutex_lock(&_mutex);
    info = [_infos member:(__bridge id)context];
    pthread_mutex_unlock(&_mutex);
  }

  if (nil != info) {

    // take strong reference to controller
    FBKVOController *controller = info->_controller;
    if (nil != controller) {

      // take strong reference to observer
      id observer = controller.observer;
      if (nil != observer) {

        // dispatch custom block or action, fall back to default action
        if (info->_block) {
          NSDictionary<NSString *, id> *changeWithKeyPath = change;
          // add the keyPath to the change dictionary for clarity when mulitple keyPaths are being observed
          if (keyPath) {
            NSMutableDictionary<NSString *, id> *mChange = [NSMutableDictionary dictionaryWithObject:keyPath forKey:FBKVONotificationKeyPathKey];
            [mChange addEntriesFromDictionary:change];
            changeWithKeyPath = [mChange copy];
          }
          info->_block(observer, object, changeWithKeyPath);
        } else if (info->_action) {
pragma clang diagnostic push
pragma clang diagnostic ignored "-Warc-performSelector-leaks"
          [observer performSelector:info->_action withObject:change withObject:object];
pragma clang diagnostic pop
        } else {
          [observer observeValueForKeyPath:keyPath ofObject:object change:change context:info->_context];
        }
      }

  }
}

```

1. 根据context上下文获取对应的KVOInfo
2. 判断当前info的observer和controller，是否仍然存在（因为之前我们采用的weak持有）
3. 根据 info的block或者selector或者overwrite进行消息转发。

FBKVOController整体的实现就介绍完了.FBKVOController给我的感觉是，局部有写代码我也会自己实现，但是作为一个整体，真心觉得还是一个字：服！

##填前面提到的坑
###自释放
何为“自释放”？可以简单的理解为对象在生命周期结束后自动清理回收所有与其相关的资源或链接，这个清理不仅仅包括对象内存的回收，还包括对象解耦以及附属事件的清理等，比如定时器的自我停止、KVO对象的监听移除等。
那么，FBKVOController是如何做到自释放的？可以归纳为四个字——动态属性。其为观察者绑定动态属性self.KVOController，动态绑定的KVOController会随着观察者的释放而释放，KVOController在自己的dealloc函数中移除KVO监听，巧妙的将观察者的remove转移到其动态属性的dealloc函数中。

但是其还是有一定的局限性——对象无法监听自己的属性，如果你的代码是这样的

```

[self.KVOController observe:self keyPath:@"date" options:NSKeyValueObservingOptionNew block:^(NSDictionary *change) {
    // to do
}];
```

很遗憾，循环引用的问题又出现，因为FBKVOController中的NSMapTable对象会retain key对象，具体代码如下

```

[_objectInfosMap setObject:infos forKey:object];
```

###NSHash​Table & NSMap​Table
有一篇文章讲的不多，直接贴链接吧
[@click](http://nshipster.cn/nshashtable-and-nsmaptable/) 

##总结
FBKVOController对于喜好使用kvo的工程师来说，是一个好的，精简的开发框架。源码优雅，可读性高，利于自己维护。

优点如下：

1. 提供了干净的block的回调，避免了处理这个函数的逻辑散落的到处都是。
2. 不用担心remove问题，不用再在dealloc中写remove代码。当然，如果你需要在其他时机进行remove observer,你大可放心的remove，不会出现因为没有添加而crash的问题

缺点：

1. 对象无法监听自己的属性，否则会出现循环引用。但是这个case 很少见吧？哈哈~~~

以上！
