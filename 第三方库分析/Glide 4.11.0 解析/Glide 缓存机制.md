# **Glide 缓存机制分析**
 *以下分析基于 Gldie 4.11.0 版本源码。* 

Glide 的缓存主要分为两块：**内存缓存（Memory Cache）**和**磁盘盘存（Disk Cache）**。
内存缓存读取速度快，同时防止应用重复从磁盘读取图片数据。磁盘缓存防止应用重复从网络加载图片。

## Glide 的缓存
在开启图片加载任务之前，Glide 会做多重检查：
* **Active Resources（活动资源）**- 要加载的图片是否正在其他 View 上展示？

* **Memory Cache（内存缓存）**- 要加载的图片是否之前被加载到内存中了？

* **Resource（资源）**- 要加载的图片之前是否已经被解码、转换并写入磁盘缓存？

* **Data（数据）**- 之前是否已将图片的数据写入磁盘缓存中？

  前两步检查图片是否已经存在于内存缓存中，如果是则直接返回。后两步检查图片数据是否已保存在磁盘缓存中，如果是，则异步地返回数据。
  如果内存缓存和磁盘缓存中都没有找到，则 Glide 会从原始资源（如 Url，Uri 或者 File）中获取图片数据。
## CacheKeys
在 Glide V4 中，cache keys 至少包含以下两种元素：

**1、请求加载的模型（File、Url、Uri）。对于自定义模型，必须正确实现 `hashCode()` 和 `equals()` 两个方法。**
**2、签名（可选）**

对于上述前 3 步（Active Resources，Memory Cache，Resource）的 cache keys 也会包含一些其他信息：

* 宽高信息

* Transformation（变换）（可选）

* Options（选项）

* 请求的数据类型，如 Bitmap、GIF 等

  其中，内存缓存（Active Resources、Memory Cache）的 cache keys 和磁盘缓存所用的 cache keys 也存在细微差别，以适内存 Options（选项），如：影响 Bitmap 配置的选项或其他仅在解码时用到的参数。

## EngineKey
An in memory only cache key used to multiplex loads.（仅用于内存缓存的多路复用加载的 cache key）
在 Glide 的 into 方法中创建 SingleRequest 并开启任务时，在 Engine 的 load 方法中会调用 `EngineKeyFactory` 的 `buildKey` 方法，根据入参生成对应的 EngineKey。

```java
public <R> LoadStatus load(
      GlideContext glideContext,
      Object model,
      Key signature,
      int width,
      int height,
      Class<?> resourceClass,
      Class<R> transcodeClass,
      Priority priority,
      DiskCacheStrategy diskCacheStrategy,
      Map<Class<?>, Transformation<?>> transformations,
      boolean isTransformationRequired,
      boolean isScaleOnlyOrNoTransform,
      Options options,
      boolean isMemoryCacheable,
      boolean useUnlimitedSourceExecutorPool,
      boolean useAnimationPool,
      boolean onlyRetrieveFromCache,
      ResourceCallback cb,
      Executor callbackExecutor) {
    ...
    // 生成 cache key
    EngineKey key =
        keyFactory.buildKey(
            model,
            signature,
            width,
            height,
            transformations,
            resourceClass,
            transcodeClass,
            options);

    EngineResource<?> memoryResource;
    synchronized (this) {
        // 通过 key 从缓存中获取数据
        memoryResource = loadFromMemory(key, isMemoryCacheable, startTime);
        ...
    }
    ...
}
```
`EngineKeyFactory` 类：
```java
class EngineKeyFactory {
  @SuppressWarnings("rawtypes")
  EngineKey buildKey(
      Object model,
      Key signature,
      int width,
      int height,
      Map<Class<?>, Transformation<?>> transformations,
      Class<?> resourceClass,
      Class<?> transcodeClass,
      Options options) {
    return new EngineKey(
        model, signature, width, height, transformations, resourceClass, transcodeClass, options);
  }
}
```
buildKey 方法调用了 EngineKey 的构造函数来创建 EngineKey。`EngineKey` 中会保存传入的信息，并重写了 `hashCode()`  函数和 `equals()` 函数，保证只有所有信息相同才会生成相同的 EngineKey。

## 内存缓存
当生成了 EngineKey 后，在 Engine 的 load 方法中会通过 key 从内存缓存中读取数据。
```java
private EngineResource<?> loadFromMemory(
      EngineKey key, boolean isMemoryCacheable, long startTime) {
    // 1、如果没有开启内存缓存则直接返回 null
    if (!isMemoryCacheable) {
      return null;
    }

    // 2、从 Active Resouces 中读取缓存数据
    EngineResource<?> active = loadFromActiveResources(key);
    if (active != null) {
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Loaded resource from active resources", startTime, key);
      }
      return active;
    }

    // 3、从 LRUCache 中读取缓存数据
    EngineResource<?> cached = loadFromCache(key);
    if (cached != null) {
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Loaded resource from cache", startTime, key);
      }
      return cached;
    }

    // 4、没有缓存数据
    return null;
  }
```

### 1、ActiveResources 弱引用缓存
首先尝试从 ActiveResources 中查找缓存：
```java
private EngineResource<?> loadFromActiveResources(Key key) {
    EngineResource<?> active = activeResources.get(key);
    if (active != null) {
      active.acquire();
    }

    return active;
  }
```
其中 activeResources 是 ActiveResources 类型。在 Engine 的构造函数中，如果没有传入，则会创建一个 ActiveResources：
```java
Engine(
      MemoryCache cache,
      DiskCache.Factory diskCacheFactory,
      GlideExecutor diskCacheExecutor,
      GlideExecutor sourceExecutor,
      GlideExecutor sourceUnlimitedExecutor,
      GlideExecutor animationExecutor,
      Jobs jobs,
      EngineKeyFactory keyFactory,
      ActiveResources activeResources,
      EngineJobFactory engineJobFactory,
      DecodeJobFactory decodeJobFactory,
      ResourceRecycler resourceRecycler,
      boolean isActiveResourceRetentionAllowed) {
    ...
    if (activeResources == null) {
      activeResources = new ActiveResources(isActiveResourceRetentionAllowed);
    }
    this.activeResources = activeResources;
    activeResources.setListener(this);
    ...
}
```
ActiveResources 类内部维护了一个 HashMap，它的值是弱引用 `ResourceWeakReference`，用于保存弱引用缓存数据。
```java
...
final Map<Key, ResourceWeakReference> activeEngineResources = new HashMap<>();
...

ActiveResources(boolean isActiveResourceRetentionAllowed) {
    this(
        isActiveResourceRetentionAllowed,
        java.util.concurrent.Executors.newSingleThreadExecutor(
            new ThreadFactory() {
              @Override
              public Thread newThread(@NonNull final Runnable r) {
                return new Thread(
                    new Runnable() {
                      @Override
                      public void run() {
                        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                        r.run();
                      }
                    },
                    "glide-active-resources");
              }
            }));
}

ActiveResources(
      boolean isActiveResourceRetentionAllowed, Executor monitorClearedResourcesExecutor) {
    this.isActiveResourceRetentionAllowed = isActiveResourceRetentionAllowed;
    this.monitorClearedResourcesExecutor = monitorClearedResourcesExecutor;

    monitorClearedResourcesExecutor.execute(
        new Runnable() {
          @Override
          public void run() {
            cleanReferenceQueue();
          }
        });
}

...
synchronized EngineResource<?> get(Key key) {
    ResourceWeakReference activeRef = activeEngineResources.get(key);
    if (activeRef == null) {
      return null;
    }

    EngineResource<?> active = activeRef.get();
    if (active == null) {
      cleanupActiveReference(activeRef);
    }
    return active;
}
...
```
在 ActiveResources 的构造方法中会开启线程执行 `cleanReferenceQueue()` 方法，清除引用队列，进行初始化。get 方法返回一个 EngineResource 类型的对象，如果 activeEngineResources 中没有对应的 key 的值，则直接返回 null，否则获取 ResourceWeakReference 对应的 EngineSource，如果为 null，表示弱引用已被回收，则执行 `cleanupActiveReference` 方法：
```java
void cleanupActiveReference(@NonNull ResourceWeakReference ref) {
    synchronized (this) {
      activeEngineResources.remove(ref.key);

      if (!ref.isCacheable || ref.resource == null) {
        return;
      }
    }

    EngineResource<?> newResource =
        new EngineResource<>(
            ref.resource, /*isMemoryCacheable=*/ true, /*isRecyclable=*/ false, ref.key, listener);
    listener.onResourceReleased(ref.key, newResource);
  }
```
移除 `activeEngineResources` 中对应的 key。接着如果可以缓存并且对应的 resource 不为 null，则创建一个新的 EngineReource 并回调 onResourceReleased，该方法由 Engine 实现：
```java
public class Engine implements EngineJobListener, MemoryCache.ResourceRemovedListener, EngineResource.ResourceListener {
    ...
    public void onResourceReleased(Key cacheKey, EngineResource<?> resource) {
        activeResources.deactivate(cacheKey);
        if (resource.isMemoryCacheable()) {
          cache.put(cacheKey, resource);
        } else {
          resourceRecycler.recycle(resource, /*forceNextFrame=*/ false);
        }
    }
    ...
}


final class ActiveResources {
    ...
    synchronized void deactivate(Key key) {
        ResourceWeakReference removed = activeEngineResources.remove(key);
        if (removed != null) {
          removed.reset();
        }
    }
    ...
}
```
在 onResourceReleased 方法中首先调用了 ActivityResources 的 `deactivate` 方法，该方法用于清除 key 对应的资源。然后判断是否开启了缓存，如果开启了则将资源加入 MemoryCache 中，否则回收资源。

`ResouceWeakReference` 是 ActiveResources 的内部类，其中存储了图片资源和对应的 key：
```java
static final class ResourceWeakReference extends WeakReference<EngineResource<?>> {
    @SuppressWarnings("WeakerAccess")
    @Synthetic
    final Key key;

    @SuppressWarnings("WeakerAccess")
    @Synthetic
    final boolean isCacheable;

    @Nullable
    @SuppressWarnings("WeakerAccess")
    @Synthetic
    Resource<?> resource;

    ...
}
```
ResourceWeakReference 继承自 WeakReference，所以它是弱引用类型。
再次回到 Engine 的 loadFromActiveResouces 方法中，如果获取到的 EngineSource 对象不为 null，则会调用 EngineSource 的 acquire 方法：

```java
/**
   * Increments the number of consumers using the wrapped resource. Must be called on the main
   * thread.
   *
   * <p>This must be called with a number corresponding to the number of new consumers each time new
   * consumers begin using the wrapped resource. It is always safer to call acquire more often than
   * necessary. Generally external users should never call this method, the framework will take care
   * of this for you.
   */
  synchronized void acquire() {
    if (isRecycled) {
      throw new IllegalStateException("Cannot acquire a recycled resource");
    }
    ++acquired;
}
```
acquired 记录了资源被引用的次数，调用 acquire 方法说明资源被引用了一次。acquired 大于 0 说明资源正在被使用。

**总结：Active Resouces 是一种弱引用类型的缓存，其内部维护了一个 HashMap，值类型为 ResourceWeakReference，它是弱引用类型，内部持有资源和对应的 key。loadFromActiveResouces 用于从 ActiveResources 中获取 key 对应的 EngineSource 对象。如果弱引用被回收，则需要清理 HashMap 及其对应的资源，并创建一个新的 EngineSource 然后回调 onResouceReleased，如果开启了缓存则将新的 EngineSource 加入 MemoryCache 中，否则回收资源。最后更新资源引用次数，将其加一。**

### 2、LRUCache 内存缓存
如果 ActiveResource 中没有找到对应的缓存，则执行 `loadFromCache` 查找缓存数据。
```java
private EngineResource<?> loadFromCache(Key key) {
    EngineResource<?> cached = getEngineResourceFromCache(key);
    if (cached != null) {
      cached.acquire();
      activeResources.activate(key, cached);
    }
    return cached;
}
```
该方法中首先调用 getEngineResourceFromCache 方法获取 EngineSource：
```java
private EngineResource<?> getEngineResourceFromCache(Key key) {
    Resource<?> cached = cache.remove(key);

    final EngineResource<?> result;
    if (cached == null) {
      result = null;
    } else if (cached instanceof EngineResource) {
      // Save an object allocation if we've cached an EngineResource (the typical case).
      result = (EngineResource<?>) cached;
    } else {
      result =
          new EngineResource<>(
              cached, /*isMemoryCacheable=*/ true, /*isRecyclable=*/ true, key, /*listener=*/ this);
    }
    return result;
}
```
其中 cache 是 MemoryCache 类型，MemoryCache 是接口类型，它用于添加和移除内存缓存，cache 在构造 Engine 时传入。在创建 Glide 实例时创建了 MemoryCache 并传入了 Engine 中：
```java
Glide build(@NonNull Context context) {
    ...
    if (memoryCache == null) {
      memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
    }
    ...
    if (engine == null) {
      engine =
          new Engine(
              memoryCache,
              diskCacheFactory,
              diskCacheExecutor,
              sourceExecutor,
              GlideExecutor.newUnlimitedSourceExecutor(),
              animationExecutor,
              isActiveResourceRetentionAllowed);
    }
    ...
}
```
因此，Engine 中的 MemoryCache 具体是 `LruResourceCache`，它继承自 LruCache 并实现了 MemoryCache。
回到 getEngineResourceFromCache 方法中，如果内存缓存为空，则直接返回 null，如果返回类型为 EngineSource，则直接返回，否则创建一个新的 EngineSource 并返回。继续看 loadFromCache 方法，当获取到 EngineSource 后，如果不为 null，则调用 EngineSource 的 acquire 方法更新资源引用次数，然后执行 ActiveResources 的 activate 方法：
```java
synchronized void activate(Key key, EngineResource<?> resource) {
    ResourceWeakReference toPut =
        new ResourceWeakReference(
            key, resource, resourceReferenceQueue, isActiveResourceRetentionAllowed);

    ResourceWeakReference removed = activeEngineResources.put(key, toPut);
    if (removed != null) {
      removed.reset();
    }
}
```
在 activate 方法中，首先用 key 和其对应的资源创建了一个 ResourceWeakReference，并将它放入 activeEngineResources 中，并清除这个 key 之前对应的资源。

**总结：loadFromCache 用于从 LruCache 中获取一份 key 对应的缓存资源，如果获取到了则更新资源引用次数，然后创建一个弱引用并存入 ActiveResources 中，最后清除该 key 对应的上一份缓存资源。**

## 磁盘缓存
默认的 DiskCache 缓存策略：
```java
public static final DiskCacheStrategy AUTOMATIC =
      new DiskCacheStrategy() {
        @Override
        public boolean isDataCacheable(DataSource dataSource) {
          return dataSource == DataSource.REMOTE;
        }

        @Override
        public boolean isResourceCacheable(
            boolean isFromAlternateCacheKey, DataSource dataSource, EncodeStrategy encodeStrategy) {
          return ((isFromAlternateCacheKey && dataSource == DataSource.DATA_DISK_CACHE)
                  || dataSource == DataSource.LOCAL)
              && encodeStrategy == EncodeStrategy.TRANSFORMED;
        }

        @Override
        public boolean decodeCachedResource() {
          return true;
        }

        @Override
        public boolean decodeCachedData() {
          return true;
        }
      };
```
在 Engine 的 load 方法中如果没有从内存缓存中取得数据，则回调执行 `waitForExistingOrStartNewJob` 方法，在这个方法中会开启线程执行 DecodeJob 的 run 方法，在 run 方法中会执行 runWrapped 方法：
```java
private void runWrapped() {
    switch (runReason) {
      case INITIALIZE:
        stage = getNextStage(Stage.INITIALIZE);
        currentGenerator = getNextGenerator();
        runGenerators();
        break;
      case SWITCH_TO_SOURCE_SERVICE:
        runGenerators();
        break;
      case DECODE_DATA:
        decodeFromRetrievedData();
        break;
      default:
        throw new IllegalStateException("Unrecognized run reason: " + runReason);
    }
}
...
private Stage getNextStage(Stage current) {
    switch (current) {
      case INITIALIZE:
        return diskCacheStrategy.decodeCachedResource()
            ? Stage.RESOURCE_CACHE
            : getNextStage(Stage.RESOURCE_CACHE);
      case RESOURCE_CACHE:
        return diskCacheStrategy.decodeCachedData()
            ? Stage.DATA_CACHE
            : getNextStage(Stage.DATA_CACHE);
      case DATA_CACHE:
        // Skip loading from source if the user opted to only retrieve the resource from cache.
        return onlyRetrieveFromCache ? Stage.FINISHED : Stage.SOURCE;
      case SOURCE:
      case FINISHED:
        return Stage.FINISHED;
      default:
        throw new IllegalArgumentException("Unrecognized stage: " + current);
    }
}
...
private DataFetcherGenerator getNextGenerator() {
    switch (stage) {
      case RESOURCE_CACHE:
        return new ResourceCacheGenerator(decodeHelper, this);
      case DATA_CACHE:
        return new DataCacheGenerator(decodeHelper, this);
      case SOURCE:
        return new SourceGenerator(decodeHelper, this);
      case FINISHED:
        return null;
      default:
        throw new IllegalStateException("Unrecognized stage: " + stage);
    }
}
...
private void runGenerators() {
    currentThread = Thread.currentThread();
    startFetchTime = LogTime.getLogTime();
    boolean isStarted = false;
    while (!isCancelled
        && currentGenerator != null
        && !(isStarted = currentGenerator.startNext())) {
      stage = getNextStage(stage);
      currentGenerator = getNextGenerator();

      if (stage == Stage.SOURCE) {
        reschedule();
        return;
      }
    }
    // We've run out of stages and generators, give up.
    if ((stage == Stage.FINISHED || isCancelled) && !isStarted) {
      notifyFailed();
    }

    // Otherwise a generator started a new load and we expect to be called back in
    // onDataFetcherReady.
  }
```
默认 runReason 为 INITIALIZE，此时获取到的 stage 为 Stage.RESOURCE_CACHE，currentGenerator 为 ResourceCacheGenerator，接着执行 runGenerators 方法，在 runGenerators 方法的 while 循环中执行了 currentGenerator.startNext()，通过 ResourceCacheGenerator 无法获取磁盘缓存，此时再次执行 getNextStage 和 getNextGenerator 方法，currentGenerator 变成 `DataCacheGenerator` 并执行它的 startNext 方法：
```java
@Override
  public boolean startNext() {
    while (modelLoaders == null || !hasNextModelLoader()) {
      sourceIdIndex++;
      if (sourceIdIndex >= cacheKeys.size()) {
        return false;
      }
    
      Key sourceId = cacheKeys.get(sourceIdIndex);
      // PMD.AvoidInstantiatingObjectsInLoops The loop iterates a limited number of times
      // and the actions it performs are much more expensive than a single allocation.
      @SuppressWarnings("PMD.AvoidInstantiatingObjectsInLoops")
      // 创建 DataCacheKey
      Key originalKey = new DataCacheKey(sourceId, helper.getSignature());
      // 通过 DataCacheKey 获取对应的 File
      cacheFile = helper.getDiskCache().get(originalKey);
      if (cacheFile != null) {
        this.sourceKey = sourceId;
        modelLoaders = helper.getModelLoaders(cacheFile);
        modelLoaderIndex = 0;
      }
    }
    
    loadData = null;
    boolean started = false;
    while (!started && hasNextModelLoader()) {
      ModelLoader<File, ?> modelLoader = modelLoaders.get(modelLoaderIndex++);
      loadData =
          modelLoader.buildLoadData(
              cacheFile, helper.getWidth(), helper.getHeight(), helper.getOptions());
      if (loadData != null && helper.hasLoadPath(loadData.fetcher.getDataClass())) {
        started = true;
        // 加载缓存 File
        loadData.fetcher.loadData(helper.getPriority(), this);
      }
    }
    return started;
}
```
在 DataCacheGenerator 的 startNext 方法中，创建了 DataCacheKey，它用于原始数据的缓存 key。然后通过 key 获取缓存的 File。其中 helper.getDiskCache 通过 DiskCacheProvider 获取 DiskCache 对象，DiskCacheProvider 的实现类为 Engine 的内部类：
```java
private static class LazyDiskCacheProvider implements DecodeJob.DiskCacheProvider {

    private final DiskCache.Factory factory;
    private volatile DiskCache diskCache;

    LazyDiskCacheProvider(DiskCache.Factory factory) {
      this.factory = factory;
    }

    @VisibleForTesting
    synchronized void clearDiskCacheIfCreated() {
      if (diskCache == null) {
        return;
      }
      diskCache.clear();
    }

    @Override
    public DiskCache getDiskCache() {
      if (diskCache == null) {
        synchronized (this) {
          if (diskCache == null) {
            diskCache = factory.build();
          }
          if (diskCache == null) {
            diskCache = new DiskCacheAdapter();
          }
        }
      }
      return diskCache;
    }
}
```
getDiskCache 方法中通过 DiskCache.Factory 的 build 创建 DiskCache。DiskLruCacheFactory 实现了 DiskCache.Factory 接口：
```java
public class DiskLruCacheFactory implements DiskCache.Factory {
    ...
    @Override
    public DiskCache build() {
        File cacheDir = cacheDirectoryGetter.getCacheDirectory();
    
        if (cacheDir == null) {
          return null;
        }
    
        if (!cacheDir.mkdirs() && (!cacheDir.exists() || !cacheDir.isDirectory())) {
          return null;
        }
    
        return DiskLruCacheWrapper.create(cacheDir, diskCacheSize);
    }
}
```
其中调用了 `DiskLruCacheWrapper.create`：
```java
public class DiskLruCacheWrapper implements DiskCache {
    ...
    public static DiskCache create(File directory, long maxSize) {
        return new DiskLruCacheWrapper(directory, maxSize);
    }
    ...
    @Override
    public File get(Key key) {
        String safeKey = safeKeyGenerator.getSafeKey(key);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
          Log.v(TAG, "Get: Obtained: " + safeKey + " for for Key: " + key);
        }
        File result = null;
        try {
          // It is possible that the there will be a put in between these two gets. If so that shouldn't
          // be a problem because we will always put the same value at the same key so our input streams
          // will still represent the same data.
          final DiskLruCache.Value value = getDiskCache().get(safeKey);
          if (value != null) {
            result = value.getFile(0);
          }
        } catch (IOException e) {
          if (Log.isLoggable(TAG, Log.WARN)) {
            Log.w(TAG, "Unable to get from disk cache", e);
          }
        }
        return result;
    }
    ...
}
```
在这里创建了 DiskLruCacheWrapper 实例。所以回到 DataCacheGenerator 的 startNext 方法中，
`cacheFile = helper.getDiskCache().get(originalKey);` 获取到的是磁盘中缓存的 File 文件，然后通过该文件获取 ModelLoader，ModelLoader 的实现类是 ByteBufferFileLoader：
```java
public class ByteBufferFileLoader implements ModelLoader<File, ByteBuffer> {
    ...
    @Override
    public LoadData<ByteBuffer> buildLoadData(
          @NonNull File file, int width, int height, @NonNull Options options) {
        return new LoadData<>(new ObjectKey(file), new ByteBufferFetcher(file));
      }
    ...
    private static final class ByteBufferFetcher implements DataFetcher<ByteBuffer> {
    
        private final File file;
    
        @Synthetic
        @SuppressWarnings("WeakerAccess")
        ByteBufferFetcher(File file) {
          this.file = file;
        }
    
        @Override
        public void loadData(
            @NonNull Priority priority, @NonNull DataCallback<? super ByteBuffer> callback) {
          ByteBuffer result;
          try {
            result = ByteBufferUtil.fromFile(file);
          } catch (IOException e) {
            if (Log.isLoggable(TAG, Log.DEBUG)) {
              Log.d(TAG, "Failed to obtain ByteBuffer for file", e);
            }
            callback.onLoadFailed(e);
            return;
          }
    
          callback.onDataReady(result);
        }
        ...
      }
}
```
所以在 DataCacheGenerator 的 startNext 方法中执行 `loadData.fetcher.loadData(helper.getPriority(), this);` 最终执行的是 ByteBufferFileLoader 的内部类 ByteBufferFetcher 的 loadData 方法，其中将 File 读取成 ByteBuffer 并回调了 onDataReady，这里的 callback 是 DataCacheGenerator：
```java
@Override
public void onDataReady(Object data) {
    cb.onDataFetcherReady(sourceKey, data, loadData.fetcher, DataSource.DATA_DISK_CACHE, sourceKey);
}
```
这里的 cb 是 DecodeJob：
```java
@Override
public void onDataFetcherReady(
  Key sourceKey, Object data, DataFetcher<?> fetcher, DataSource dataSource, Key attemptedKey) {
    ...
    if (Thread.currentThread() != currentThread) {
      ...
    } else {
      GlideTrace.beginSection("DecodeJob.decodeFromRetrievedData");
      try {
        decodeFromRetrievedData();
      } finally {
        GlideTrace.endSection();
      }
    }
}
```
进入 `decodeFromRetrievedData` 方法：
```java
private void decodeFromRetrievedData() {
    ...
    Resource<R> resource = null;
    try {
      resource = decodeFromData(currentFetcher, currentData, currentDataSource);
    } catch (GlideException e) {
      e.setLoggingDetails(currentAttemptingKey, currentDataSource);
      throwables.add(e);
    }
    if (resource != null) {
      notifyEncodeAndRelease(resource, currentDataSource);
    } else {
      runGenerators();
    }
}
```
这里 resource 不为 null，因此执行 `notifyEncodeAndRelease` 方法：
```java
private void notifyEncodeAndRelease(Resource<R> resource, DataSource dataSource) {
    ...
    notifyComplete(result, dataSource);
    ...
}
...
private void notifyComplete(Resource<R> resource, DataSource dataSource) {
    setNotifiedOrThrow();
    callback.onResourceReady(resource, dataSource);
}
```
此后将进入前文 load 方法的流程，将数据一步步回调到 SingleRequest 并最终显示到 View 上。

**总结：通过 DataCacheGenerator 获取 DiskCache，实际是 DiskLruCacheWrapper，并通过 DiskCache 获取磁盘缓存中的 File 文件，然后通过 ByteBufferFileLoader 的内部类 ByteBufferFetcher 读取 File 生成 ByteBuffer 并回调 DataCacheGenerator 的 onDataReady，最终会一步步回调到 SingleRequest 中，然后回调进入 Target 的 onResourceReady，并最终调用 Target 的 setResource 方法显示图片。**

## 磁盘缓存的写入时机
如上文分析，在 DecodeJob 的 `runGenerators()` 方法中，如果 DataCacheGenerator 无法获取到磁盘缓存，将继续执行 `getNextStage` 和 `getNextGenerator` 方法，此时 stage 将变为 Stage.SOURCE，currentGenerator 将变为 SourceGenerator，如 Glide 的 load 方法分析中那样，从网络加载完成后将回调到：
```java
@Synthetic
void onDataReadyInternal(LoadData<?> loadData, Object data) {
    DiskCacheStrategy diskCacheStrategy = helper.getDiskCacheStrategy();
    if (data != null && diskCacheStrategy.isDataCacheable(loadData.fetcher.getDataSource())) {
      dataToCache = data;
      // We might be being called back on someone else's thread. Before doing anything, we should
      // reschedule to get back onto Glide's thread.
      cb.reschedule();
    } else {
      ...
}
```
开启了缓存后将保存 data 到 dataToCache 变量，并回调 DecodeJob 的 `reschedule` 方法：
```java
@Override
public void reschedule() {
    runReason = RunReason.SWITCH_TO_SOURCE_SERVICE;
    callback.reschedule(this);
}
```
这里将 runReason 修改为 RunReason.SWITCH_TO_SOURCE_SERVICE，然后回调 EngineJob 的 `reschedule`：
```java
@Override
public void reschedule(DecodeJob<?> job) {
    // Even if the job is cancelled here, it still needs to be scheduled so that it can clean itself
    // up.
    getActiveSourceExecutor().execute(job);
}
```
可见最终还是执行 DecodeJob 的 run 方法，进入 runWrapped 方法：
```java
private void runWrapped() {
    switch (runReason) {
      ...
      case SWITCH_TO_SOURCE_SERVICE:
        runGenerators();
        break;
      ...
    }
}
```
将执行 runGenerators 方法，此时的 currentGenerator 是 SourceGenerator，所以将会执行 SourceGenerator 的 startNext 方法：
```java
@Override
public boolean startNext() {
    if (dataToCache != null) {
      Object data = dataToCache;
      dataToCache = null;
      cacheData(data);
    }
    ...
}
```
上文中提到，在网络加载完成后回调 onDataReadyInternal 是将数据赋值给了 dataToCache，所以此时 dataToCache 不为 null，执行 cacheData 方法：
```java
private void cacheData(Object dataToCache) {
    long startTime = LogTime.getLogTime();
    try {
      Encoder<Object> encoder = helper.getSourceEncoder(dataToCache);
      DataCacheWriter<Object> writer =
          new DataCacheWriter<>(encoder, dataToCache, helper.getOptions());
      originalKey = new DataCacheKey(loadData.sourceKey, helper.getSignature());
      helper.getDiskCache().put(originalKey, writer);
      ...
}
```
创建 DataCacheWriter，DataCacheKey，如前文分析，`helper.getDiskCache().put(originalKey, writer);` 执行的是 DiskLruCacheWrapper 的 put 方法，将文件写入磁盘：
```java
@Override
public void put(Key key, Writer writer) {
    // We want to make sure that puts block so that data is available when put completes. We may
    // actually not write any data if we find that data is written by the time we acquire the lock.
    String safeKey = safeKeyGenerator.getSafeKey(key);
    writeLocker.acquire(safeKey);
    try {
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        Log.v(TAG, "Put: Obtained: " + safeKey + " for for Key: " + key);
      }
      try {
        // We assume we only need to put once, so if data was written while we were trying to get
        // the lock, we can simply abort.
        DiskLruCache diskCache = getDiskCache();
        Value current = diskCache.get(safeKey);
        if (current != null) {
          return;
        }
    
        DiskLruCache.Editor editor = diskCache.edit(safeKey);
        if (editor == null) {
          throw new IllegalStateException("Had two simultaneous puts for: " + safeKey);
        }
        try {
          File file = editor.getFile(0);
          if (writer.write(file)) {
            editor.commit();
          }
        } finally {
          editor.abortUnlessCommitted();
        }
      } catch (IOException e) {
        if (Log.isLoggable(TAG, Log.WARN)) {
          Log.w(TAG, "Unable to put to disk cache", e);
        }
      }
    } finally {
      writeLocker.release(safeKey);
    }
}
```


