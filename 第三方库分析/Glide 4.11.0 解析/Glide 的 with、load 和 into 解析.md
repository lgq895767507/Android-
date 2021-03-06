# 解析 Glide 的 with、load、into 方法
以下源码分析基于 Glide 4.11.0 版本。
## Glide 的使用
```
Glide.with(Activity).load("an image url").into(ImageView)
```
# with 方法
with 方法有个很多重载方法，以传入 `FragmentActivity` 为例：
```
/**
   * Begin a load with Glide that will tied to the give {@link
   * androidx.fragment.app.FragmentActivity}'s lifecycle and that uses the given {@link
   * androidx.fragment.app.FragmentActivity}'s default options.
   *
   * @param activity The activity to use.
   * @return A RequestManager for the given FragmentActivity that can be used to start a load.
   */
  @NonNull
  public static RequestManager with(@NonNull FragmentActivity activity) {
    return getRetriever(activity).get(activity);
  }
```
with 方法中首先调用了 `getRetriever` 方法：
```
@NonNull
  private static RequestManagerRetriever getRetriever(@Nullable Context context) {
    // Context could be null for other reasons (ie the user passes in null), but in practice it will
    // only occur due to errors with the Fragment lifecycle.
    Preconditions.checkNotNull(
        context,
        "You cannot start a load on a not yet attached View or a Fragment where getActivity() "
            + "returns null (which usually occurs when getActivity() is called before the Fragment "
            + "is attached or after the Fragment is destroyed).");
    return Glide.get(context).getRequestManagerRetriever();
  }
```
该方法返回了 Glide 的 RequestManagerRetriever，因此，with 方法最终调用的是 RequestManagerRetriever 的 `get` 方法，RequestManagerRetriever 的 get 方法同样有很多重载方法，还是以传入 FragmentActivity 为例：
```
@NonNull
  public RequestManager get(@NonNull FragmentActivity activity) {
    if (Util.isOnBackgroundThread()) {
      return get(activity.getApplicationContext());
    } else {
      assertNotDestroyed(activity);
      FragmentManager fm = activity.getSupportFragmentManager();
      return supportFragmentGet(activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
    }
  }
```
首先对当前线程进行了判断，如果是后台线程，则当做传入 Application 处理，最终通过调用 `getApplicationManager` 方法返回 RequestManager。否则先判断当前 activity 是否已销毁，然后调用了 `supportFragmentGet` 方法：
```
@NonNull
  private RequestManager supportFragmentGet(
      @NonNull Context context,
      @NonNull FragmentManager fm,
      @Nullable Fragment parentHint,
      boolean isParentVisible) {
    SupportRequestManagerFragment current =
        getSupportRequestManagerFragment(fm, parentHint, isParentVisible);
    RequestManager requestManager = current.getRequestManager();
    if (requestManager == null) {
      // TODO(b/27524013): Factor out this Glide.get() call.
      Glide glide = Glide.get(context);
      requestManager =
          factory.build(
              glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
      current.setRequestManager(requestManager);
    }
    return requestManager;
  }
```
在该方法中首先调用了 `getSupportRequestManagerFragment` 方法，该方法主要用于向当前 activity 或 fragment 中添加一个 SupportRequestManagerFragment。SupportRequestManagerFragment 用于存储 RequestManager 并管理 Glide 的请求。并最终返回了 RequestManager。
```
@NonNull
  private SupportRequestManagerFragment getSupportRequestManagerFragment(
      @NonNull final FragmentManager fm, @Nullable Fragment parentHint, boolean isParentVisible) {
    SupportRequestManagerFragment current =
        (SupportRequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
    if (current == null) {
      current = pendingSupportRequestManagerFragments.get(fm);
      if (current == null) {
        current = new SupportRequestManagerFragment();
        current.setParentFragmentHint(parentHint);
        if (isParentVisible) {
          current.getGlideLifecycle().onStart();
        }
        pendingSupportRequestManagerFragments.put(fm, current);
        fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
        handler.obtainMessage(ID_REMOVE_SUPPORT_FRAGMENT_MANAGER, fm).sendToTarget();
      }
    }
    return current;
  }
```
**总结：Glide 的 with 方法主要作用：1、实例化 Glide。2、添加 fragment 用于管理 Glide 的请求。3、获取 RequestManager。**

## Glide 实例的创建过程
上述 `getRetriever` 中首先对 context 进行了判空，然后调用了 `Glide.get(context).getRequestManagerRetriever()`，`Glide.with(context)` 方法：
```
/**
   * Get the singleton.
   *
   * @return the singleton
   */
  @NonNull
  public static Glide get(@NonNull Context context) {
    if (glide == null) {
      GeneratedAppGlideModule annotationGeneratedModule =
          getAnnotationGeneratedGlideModules(context.getApplicationContext());
      synchronized (Glide.class) {
        if (glide == null) {
          checkAndInitializeGlide(context, annotationGeneratedModule);
        }
      }
    }

    return glide;
  }
```
双空判断并调用 `checkAndInitializeGlide` 创建了 Glide 实例：
```
@GuardedBy("Glide.class")
  private static void checkAndInitializeGlide(
      @NonNull Context context, @Nullable GeneratedAppGlideModule generatedAppGlideModule) {
    // In the thread running initGlide(), one or more classes may call Glide.get(context).
    // Without this check, those calls could trigger infinite recursion.
    if (isInitializing) {
      throw new IllegalStateException(
          "You cannot call Glide.get() in registerComponents(),"
              + " use the provided Glide instance instead");
    }
    isInitializing = true;
    initializeGlide(context, generatedAppGlideModule);
    isInitializing = false;
  }
```
调用了 `initializeGlide`：
```
  @GuardedBy("Glide.class")
  @SuppressWarnings("deprecation")
  private static void initializeGlide(
      @NonNull Context context,
      @NonNull GlideBuilder builder,
      @Nullable GeneratedAppGlideModule annotationGeneratedModule) {
        ...
        RequestManagerRetriever.RequestManagerFactory factory =
        annotationGeneratedModule != null
            ? annotationGeneratedModule.getRequestManagerFactory()
            : null;
        builder.setRequestManagerFactory(factory);
        ...
        Glide glide = builder.build(applicationContext);
        ...
        applicationContext.registerComponentCallbacks(glide);
        Glide.glide = glide;
  }
```
在该方法中为 `GlideBuilder` 设置了工厂类，并调用 `GlideBuilder` 的 `build` 方法实际创建 Glide 实例，并为其注册了 Application 组件生命周期回调。
```
  @NonNull
  Glide build(@NonNull Context context) {
    if (sourceExecutor == null) {
      sourceExecutor = GlideExecutor.newSourceExecutor();
    }

    if (diskCacheExecutor == null) {
      diskCacheExecutor = GlideExecutor.newDiskCacheExecutor();
    }

    if (animationExecutor == null) {
      animationExecutor = GlideExecutor.newAnimationExecutor();
    }

    if (memorySizeCalculator == null) {
      memorySizeCalculator = new MemorySizeCalculator.Builder(context).build();
    }

    if (connectivityMonitorFactory == null) {
      connectivityMonitorFactory = new DefaultConnectivityMonitorFactory();
    }

    if (bitmapPool == null) {
      int size = memorySizeCalculator.getBitmapPoolSize();
      if (size > 0) {
        bitmapPool = new LruBitmapPool(size);
      } else {
        bitmapPool = new BitmapPoolAdapter();
      }
    }

    if (arrayPool == null) {
      arrayPool = new LruArrayPool(memorySizeCalculator.getArrayPoolSizeInBytes());
    }

    if (memoryCache == null) {
      memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
    }

    if (diskCacheFactory == null) {
      diskCacheFactory = new InternalCacheDiskCacheFactory(context);
    }

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

    if (defaultRequestListeners == null) {
      defaultRequestListeners = Collections.emptyList();
    } else {
      defaultRequestListeners = Collections.unmodifiableList(defaultRequestListeners);
    }

    RequestManagerRetriever requestManagerRetriever =
        new RequestManagerRetriever(requestManagerFactory);

    return new Glide(
        context,
        engine,
        memoryCache,
        bitmapPool,
        arrayPool,
        requestManagerRetriever,
        connectivityMonitorFactory,
        logLevel,
        defaultRequestOptionsFactory,
        defaultTransitionOptions,
        defaultRequestListeners,
        isLoggingRequestOriginsEnabled,
        isImageDecoderEnabledForBitmaps);
  }
```
在 `GlideBuilder` 的 `build` 方法中为 Glide 初始化了设置，如线程池、Bitmap 池、缓存等，最后调用了 Glide 的构造方法创建了 Glide 实例。

## load 方法
load 方法同样有很多重载方法，以传入 String 类型参数为例：
```
/**
   * Equivalent to calling {@link #asDrawable()} and then {@link RequestBuilder#load(String)}.
   *
   * @return A new request builder for loading a {@link Drawable} using the given model.
   */
  @NonNull
  @CheckResult
  @Override
  public RequestBuilder<Drawable> load(@Nullable String string) {
    return asDrawable().load(string);
  }
```
在 load 方法中首先调用了 `asDrawable` 方法，该方法中又调用了 as 方法并传入 Drawable.class ，
```
/**
   * Attempts to load the resource using any registered {@link
   * com.bumptech.glide.load.ResourceDecoder}s that can decode the given resource class or any
   * subclass of the given resource class.
   *
   * @param resourceClass The resource to decode.
   * @return A new request builder for loading the given resource class.
   */
  @NonNull
  @CheckResult
  public <ResourceType> RequestBuilder<ResourceType> as(
      @NonNull Class<ResourceType> resourceClass) {
    return new RequestBuilder<>(glide, this, resourceClass, context);
  }
```
as 方法中创建了 RequestBuilder，所以最终调用了 RequestBuilder 的 load 方法：
```
public RequestBuilder<TranscodeType> load(@Nullable String string) {
    return loadGeneric(string);
  }
```
其中又调用了 `loadGeneric` ：
```
@NonNull
  private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
    this.model = model;
    isModelSet = true;
    return this;
  }
```
loadGeneric 方法中保存了入参并将 isModelSet 设置为 true，并返回自身。

**总结：load 方法主要用于创建一个 RequestBuilder 并保存入参。**

## into 方法
```
@NonNull
  public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
    Util.assertMainThread();
    ...
    return into(
        glideContext.buildImageViewTarget(view, transcodeClass),
        /*targetListener=*/ null,
        requestOptions,
        Executors.mainThreadExecutor());
  }
```
在 into 方法中对当前是否 UI 线程做了判断，因为只有 UI 线程才能更新 UI，即为 ImageView 设置图片。接着调用了 into 方法，并在参数中调用了 `glideContext.buildImageViewTarget` 方法。
```
@NonNull
  public <X> ViewTarget<ImageView, X> buildImageViewTarget(
      @NonNull ImageView imageView, @NonNull Class<X> transcodeClass) {
    return imageViewTargetFactory.buildTarget(imageView, transcodeClass);
  }
```
其中通过调用工厂类的 `buildTarget` 方法创建 ViewTarget：
```
public class ImageViewTargetFactory {
  @NonNull
  @SuppressWarnings("unchecked")
  public <Z> ViewTarget<ImageView, Z> buildTarget(
      @NonNull ImageView view, @NonNull Class<Z> clazz) {
    if (Bitmap.class.equals(clazz)) {
      return (ViewTarget<ImageView, Z>) new BitmapImageViewTarget(view);
    } else if (Drawable.class.isAssignableFrom(clazz)) {
      return (ViewTarget<ImageView, Z>) new DrawableImageViewTarget(view);
    } else {
      throw new IllegalArgumentException(
          "Unhandled class: " + clazz + ", try .as*(Class).transcode(ResourceTranscoder)");
    }
  }
}
```
根据传入的 class 类型创建不同的 ViewTarget。这个class参数其实基本上只有两种情况，如果你在使用Glide加载图片的时候调用了asBitmap()方法，那么这里就会构建出 BitmapImageViewTarget 对象，否则的话构建的都是 DrawableImageViewTarget 对象。再回到之前的 into 方法：
```
private <Y extends Target<TranscodeType>> Y into(
      @NonNull Y target,
      @Nullable RequestListener<TranscodeType> targetListener,
      BaseRequestOptions<?> options,
      Executor callbackExecutor) {
    ...
    Request request = buildRequest(target, targetListener, options, callbackExecutor);

    Request previous = target.getRequest();
    if (request.isEquivalentTo(previous)
        && !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
      // If the request is completed, beginning again will ensure the result is re-delivered,
      // triggering RequestListeners and Targets. If the request is failed, beginning again will
      // restart the request, giving it another chance to complete. If the request is already
      // running, we can let it continue running without interruption.
      if (!Preconditions.checkNotNull(previous).isRunning()) {
        // Use the previous request rather than the new one to allow for optimizations like skipping
        // setting placeholders, tracking and un-tracking Targets, and obtaining View dimensions
        // that are done in the individual Request.
        previous.begin();
      }
      return target;
    }

    requestManager.clear(target);
    target.setRequest(request);
    requestManager.track(target, request);

    return target;
  }
```
首先调用了 `buildRequest` 方法创建一个 request，在 `buildRequest` 中又调用了 `buildRequestRecursive` 方法：
```
private Request buildRequestRecursive(
      Object requestLock,
      Target<TranscodeType> target,
      @Nullable RequestListener<TranscodeType> targetListener,
      @Nullable RequestCoordinator parentCoordinator,
      TransitionOptions<?, ? super TranscodeType> transitionOptions,
      Priority priority,
      int overrideWidth,
      int overrideHeight,
      BaseRequestOptions<?> requestOptions,
      Executor callbackExecutor) {

    // Build the ErrorRequestCoordinator first if necessary so we can update parentCoordinator.
    ErrorRequestCoordinator errorRequestCoordinator = null;
    if (errorBuilder != null) {
      errorRequestCoordinator = new ErrorRequestCoordinator(requestLock, parentCoordinator);
      parentCoordinator = errorRequestCoordinator;
    }

    Request mainRequest =
        buildThumbnailRequestRecursive(
            requestLock,
            target,
            targetListener,
            parentCoordinator,
            transitionOptions,
            priority,
            overrideWidth,
            overrideHeight,
            requestOptions,
            callbackExecutor);

    if (errorRequestCoordinator == null) {
      return mainRequest;
    }
    ...
  }
```
在该方法中调用了 `buildThumbnailRequestRecursive` ：
```
private Request buildThumbnailRequestRecursive(
      Object requestLock,
      Target<TranscodeType> target,
      RequestListener<TranscodeType> targetListener,
      @Nullable RequestCoordinator parentCoordinator,
      TransitionOptions<?, ? super TranscodeType> transitionOptions,
      Priority priority,
      int overrideWidth,
      int overrideHeight,
      BaseRequestOptions<?> requestOptions,
      Executor callbackExecutor) {
      ...
      ThumbnailRequestCoordinator coordinator =
          new ThumbnailRequestCoordinator(requestLock, parentCoordinator);
      Request fullRequest =
          obtainRequest(
              requestLock,
              target,
              targetListener,
              requestOptions,
              coordinator,
              transitionOptions,
              priority,
              overrideWidth,
              overrideHeight,
              callbackExecutor);
      isThumbnailBuilt = true;
      // Recursively generate thumbnail requests.
      Request thumbRequest =
          thumbnailBuilder.buildRequestRecursive(
              requestLock,
              target,
              targetListener,
              coordinator,
              thumbTransitionOptions,
              thumbPriority,
              thumbOverrideWidth,
              thumbOverrideHeight,
              thumbnailBuilder,
              callbackExecutor);
      isThumbnailBuilt = false;
      coordinator.setRequests(fullRequest, thumbRequest);
      return coordinator;
    } else if (thumbSizeMultiplier != null) {
      // Base case: thumbnail multiplier generates a thumbnail request, but cannot recurse.
      ThumbnailRequestCoordinator coordinator =
          new ThumbnailRequestCoordinator(requestLock, parentCoordinator);
      Request fullRequest =
          obtainRequest(
              requestLock,
              target,
              targetListener,
              requestOptions,
              coordinator,
              transitionOptions,
              priority,
              overrideWidth,
              overrideHeight,
              callbackExecutor);
      BaseRequestOptions<?> thumbnailOptions =
          requestOptions.clone().sizeMultiplier(thumbSizeMultiplier);

      Request thumbnailRequest =
          obtainRequest(
              requestLock,
              target,
              targetListener,
              thumbnailOptions,
              coordinator,
              transitionOptions,
              getThumbnailPriority(priority),
              overrideWidth,
              overrideHeight,
              callbackExecutor);

      coordinator.setRequests(fullRequest, thumbnailRequest);
      return coordinator;
    } else {
      // Base case: no thumbnail.
      return obtainRequest(
          requestLock,
          target,
          targetListener,
          requestOptions,
          parentCoordinator,
          transitionOptions,
          priority,
          overrideWidth,
          overrideHeight,
          callbackExecutor);
    }
  }
```
可以看到最后都是调用了 `obtainRequest` 方法：
```
private Request obtainRequest(
      Object requestLock,
      Target<TranscodeType> target,
      RequestListener<TranscodeType> targetListener,
      BaseRequestOptions<?> requestOptions,
      RequestCoordinator requestCoordinator,
      TransitionOptions<?, ? super TranscodeType> transitionOptions,
      Priority priority,
      int overrideWidth,
      int overrideHeight,
      Executor callbackExecutor) {
    return SingleRequest.obtain(
        context,
        glideContext,
        requestLock,
        model,
        transcodeClass,
        requestOptions,
        overrideWidth,
        overrideHeight,
        priority,
        target,
        targetListener,
        requestListeners,
        requestCoordinator,
        glideContext.getEngine(),
        transitionOptions.getTransitionFactory(),
        callbackExecutor);
  }
```
在 `obtainRequest` 方法中调用了 `SingleRequest.obtain` ：
```
public static <R> SingleRequest<R> obtain(
      Context context,
      GlideContext glideContext,
      Object requestLock,
      Object model,
      Class<R> transcodeClass,
      BaseRequestOptions<?> requestOptions,
      int overrideWidth,
      int overrideHeight,
      Priority priority,
      Target<R> target,
      RequestListener<R> targetListener,
      @Nullable List<RequestListener<R>> requestListeners,
      RequestCoordinator requestCoordinator,
      Engine engine,
      TransitionFactory<? super R> animationFactory,
      Executor callbackExecutor) {
    return new SingleRequest<>(
        context,
        glideContext,
        requestLock,
        model,
        transcodeClass,
        requestOptions,
        overrideWidth,
        overrideHeight,
        priority,
        target,
        targetListener,
        requestListeners,
        requestCoordinator,
        engine,
        animationFactory,
        callbackExecutor);
  }
```
可以看到其中创建了一个 SingleRequest 实例。再回到 into 方法中，因此在 into 方法中通过调用 buildRequest 方法最终获取到了一个 SingleRequest。而后将 request 设置到 requestManager 并调用 RequestManager 的 track 方法：
```
synchronized void track(@NonNull Target<?> target, @NonNull Request request) {
    targetTracker.track(target);
    requestTracker.runRequest(request);
  }
```
TargetTracker 实现了 LifecycleListener，可以感知生命周期变化。接着调用了 RequestTracker 的 `runRequest`：
```
/** Starts tracking the given request. */
  public void runRequest(@NonNull Request request) {
    requests.add(request);
    if (!isPaused) {
      request.begin();
    } else {
      request.clear();
    ...
    }
}
```
在这个方法中对当前 Glide 状态进行了判断，如果不是 paused，则调用 request 的 begin 方法开始请求，否则取消 request。通过之前的分析，我们知道 request 是一个 SingleRequest 实例，begin 的具体实现在 SingleRequest 中：
```
@Override
  public void begin() {
    synchronized (requestLock) {
      assertNotCallingCallbacks();
      stateVerifier.throwIfRecycled();
      startTime = LogTime.getLogTime();
      if (model == null) {
        if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
          width = overrideWidth;
          height = overrideHeight;
        }
        ...
        onLoadFailed(new GlideException("Received null model"), logLevel);
        return;
      }

      if (status == Status.RUNNING) {
        throw new IllegalArgumentException("Cannot restart a running request");
      }

      // If we're restarted after we're complete (usually via something like a notifyDataSetChanged
      // that starts an identical request into the same Target or View), we can simply use the
      // resource and size we retrieved the last time around and skip obtaining a new size, starting
      // a new load etc. This does mean that users who want to restart a load because they expect
      // that the view size has changed will need to explicitly clear the View or Target before
      // starting the new load.
      if (status == Status.COMPLETE) {
        onResourceReady(resource, DataSource.MEMORY_CACHE);
        return;
      }

      // Restarts for requests that are neither complete nor running can be treated as new requests
      // and can run again from the beginning.

      status = Status.WAITING_FOR_SIZE;
      if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
        onSizeReady(overrideWidth, overrideHeight);
      } else {
        target.getSize(this);
      }

      if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
          && canNotifyStatusChanged()) {
        target.onLoadStarted(getPlaceholderDrawable());
      }
      if (IS_VERBOSE_LOGGABLE) {
        logV("finished run method in " + LogTime.getElapsedMillis(startTime));
      }
    }
  }
```
在 begin 方法中对 model，也就是调用 load 方法传入的参数进行空判断，如果为空，则调用 onLoadFailed 方法，在该方法中会将状态设置为 failed，如果设置了 RequestListener，则回调 RequestListener 的 onLoadFailed，并调用 setErrorPlaceholder 方法设置失败占位图。
如果当前状态为 `Status.COMPLETE` 则直接调用 `onResourceReady`，最终调用了 Target 的 `onResourceReady` 方法，通过前面的分析，此处 Target 是 DrawableImageViewTarget，DrawableImageViewTarget 继承自 ImageViewTarget，方法在 ImageViewTarget 中实现：
```
@Override
  public void onResourceReady(@NonNull Z resource, @Nullable Transition<? super Z> transition) {
    if (transition == null || !transition.transition(resource, this)) {
      setResourceInternal(resource);
    } else {
      maybeUpdateAnimatable(resource);
    }
  }
```
最终都会调用 setResource 方法：
```
@Override
  protected void setResource(@Nullable Drawable resource) {
    view.setImageDrawable(resource);
  }
```
至此，则将加载的 drawable 设置到了 view 上。
接着看 begin 方法，如果在使用 Glide 时通过 override 方法指定了宽高，则调用 `onSizeReady` 方法，否则调用 Target 的 getSize 方法，该方法通过 Target 获取到宽高后也会调用 `onSizeReady` 方法：
```
/** A callback method that should never be invoked directly. */
  @Override
  public void onSizeReady(int width, int height) {
    stateVerifier.throwIfRecycled();
    synchronized (requestLock) {
      ...
      status = Status.RUNNING;
    ...
      loadStatus =
          engine.load(
              glideContext,
              model,
              requestOptions.getSignature(),
              this.width,
              this.height,
              requestOptions.getResourceClass(),
              transcodeClass,
              priority,
              requestOptions.getDiskCacheStrategy(),
              requestOptions.getTransformations(),
              requestOptions.isTransformationRequired(),
              requestOptions.isScaleOnlyOrNoTransform(),
              requestOptions.getOptions(),
              requestOptions.isMemoryCacheable(),
              requestOptions.getUseUnlimitedSourceGeneratorsPool(),
              requestOptions.getUseAnimationPool(),
              requestOptions.getOnlyRetrieveFromCache(),
              this,
              callbackExecutor);
        ...
    }
  }
```
将状态设置为 RUNNING，并调用了 Engine 的 load 方法。
```
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
    long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;

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
      memoryResource = loadFromMemory(key, isMemoryCacheable, startTime);

      if (memoryResource == null) {
        return waitForExistingOrStartNewJob(
            glideContext,
            model,
            signature,
            width,
            height,
            resourceClass,
            transcodeClass,
            priority,
            diskCacheStrategy,
            transformations,
            isTransformationRequired,
            isScaleOnlyOrNoTransform,
            options,
            isMemoryCacheable,
            useUnlimitedSourceExecutorPool,
            useAnimationPool,
            onlyRetrieveFromCache,
            cb,
            callbackExecutor,
            key,
            startTime);
      }
    }

    // Avoid calling back while holding the engine lock, doing so makes it easier for callers to
    // deadlock.
    cb.onResourceReady(memoryResource, DataSource.MEMORY_CACHE);
    return null;
  }
```
Engine 的 load 方法中首先通过参数构建了 key，它是一个 EngineKey，用作内存缓存的 key，然后尝试通过 key 从内存缓存中读取缓存，如果读取到了，则直接用这份缓存数据调用 SingleRequest 的 `onResourceReady` 方法，则会回到上文中的状态为 COMPLETE 的流程。否则调用 `waitForExistingOrStartNewJob` 方法创建新任务：
```
private <R> LoadStatus waitForExistingOrStartNewJob(
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
      Executor callbackExecutor,
      EngineKey key,
      long startTime) {

    EngineJob<?> current = jobs.get(key, onlyRetrieveFromCache);
    if (current != null) {
      current.addCallback(cb, callbackExecutor);
      ...
      return new LoadStatus(cb, current);
    }

    EngineJob<R> engineJob =
        engineJobFactory.build(
            key,
            isMemoryCacheable,
            useUnlimitedSourceExecutorPool,
            useAnimationPool,
            onlyRetrieveFromCache);

    DecodeJob<R> decodeJob =
        decodeJobFactory.build(
            glideContext,
            model,
            key,
            signature,
            width,
            height,
            resourceClass,
            transcodeClass,
            priority,
            diskCacheStrategy,
            transformations,
            isTransformationRequired,
            isScaleOnlyOrNoTransform,
            onlyRetrieveFromCache,
            options,
            engineJob);

    jobs.put(key, engineJob);

    engineJob.addCallback(cb, callbackExecutor);
    engineJob.start(decodeJob);
    ...
    return new LoadStatus(cb, engineJob);
  }
```
如果任务已经存在，则执行回调方法。否则创建新任务，并为任务添加回调，同时也创建了一个 DecodeJob，用于解析图片。最后调用了 EnginJob 的 start 方法，并将 DecodeJob 作为参数传入：
```
public synchronized void start(DecodeJob<R> decodeJob) {
    this.decodeJob = decodeJob;
    GlideExecutor executor =
        decodeJob.willDecodeFromCache() ? diskCacheExecutor : getActiveSourceExecutor();
    executor.execute(decodeJob);
}
```
start 方法中调用了 GlideExecutor 的 execute 方法：
```
@Override
  public void execute(@NonNull Runnable command) {
    delegate.execute(command);
  }
```
开启了一个新线程加载资源，从主线程进入子线程，最终执行的是 DecodeJob 的 run 方法：
```
  @Override
  public void run() {
    // This should be much more fine grained, but since Java's thread pool implementation silently
    // swallows all otherwise fatal exceptions, this will at least make it obvious to developers
    // that something is failing.
    GlideTrace.beginSectionFormat("DecodeJob#run(model=%s)", model);
    // Methods in the try statement can invalidate currentFetcher, so set a local variable here to
    // ensure that the fetcher is cleaned up either way.
    DataFetcher<?> localFetcher = currentFetcher;
    try {
      if (isCancelled) {
        notifyFailed();
        return;
      }
      runWrapped();
    } catch (CallbackException e) {
      // If a callback not controlled by Glide throws an exception, we should avoid the Glide
      // specific debug logic below.
      throw e;
    } catch (Throwable t) {
      // Catch Throwable and not Exception to handle OOMs. Throwables are swallowed by our
      // usage of .submit() in GlideExecutor so we're not silently hiding crashes by doing this. We
      // are however ensuring that our callbacks are always notified when a load fails. Without this
      // notification, uncaught throwables never notify the corresponding callbacks, which can cause
      // loads to silently hang forever, a case that's especially bad for users using Futures on
      // background threads.
      ...
      // When we're encoding we've already notified our callback and it isn't safe to do so again.
      if (stage != Stage.ENCODE) {
        throwables.add(t);
        notifyFailed();
      }
      if (!isCancelled) {
        throw t;
      }
      throw t;
    } finally {
      // Keeping track of the fetcher here and calling cleanup is excessively paranoid, we call
      // close in all cases anyway.
      if (localFetcher != null) {
        localFetcher.cleanup();
      }
      GlideTrace.endSection();
    }
  }
```
调用了 `runWrapped` 方法：
```
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
```
首先会进入 INITIALIZE，不考虑缓存，即关闭了缓存的情况下  getNextGenerator 返回 SourceGenerator，接着调用了 runGenerators：
```
private void runGenerators() {
    currentThread = Thread.currentThread();
    startFetchTime = LogTime.getLogTime();
    boolean isStarted = false;
    while (!isCancelled
        && currentGenerator != null
        && !(isStarted = currentGenerator.startNext())) {
      ...
  }
```
在 while 循环中执行了 currentGenerator.startNext()，如下：
```
@Override
  public boolean startNext() {
    ...
    while (!started && hasNextModelLoader()) {
      loadData = helper.getLoadData().get(loadDataListIndex++);
      if (loadData != null
          && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())
              || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
        started = true;
        startNextLoad(loadData);
      }
    }
    return started;
  }
```
在这里调用 startNextLoad：
```
private void startNextLoad(final LoadData<?> toStart) {
    loadData.fetcher.loadData(
        helper.getPriority(),
        new DataCallback<Object>() {
          @Override
          public void onDataReady(@Nullable Object data) {
            if (isCurrentRequest(toStart)) {
              onDataReadyInternal(toStart, data);
            }
          }

          @Override
          public void onLoadFailed(@NonNull Exception e) {
            if (isCurrentRequest(toStart)) {
              onLoadFailedInternal(toStart, e);
            }
          }
        });
}
```
执行了 loadData.fetcher.loadData ，其中 fetcher 是 HttpUrlFetcher。
```
@Override
  public void loadData(
      @NonNull Priority priority, @NonNull DataCallback<? super InputStream> callback) {
    long startTime = LogTime.getLogTime();
    try {
      InputStream result = loadDataWithRedirects(glideUrl.toURL(), 0, null, glideUrl.getHeaders());
      callback.onDataReady(result);
    } catch (IOException e) {
      ...
      callback.onLoadFailed(e);
    } finally {
      ...
    }
  }
```
这里通过调用 loadDataWithRedirects 获取输入流：
```
private InputStream loadDataWithRedirects(
      URL url, int redirects, URL lastUrl, Map<String, String> headers) throws IOException {
    if (redirects >= MAXIMUM_REDIRECTS) {
      throw new HttpException("Too many (> " + MAXIMUM_REDIRECTS + ") redirects!");
    } else {
      // Comparing the URLs using .equals performs additional network I/O and is generally broken.
      // See http://michaelscharf.blogspot.com/2006/11/javaneturlequals-and-hashcode-make.html.
      try {
        if (lastUrl != null && url.toURI().equals(lastUrl.toURI())) {
          throw new HttpException("In re-direct loop");
        }
      } catch (URISyntaxException e) {
        // Do nothing, this is best effort.
      }
    }

    urlConnection = connectionFactory.build(url);
    for (Map.Entry<String, String> headerEntry : headers.entrySet()) {
      urlConnection.addRequestProperty(headerEntry.getKey(), headerEntry.getValue());
    }
    urlConnection.setConnectTimeout(timeout);
    urlConnection.setReadTimeout(timeout);
    urlConnection.setUseCaches(false);
    urlConnection.setDoInput(true);

    // Stop the urlConnection instance of HttpUrlConnection from following redirects so that
    // redirects will be handled by recursive calls to this method, loadDataWithRedirects.
    urlConnection.setInstanceFollowRedirects(false);

    // Connect explicitly to avoid errors in decoders if connection fails.
    urlConnection.connect();
    // Set the stream so that it's closed in cleanup to avoid resource leaks. See #2352.
    stream = urlConnection.getInputStream();
    if (isCancelled) {
      return null;
    }
    final int statusCode = urlConnection.getResponseCode();
    if (isHttpOk(statusCode)) {
      return getStreamForSuccessfulRequest(urlConnection);
    } else if (isHttpRedirect(statusCode)) {
      String redirectUrlString = urlConnection.getHeaderField("Location");
      if (TextUtils.isEmpty(redirectUrlString)) {
        throw new HttpException("Received empty or null redirect url");
      }
      URL redirectUrl = new URL(url, redirectUrlString);
      // Closing the stream specifically is required to avoid leaking ResponseBodys in addition
      // to disconnecting the url connection below. See #2352.
      cleanup();
      return loadDataWithRedirects(redirectUrl, redirects + 1, url, headers);
    } else if (statusCode == INVALID_STATUS_CODE) {
      throw new HttpException(statusCode);
    } else {
      throw new HttpException(urlConnection.getResponseMessage(), statusCode);
    }
  }
```
通过 HttpUrlConnection 完成图片网络请求，并返回输入流，完成请求后回调 callback 的 onDataReady，进而回调 onDataReadyInernal：
```
@Synthetic
  void onDataReadyInternal(LoadData<?> loadData, Object data) {
    DiskCacheStrategy diskCacheStrategy = helper.getDiskCacheStrategy();
    if (data != null && diskCacheStrategy.isDataCacheable(loadData.fetcher.getDataSource())) {
      dataToCache = data;
      // We might be being called back on someone else's thread. Before doing anything, we should
      // reschedule to get back onto Glide's thread.
      cb.reschedule();
    } else {
      cb.onDataFetcherReady(
          loadData.sourceKey,
          data,
          loadData.fetcher,
          loadData.fetcher.getDataSource(),
          originalKey);
    }
  }
```
如果没有配置缓存则执行 cb.onDataFetcherReady 方法，这里的 cb 是 FetcherReadyCallback，DecodeJob 实现了该接口：
```
@Override
  public void onDataFetcherReady(
      Key sourceKey, Object data, DataFetcher<?> fetcher, DataSource dataSource, Key attemptedKey) {
    this.currentSourceKey = sourceKey;
    this.currentData = data;
    this.currentFetcher = fetcher;
    this.currentDataSource = dataSource;
    this.currentAttemptingKey = attemptedKey;
    if (Thread.currentThread() != currentThread) {
      runReason = RunReason.DECODE_DATA;
      callback.reschedule(this);
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
在这里执行了 decodeFromRetrieveData 方法：
```
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
调用了 notifyEncodeAndRelease：
```
private void notifyEncodeAndRelease(Resource<R> resource, DataSource dataSource) {
    ...
    notifyComplete(result, dataSource);
    ...
  }
```
在这里调用了 notifyComplete 方法：
```
private void notifyComplete(Resource<R> resource, DataSource dataSource) {
    setNotifiedOrThrow();
    callback.onResourceReady(resource, dataSource);
  }
```
callback 的实现类是 EngineJob：
```
@Override
  public void onResourceReady(Resource<R> resource, DataSource dataSource) {
    synchronized (this) {
      this.resource = resource;
      this.dataSource = dataSource;
    }
    notifyCallbacksOfResult();
  }
```
再看 notifyCallbacksOfResult：
```
@Synthetic
  void notifyCallbacksOfResult() {
    ResourceCallbacksAndExecutors copy;
    Key localKey;
    EngineResource<?> localResource;
    ...
    engineJobListener.onEngineJobComplete(this, localKey, localResource);

    for (final ResourceCallbackAndExecutor entry : copy) {
      entry.executor.execute(new CallResourceReady(entry.cb));
    }
    decrementPendingCallbacks();
  }
```
在这里通知 Engine 任务完成，再执行了 CallResourceReady，它实现了 Runnable，看它的 run 方法：
```
@Override
    public void run() {
      // Make sure we always acquire the request lock, then the EngineJob lock to avoid deadlock
      // (b/136032534).
      synchronized (cb.getLock()) {
        synchronized (EngineJob.this) {
          if (cbs.contains(cb)) {
            // Acquire for this particular callback.
            engineResource.acquire();
            callCallbackOnResourceReady(cb);
            removeCallback(cb);
          }
          decrementPendingCallbacks();
        }
      }
    }
```
run 方法中执行了 callCallbackOnResourceReady：
```
void callCallbackOnResourceReady(ResourceCallback cb) {
    try {
      // This is overly broad, some Glide code is actually called here, but it's much
      // simpler to encapsulate here than to do so at the actual call point in the
      // Request implementation.
      cb.onResourceReady(engineResource, dataSource);
    } catch (Throwable t) {
      throw new CallbackException(t);
    }
  }
```
这里的 cb 是 SingleRequest，回调到 onResourceReady 后会将状态设置为 COMPLETE，然后回调 Target 的 onResourceReady，之后流程如上文 status == COMPLETE 一样。
**总结：load 方法主要通过构建一个 SingleRequest 任务，通过 GlideExector 创建子线程使用 HttpUrlConnection 完成图片的网络请求，最终将图片设置到 ImageView 上显示。**

