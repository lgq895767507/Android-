# **理解 ActivityManagerService**
## AMS 家族（基于 Android 8.0）
![](AMS 家族.png)
ActivityManager 的 getService() 方法调用了 IActivityManagerSingleton 的 get 方法，获取名为 “activity” 的 IBinder 对象，也就是 AMS 的引用，然后将此 IBinder 对象转换为 IActivityManager 类型的对象，采用了 AIDL，而服务端的 AMS 实现了 IActivityManager.Stub 类，由此实现了进程间通信。
## AMS 的启动过程
AMS 是在 SystemServer 进程中启动的。再 SystemServer 的 main 方法中调用了 run 方法，run 方法中的 startBootstrapServices 方法中启动了 AMS 服务。
![](AMS 的启动过程.png)
