# Android
## Activity

[Android全面解析之Activity生命周期](https://juejin.cn/post/6892745298209308680)

[Activity 启动流程](https://juejin.cn/post/6990297933790838798)

[Activity 启动模式](https://juejin.cn/post/6987002152191426568)

###  从桌面点击一个图标之后，到界面显示，这个过程发生了什么？ 

手机桌面其实是一个叫做 Launcher 的 App，当我们点击桌面上的应用图标后，就会调用到 `Launcher.startActivitySafely()` 函数去启动目标应用的 LaunchActivity

之后 Launcher 进程就会请求 AMS 去启动目标 Activity， AMS 进程会进行一系列的验证工作，如判断目标 Activity 实例是否已经存在、启动模式是什么、有没有在 AndroidManifest.xml 文件中注册等等

这些工作结束后，AMS会发现此时 App 进程还没有被创建，就会请求 Zygote 进程去创建 App 进程

当 App 进程创建结束后，就会进行初始化，并通知 AMS；AMS 收到来自 App 进程的通知后，就会将创建 Activity 的一系列操作封装成事务 (ClientTransaction) 发送给 App 进程

App 进程收到来自 AMS 的事务后，根据事务创建目标 Activity，并回调目标 Activity 的生命周期

### 从 A Activity 启动 B Activity 的生命周期变化？

* 当 B Activity 的 launchMode 为 standard 时，生命周期变化是：

   `A.onPause()` --> `B.onCreate()` --> `B.onStart()` --> `B.onResume()` --> `A.onStop()`

* 当 B Activity 的 launchMode 为 singleTop 时

  * 如果发生了复用，就说明是 **自己启动自己**，生命周期的变化是：

     `A.onPause()` -> `A.onNewIntent()` -> `A.onResume()`

  * 如果没有发生复用，生命周期的变化是：

     `A.onPause()` --> `B.onCreate()` --> `B.onStart()` --> `B.onResume()` --> `A.onStop()`

* 当 B Activity 的 launchMode 为 singleTask / singleInstance 时

  * 如果发生了复用，生命周期的变化是：

     `A.onPause()` --> `B.onRestart()` --> `B.onStart()` --> `B.onNewIntent()` --> `B.onResume()` --> `A.onStop()`

  * 如果没有发生复用，生命周期的变化是：

     `A.onPause()` --> `B.onCreate()` --> `B.onStart()` --> `B.onResume()` --> `A.onStop()`

### 从 B Activity 返回 A Activity 的生命周期变化？

`B.onPause()` --> `A.onRestart()` --> `A.onStart()` --> `A.onResume()` --> `B.onStop()` --> `B.onDestroy()`

### 能说说 Activity 的四种启动模式吗？常见的应用场景是什么？

常见的启动模式分别是  `android:launchMode`  的四种属性值：

- 标准模式 (standard)
- 栈顶复用模式 (singleTop)
- 全局单例模式 (singleTask)
- 单实例模式 (singleInstance)

standard 就是默认模式，每次启动都会将新 Activity 实例添加入当前 Task；

singleTop 是栈顶复用模式，如果当前 Task 的栈顶就是目标 Activity，就复用位于栈顶的实例，回调该实例的 `onNewIntent()` 函数；

singleTask 是全局单例模式，如果系统中已经有了目标 Activity 的实例，就会将该实例所在的 Task 设置为前台 Task，并复用该实例，回调该实例的 `onNewIntent` 函数；如果该实例的上方有别的 Activity 实例，就将这些 Activity 实例移除；

singleInstance 是单实例模式，如果系统中已经有了目标 Activity 的实例，就会将该实例所在的 Task 设置为前台 Task，并复用该实例，回调该实例的 `onNewIntent()` 函数；同时 Task 内只允许存在目标 Activity 的单一实例

应用场景：

| LaunchMode     |                           应用场景                           |
| -------------- | :----------------------------------------------------------: |
| standard       |                       普通的 Activity                        |
| singleTop      | 适合可能重复使用的 Activity。例如：接收 Notification 启动的内容显示页面、耗时操作的返回页面、登录页面等 |
| singleTask     | 适合能作为程序入口的 Activity。例如：WebView 页面、扫一扫页面、确认订单界面、付款界面等 |
| singleInstance | 适合需要与程序分离开的 Activity。例如：闹铃的响铃界面、来电页面、锁屏页等 |

### 弹出 Dialog 对当前显示的 Activity 的生命周期有什么影响？

会回调当前 Activity 的 `onPause()` 函数，并不会回调 `onStop()` 函数

因为 `onStop` 函数只有当 Activity **完全不可见** 的时候才会被回调，通常只有当 Activity 被移至后台才会被调用

弹出 Dialog 只是让当前显示的 Activity 失去焦点并没有让其完全不可见，所以不会回调 Activity 的 `onStop()` 函数

### onActivityResult 什么时候会被回调？

`onActivityResult()` 函数不属于 Activity 的生命周期，所以它会优先于生命周期函数而执行

从 B Activity 返回 A Activity，生命周期变化是：

`B.onPause()` --> `A.onActivityResult()` --> `A.onRestart()` --> `A.onStart()` --> `A.onResume()`

### onCreate 函数里写死循环会导致 ANR 吗？

ANR 产生的原因：

- KeyDispatchTimeout：输入事件 (触摸事件等) 在 5s 内没有被处理完成
- BroadcastTimeout： 
  - 前台 Broadcast：onReceiver 函数在 10s 内没有执行结束
  - 后台 Broadcast：onReceiver 函数在 60s 内没有执行结束
- ServiceTimeout： 
  - 前台 Service：onCreate、onStart、onBind 等生命周期函数在 20s 内没有执行结束
  - 后台 Service：onCreate、onStart、onBind 等生命周期函数在 200s 内没有执行结束
- ContentProviderTimeout：ContentProvider 在 10s 内没有处理完成当前事务

onCreate 函数被阻塞并不在触发 ANR 的场景里面，所以并不会直接触发 ANR

只不过死循环阻塞了主线程，如果此时用户触摸屏幕想触发控件的点击事件，就会因为点击事件没有在 5s 内被处理而触发 ANR

## Broadcast

### BroadcastReciver 静态注册与动态注册的区别？

* BroadcastReciver 静态注册通过在 AndroidManifest 文件中声明 receiver 标签进行注册

  当 APP 应用安装时，PMS (PackageManagerService) 会解析 APK 包中的 AndroidManifest 文件，实例化广播类，并注册到广播系统中

  静态注册的 BroadcastReciver 会常驻后台运行，不会因为 APP 进程的结束而停止运行，因此会增加手机的耗电量和运行时内存

  为了优化 Android 系统运行，Goolge 规定在 Android 7.0 以上的系统中不能静态注册 BroadcastReciver

* BroadcastReciver 动态注册通过调用 `registerReceiver()` 函数进行注册，通过调用 `unregisterReceiver()` 函数进行注销

  动态注册的注册和注销需要开发者自行调用，灵活可控，不同于静态注册的注册和注销由系统管理

  同时动态注册的 BroadcastReciver 比静态注册的 BroadcastReciver 优先级高

### 有序 Broadcast 和无序 Broadcast 的区别？

* 无序 Broadcast 通过 `sendBroadcast()` 函数进行发送，并通过 intent 传递数据

  顾名思义，无序广播的发送顺序是无序的

  无序广播不可以被拦截，不可以被终止，不可以被修改，接收无优先级之分，BroadcastReciver 之间不能通过无序广播传递数据

* 有序 Broadcast 通过 `sendOrderedBroadcast()` 函数进行发送，该函数有几个比较重要的参数：

  * receiverPermission：指定接收权限

    如果设置了 receiverPermission，BroadcastReciver 需要声明该权限才能接收到广播

  * resultReceiver：指定最终的 BroadcastReciver

    通过该参数指定的 BroadcastReciver，无论该有序广播是否被拦截，最终都会接收到该有序广播

  * initialData：指定该有序广播初始携带的字符串数据

    当 BroadcastReciver 接收到有序广播后，可通过 `getResultData()` 函数获取该数据，可通过 `setResultData()` 函数修改该数据

  * initialExtras：指定该有序广播初始携带的 Bundle 数据

    当 BroadcastReciver 接收到有序广播后，可通过 `getResultExtras()` 函数获取该 Bundle 对象，可通过 `setResultExtras()` 函数传入一个修改过的 Bundle 对象

  有序广播的发送顺序由 BroadcastReciver 的 android:priority 属性确定，android:priority 是一个 int 型整数，值越大，其所对应的 BroadcastReciver，越先接收到广播

  android:priority 相同的情况下，如果 BroadcastReciver 是通过静态注册的，则接收到广播的顺序不确定；如果 BroadcastReciver 是动态注册的，则先注册的将先接收到广播

  有序广播可以被拦截，当 BroadcastReciver 接收到有序广播后，可通过 `abortBroadcast()` 函数拦截该有序广播

  如果有序广播被较高优先级的 BroadcastReciver 拦截，则较低优先级的 BroadcastReciver 无法接收到该有序广播，但通过 resultReceiver 参数指定的 BroadcastReciver 仍可以接收到该有序广播

  有序广播也可通过 intent.putExtra 传递数据，但通过 intent.putExtra 传递的数据无法中途被修改

### 全局 Broadcast 和本地 Broadcast 的区别？

* 本地 Broadcast 的发送和接收都在同一 APP 应用中，不影响其他 APP 应用也不受其他 APP 应用影响

  本地广播只能被动态注册的 BroadcastReciver 接收

  主要通过 LocalBroadcastManager 进行使用

* 全局 Broadcast 的发送和接收可以在不同的 APP 应用中

  BroadcastReciver 可以接收其他 APP 应用 / 系统发送的全局广播

  也可以发送全局广播让其他 APP 应用中的 BroadcastReciver 接收

  无论是通过静态注册还是动态注册的 BroadcastReciver 都可以接收全局广播

### Broadcast 发送和接收过程？

Broadcast 的发送主要由 AMS 管理，AMS 会将所要发送的 Broadcast 信息与所有已注册的 BroadcastReciver 信息 (权限 + IntentFilter) 进行比对，将广播发送给满足条件的 BroadcastReciver

由于 BroadcastReciver 运行在其他进程中，AMS 需要 AIDL 调用 BroadcastReciver 运行所在进程，传递 intent 等信息，并最终调用 BroadcastReceiver 的 `onReceive()` 函数

以无序广播的发送流程为例，总体流程如下图：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/618498747a3149869381196be0d093ea~tplv-k3u1fbpfcp-zoom-1.image)

图中默认 BroadcastReciver 是通过动态注册的，因此 BroadcastReciver 运行在 APP 进程中 (动态注册的 BroadcastReciver 都是运行在 APP 进程中)

### BroadcastReceiver 有哪几种区别？分别运行在哪个进程中？onReceive 运行在哪个线程中？

BroadcastReceiver 分为静态注册和动态注册

静态注册的 BroadcastReceiver 可以运行在独立的进程中，动态注册的 BroadcastReceiver 只能运行在 APP 进程中

`onReceive()` 函数回调运行在主线程中

### BroadcastReceiver 没有收到 Broadcast 的原因？超过五秒后收到的原因？

BroadcastReceiver 没有收到 Broadcast 的原因可能有：

* BroadcastReceiver 的 IntentFilter 无法匹配
* BroadcastReceiver 没有添加对应的权限
* APP 没有声明对应的权限
* 如果是没有收到系统的 Broadcast，可能需要在 IntentFilter 中添加设置 (addDataScheme)

BroadcastReceiver 超过五秒后收到 Broadcast 的原因可能有：

* 在 Broadcast 发送流程中，AMS 会通过 Handler (BroadcastHandler) 从 Binder 线程切换到主线程执行

  由于 AMS 运行在 system_server 进程中，此进程中还运行着其他系统服务 (例如 WMS、PMS 等)，就有可能出现 system_server 进程的主线程 Looper 出现阻塞，导致广播发送延迟

* 在 Broadcast 发送流程中，AMS 会通过 AIDL 调用到 BroadcastReceiver 运行所在进程

  当出现手机系统资源紧张时 (例如内存不足)，就有可能导致 AIDL 调用无法执行 (内存不足，无法进行 mmap 内存映射)，导致广播发送延迟

### EventBus 和 Broadcast 各自的优劣？

|              Broadcast               |                       EventBus                        |
| :----------------------------------: | :---------------------------------------------------: |
|    可以在不同进程间发送和接收消息    |            只能在同一进程中发送和接收消息             |
|             属于异步调用             |      同一线程属于同步调用，不同线程属于异步调用       |
| 依赖于 Context，使其可以调用系统函数 | 不依赖于 Context，使用时无需关注 Context 的注入与传递 |

由于 Broadcast 机制使用了 Binder 进行跨进程通信，会消耗一定的系统资源，是一个重量级的订阅机制，因此 Broadcast 主要用于监听系统状态变化和 IPC

EventBus 相较于 Broadcast 更轻量，由于 EventBus 不支持多进程，因此 EventBus 主要用于多线程间的交互

### Broadcast 可以跨进程么？如果可以，是通过什么实现的？

可以，BroadcastReceiver 可以运行在独立的进程中

在 Broadcast 发送流程中，AMS 会通过 AIDL 调用到 BroadcastReceiver 运行所在进程

AIDL 的底层实现原理是 Binder 机制

## Service

### Activity 和 Service 有什么区别 (Service 存在的意义是什么) ？

基于 C / S 架构思维，Activity 就是客户端 (Client)，负责展示界面、与用户交互

Service 就是服务端，负责接收 Activity 的指令，在后台运行代码，并返回结果给 Activity

Service 可以理解为是无界面关联纯逻辑的 Activity

[关于android编程中service和activity的区别](https://blog.csdn.net/foreverhuylee/article/details/20372055)

### startService 与 bindService 的区别？生命周期？应用场景？

* 通过 startService 启动的 Service 生命周期为：

  `onCreate()` --> `onStartCommand()` --> `onDestory()`

  多次调用 startService 时，`onCreate()` 函数不重复执行，但 `onStartCommand()` 函数会多次执行

  通过 startService 启动的 Service 会独立在后台运行，不受 Activity 的影响，只有当 stopService 被调用 / 被系统回收时 Service 才会停止运行

  当 Service 独立在后台运行时，会提高 APP 进程的优先级，导致正常退出应用程序可能无法销毁 APP 进程 (强行杀死进程不算正常退出应用程序)

  当应用程序正常退出后，Service 在后台的运行时间一般不超过 80 s

* 通过 bindService 启动的 Service 生命周期为：

  `onCreate()` --> `onBind()` --> `onUnbind()` --> `onDestory()`

  多次调用 bindService 时，`onCreate()` 函数和 `onBind()` 函数不重复执行，但 `ServiceConnection.onServiceConnected()` 函数会多次执行

  通过 bindService 启动的 Service 生命周期由开发者控制，一般跟随 Activity 生命周期；当 Service 没有绑定时，系统会回收该 Service

  Activity 可以通过 ServiceConnection 获取到 Service 的 Binder 通信接口 (onBind 的返回值)，通过此接口 Activity 可以调用定义在 Service 内部的函数，从而实现 Activity 与 Service 间的交互

  通过 bindService 启动 Service 时，如果目标 Service 已在运行中且与 APP 进程进行过绑定，则不会执行 `onBind()` 函数，而是执行 `onReBind()` 函数 (前提：前一次 Service 解绑时 `onUnbind()` 函数返回 true)

可以同时通过 startService 和 bindService 启动同一个 Service，无先后之分，目标 Service 的 `onCreate()` 函数只会执行一次

关闭该 Service 时需要调用 stopService 和 unbindService，也无先后之分，目标 Service 的 `onDestroy()` 函数也只会执行一次

但如果只调用 stopService / unbindService 其中一个来关闭 Service，不论是调用哪个，Service 的 `onDestroy()` 函数都不会被执行，Service 也不会被关闭

如果想要在后台长期进行某项任务，那么可以使用 startService

如果想要 Service 伴随 Activity 生命周期，那么可以使用 bindService

如果想要在后台长期进行某项任务，且这个过程中需要与调用者 (Activity) 进行交互，那么可以两者同时使用，或者使用 startService + BoardCast / EventBus 等方式

### 多个 Activity 绑定同一个 Service 和单个 Activity 绑定一个 Service 有什么差别？

当 Activity 与 Service 一对一绑定时，在 Activity 中调用 unbindService 解绑该 Service，该 Service 会被关闭，并回调 `onUnbind()` 函数和 `onDestroy()` 函数

当多个 Activity 绑定同一个 Service 时，只有当绑定在该 Service 上的所有 Activity 都调用 unbindService 解绑，该 Service 才能被关闭，并回调 `onUnbind()` 函数和 `onDestroy()` 函数

### Activity 与 Service 之间如何通信？

Activity 可以通过 bindService 启动 Service

Service 启动成功后，Activity 可以通过 ServiceConnection 获取到 Service 的 Binder 通信接口 (onBind 的返回值)

通过此接口 Activity 可以调用定义在 Service 内部的函数，从而实现 Activity 与 Service 间的交互

### Service 与 IntentService 的区别？

Service 是运行在主线程中的，因此耗时操作 (例如访问网络、文件等) 是不能直接在 Service 中执行的

IntentService 继承自 Service，为了弥补普通 Service 无法执行耗时操作的缺陷，IntentService 在实现时使用了 HandlerThread 来执行 Activity 所传入的 intent

HandlerThread 本质上是一个线程，其在运行时初始化了 Looper，使得开发者可以通过 Handler 去操控 HandlerThread 执行耗时任务

当多次调用 startService 时，Service 的 `onStartCommand()` 函数会多次执行

因此 IntentService 在实现 `onStartCommand()` 函数时，将每次 Activity 所传入的 intent 封装成 message，并把 message 交给 Handler 执行

当 message 被 HandlerThread 中的 Looper 取出执行时，就会回调 IntentService 的 `onHandleIntent()` 函数，将 intent 交给开发者处理

因此 IntentService 可以按顺序依次执行多个耗时任务 (通过 startService 传入 intent)

### Service 和 Thread 都可以用来执行后台任务，为什么选 Serice 而不选 Thread (Service 与 Thread 的区别) ？

Service 是用于执行后台操作的，Thread 是用于执行耗时操作的，但后台操作并不完全等于耗时操作

在 Android 中，后台操作是指不依赖于 UI 界面的操作；耗时操作是后台操作的一种类型，由于其会阻塞线程所以需要通过子线程来执行，但不是所有的后台操作都是耗时操作

但在实际业务中，绝大多数的后台操作都是耗时操作，这就意味着通常 Service 需要通过子线程来执行后台操作

既然在 Service 里也要创建子线程，那为什么不直接在 Activity 里创建呢？

这是因为 Activity 很难对 Thread 进行控制，当 Activity 被销毁后，就没有其它的任何方法可以再重新获取到之前创建的子线程实例

并且在一个 Activity 中创建的子线程，另一个 Activity 无法对其进行操作

但 Service 就不同了，所有的 Activity 都可以与 Service 进行绑定 (bindService)，然后 Activity 就可以通过 Service 的 Binder 通信接口调用定义在 Service 内部的函数

即使 Activity 被销毁了，只要重新与 Service 进行绑定，就又能够获取到 Service 的 Binder 通信接口

因此，使用 Service 来执行后台操作，就完全不需要担心无法对后台操作进行控制的情况

## ContentProvider

### Android 系统为什么会设计出 ContentProvider？

* **隐藏数据的存储方式，对外提供统一的数据访问接口**

* **提供一种 IPC 方式**

  由系统来管理 ContentProvider 的创建、生命周期及访问的线程分配，简化我们实现 IPC 的步骤

  我们只需要通过 ContentResolver 访问 ContentProvider 所提示的数据接口，而不需要担心它所在进程是否启动

* **更好的数据访问权限管理**

  ContentProvider 可以对开发的数据进行权限设置，不同的 Uri 可以对应不同的权限，只有符合权限要求的组件才能访问到 ContentProvider 的具体操作

### ContentProvider 的启动流程？

当 ContentProvider 运行所在进程与 APP 进程相同时，ContentProvider 会伴随 APP 进程初始化一起启动

APP 进程在初始化时，会创建所有运行在本进程中的 ContentProvider 实例，并回调所有 ContentProvider 实例的 `onCreate()` 函数，完成启动

当所有 ContentProvider 启动完成后，会将这些 ContentProvider 的信息发送给 AMS 进行记录，这样其他进程就可以通过 AMS 来访问这些 ContentProvider 了

总体流程如下图：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f9396e76a1304371bdef078f96adbee4~tplv-k3u1fbpfcp-zoom-1.image)

当 ContentProvider 运行所在进程与 APP 进程不同时，ContentProvider 不会在 APP 进程初始化时被启动

当 APP 进程通过 `getContentResolver()` 函数访问 ContentProvider 时，会访问 AMS 尝试获取目标 ContentProvider 的 Binder 通信接口

如果 AMS 中保存有该 ContentProvider 的 Binder 通信接口就直接返回给 APP 进程

如果 AMS 中没有保存有该 ContentProvider 的 Binder 通信接口，说明该 ContentProvider 还未启动，新建进程启动目标 ContentProvider，最终回调目标 ContentProvider 实例的 `onCreate()` 函数，完成启动

当目标 ContentProvider 启动完成后，再将目标 ContentProvider 的信息发送给 AMS 进行记录

总体流程如下图：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8b4c16109d354129be85b87400d29d47~tplv-k3u1fbpfcp-zoom-1.image)

### ContentProvider 的生命周期？

当 ContentProvider 所在进程启动后，会回调 `ContentProvider.onCreate()` 函数

该函数在 `Application.onCreate()` 函数之前执行

### ContentProvider 的设计模式？

ContentProvider 典型实现了提供者模式

对于访问方而言，无需知道知道访问数据的具体细节 (如 SQLite、文件、xml、网络 等)

ContentProvider 会自动根据访问方输入的 Uri 进行匹配 (UriMatcher)，返回对应的数据

### ContentProvider 如何实现权限管理？

* ContentProvider 可以设置 `android:exported` 属性，标识该 ContentProvider 是否可以被其他 APP 应用访问

* ContentProvider 可以设置 `android:permission` 属性，标识访问该 ContentProvider 所需要的权限

  该属性值可以任意指定一个字符串，通常使用应用程序包名作为其中的一部分，这样可以避免和其他 ContentProvider 的权限声明冲突

* 还可以通过设置 `android:readPermission` / `android:writePermission` 属性进一步细化 ContentProvider 的访问权限 (读写权限分离)

  需要注意的是，`android:writePermission` 与 `android:readPermission` 属性的优先级比 `android:permission` 属性的优先级高

* 如果基于 SQLite 存储数据，还可以通过 Uri 和 SQLite 对数据库的访问进行权限设置

### ContentProvider 与 SQLite 的区别？

* SQLite 是一个轻量级的关系型数据库

  SQLite 被嵌入在每一台 Android 系统的设备中，无需用户自行安装

  我们只需要定义相应的 SQL 语句交给系统执行，Android 系统会自动帮助我们维护 SQLite

  也可以通过调用 SQLiteQueryBuilder 等帮助类中定义的函数，由系统根据传入的参数自行构建 SQL 语句并执行

* ContentProvider 是一个 Android 系统提供的统一数据访问接口 (可以供用户自定义)

  通过 ContentProvider，访问方不需要知道访问数据的具体细节 (如 SQLite、文件、xml、网络 等)

  ContentProvider 会自动根据访问方输入的 Uri 进行匹配 (UriMatcher)，返回对应的数据

### ContentProvider 保证线程安全吗？如何解决？

ContentProvider 不保证线程安全

如果不做任何处理，当出现多个线程并发通过 ContentProvider 访问同一数据源时，就会导致线程安全问题 (例如并发写入 SQLite 的同一张数据库表)

最简单的解决办法就是在 ContentProvider 中每个访问数据的函数前加上 synchoronized 关键字

### ContentProvider 如何实现 IPC 通信？

[传送门](#ContentProvider 如何实现 IPC？)

## Handler

[Handler 浅析](https://juejin.cn/post/6965416964634181663)

### 一个线程有几个 Handler？

Handler 可以创建无数个，Handler 对于用户发送消息操作进行了封装，相当于生产者，可以创建无数个

### 一个线程有几个 Looper？如何保证？

一个线程只能绑定一个 looper 对象

每个线程会在自己 ThreadLocalMap 中保存与自己绑定的 looper 对象引用，Key 是 Looper.sThreadLocal 这个字段

这个字段是 static final 的，所以每个线程只会保存一个 looper 对象引用

当调用 `Looper.prepare()` 函数创建 looper 对象的时候，会先确定当前线程是否已经保存了一个 looper 对象引用；如果已经保存过了，说明是二次创建，抛出异常

### Handler 内存泄漏原因？ 为什么其他的内部类没有听说过有这个问题？

Handler 内存泄漏原因是由于 **Handler 可以设置延时消息** 和 **Message 会持有 Handler 对象** 导致的

一般 Handler 对象的存活是跟着四大组件的生命周期的，但是由于 **Handler 可以设置延时消息**，有可能导致当组件的生命周期结束的时候，Message 还没有被处理

这时由于 **Message 持有了 Handler 对象**，如果 Handler 对象中又调用了定义在组件中的函数，就会导致 Handler 对象持有了组件对象 (Java语法，内部类调用外部类函数会持有外部类对象)，此时就会导致 GC 无法将组件对象进行回收，造成 **内存泄露**

我们平时使用的内部类虽然也会持有外部类对象，但是这些内部类的存活是跟着组件的生命周期；当组件生命周期结束后，组件对象和内部类对象会被一起回收，因此不会造成 **内存泄漏**

解决该问题的最有效的方法是：**将 Handler 定义成静态的内部类，在内部持有组件的弱引用**

```java
public static class SafeHandler extends Handler
{
    final WeakReference<Context> mContext;

    public SafeHandler(@NonNull Context context, @NonNull Looper looper)
    {
        super(looper);
        mContext = new WeakReference<>(context);
    }

    @Override
    public void handleMessage(@NonNull Message msg)
    {
        Context thiz = mContext.get();
        thiz.handleMessage(msg);
    }
}
```

### 为何主线程可以直接 new Handler？如果想要在子线程中 new Handler 要做些什么准备？

new Handler 需要对 Looper 进行初始化。需要先调用 `Looper.prepare()` 函数创建 looper 对象，再调用 `Looper.loop()` 函数启动 Looper 轮询 MessageQueue

主线程的 Looper 初始化已经在 `ActivityThread.main()` 函数中完成了，所以我们创建主线程的 Handler 时不需要再对主线程的 Looper 初始化。但如果在子线程中创建 Handler 就需要对 Looper 初始化

### 子线程中绑定的 Looper，消息队列无消息的时候的处理方案是什么？有什么用？

当 MessageQueue 中没有消息时会让 Looper 绑定的线程睡眠，这样可以提高 CPU 资源利用率

### 既然可以存在多个 Handler 往 MessageQueue 中添加数据（发消息时各个 Handler 可能处于不同线程），那它内部是如何确保线程安全的？取消息呢？

MessageQueue 中关于存取消息的操作都使用了 synchronized 锁，并且锁的是 MessageQueue.this 对象，所以同一个 MessageQueue 对象的存取消息操作都是 **原子性** 的，保证了 **线程安全**

### 我们使用 Message 时应该如何创建它？

可以直接 new 也可以通过 `Message.obtain()` 函数创建，但是推荐通过 `Message.obtain()` 函数创建

因为 `Message.obtain()` 函数使用了 Message 的回收复用机制；如果通过直接 new 的方式创建 Message 容易导致 **OOM**

### Looper 死循环为什么不会导致应用卡死（ANR）？

首先要明确 **ANR** 产生的原因：

- KeyDispatchTimeout：输入事件（触摸事件等）在 5s 内没有被处理完成
- BroadcastTimeout： 
  - 前台 Broadcast：onReceiver 函数在 10s 内没有执行结束
  - 后台 Broadcast：onReceiver 函数在 60s 内没有执行结束
- ServiceTimeout： 
  - 前台 Service：onCreate、onStart、onBind 等生命周期函数在 20s 内没有执行结束
  - 后台 Service：onCreate、onStart、onBind 等生命周期函数在 200s 内没有执行结束
- ContentProviderTimeout：ContentProvider 在 10s 内没有处理完成当前事务

Android 中四大组件由 AMS 进行托管，AMS 进程与 App 进程之间通信的渠道是 ActivityThread，而 ActivityThread 的运行依托于主线程中的 Looper 死循环轮询 MessageQueue

可以说 Looper 死循环是保证四大组件的生命周期正常运行的引擎

而 **ANR** 出现的原因是由于四大组件的生命周期出现异常导致的，Looper 死循环与 **ANR** 没有任何关系

### MessageQueue 的休眠为什么不会导致 ANR？

MessageQueue 的休眠分为两种：

- 队列中没有消息，永久睡眠线程
- 未到消息的目标处理时刻，睡眠线程等待时刻，可自动唤醒

无论哪种情况，都不满足 **ANR** 产生的条件

### 你了解 Handler 的同步屏障机制吗？是怎样实现的？

同步屏障的作用是阻碍同步消息，只让异步消息通过。我们日常开发中发送的 message 基本都是同步消息

Handler 的同步屏障机制是基于一种 target 字段为 null 的 message 类型来实现的

当 target 字段为 null 的 message 被 `MessageQueue.next()` 函数取出时，就会开启同步屏障，轮询 MessageQueue 中的异步消息

## 事件分发

[Android全面解析之Window机制](https://juejin.cn/post/6888688477714841608)

[Android事件分发机制一：事件是如何到达activity的？](https://juejin.cn/post/6918272111152726024)

[Android事件分发机制二：核心分发逻辑源码解析](https://juejin.cn/post/6920883974952714247)

[Android事件分发机制三：事件分发工作流程](https://juejin.cn/post/6921238915143696392)

### 什么是事件分发？

事件分发是将屏幕触控信息分发给 view 树的一个套机制

当我们触摸屏幕时，会产生一系列的 MotionEvent 事件对象，经过 view 树的管理者 ViewRootImpl，调用 view 树中根 view 的 `dispatchPointerEvnet()` 函数分发事件

### 简单介绍一下事件分发的流程？

Activity 的 view 树中的根 view 是 DecorView

调用栈如下：

`DecorView.dispatchPointerEvnet()`
--> `Activity.dispatchTouchEvent()`
--> `PhoneWindow.superDispatchTouchEvent()`
--> `ViewGroup.dispatchTouchEvent()`

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/92fd8015131b42b89e5376862cce9da7~tplv-k3u1fbpfcp-zoom-1.image)

也就是一个 **Activity --> Window --> ViewGroup** 的流程

当事件传递给 ViewGroup 时，ViewGroup 会向下搜寻条件符合的控件并把事件分发给它

### 如果要实现一个屏保的效果，能大概说明一下如何实现吗？

当 Activity 接收到 ACTION_DOWN 事件时，会调用 `Activity.onUserInteraction()` 函数

这是一个空实现函数，可供我们自由实现

当用户与 Activity 开始进行交互的时候，就会回调这个函数

可以在这个函数中进行取消屏保的操作

### ViewGroup 中是如何分发事件的？

ViewGroup 分发事件的逻辑在 `ViewGroup.dispatchTouchEvent()` 函数中

ViewGroup 分发事件分为三个步骤：

1. 拦截：判断 ViewGroup 是否需要拦截事件自己处理

2. 寻找子 View：遍历子 View 寻找合适的子 View 来处理事件，如果找到了合适的子 View 为其创建一个 TouchTarget 并插入链表

3. 派发事件：遍历 TouchTarget 链表派发事件

事件分发中有一个重要的规则：**一个触控点的一个事件序列只能给一个 view 处理，即如果 viewGroup 消费了 down 事件，那么子 view 将无法收到任何事件**

DOWN 事件的分发逻辑如下图所示：

![](https://note.youdao.com/yws/api/personal/file/8A165A3072E44F2383E95C5CC882E9BA?method=download&shareKey=66467e732fc4da6905ca52c7b3451330)

MOVE 事件的分发逻辑如下图所示：

![](https://note.youdao.com/yws/api/personal/file/F30AE50E0541490D9B7416CEAC5A02A5?method=download&shareKey=119f79c075b02fa678f0f76d3776b8ff)

### 一个事件序列一定只能由一个 View 来处理吗？

不一定，当上层的 ViewGroup 决定拦截事件后，原先处理事件的 view 就会收到 cancel 事件，事件序列中的后续事件将由上层 ViewGroup 处理

### 事件类型有哪些？

- ACTION_DOWN：第一个手指初次接触到屏幕时触发
- ACTION_UP：最后一个手指离开屏幕时触发
- ACTION_POINTER_DOWN：新的手指在按下之前已经有手指在屏幕上，新的手指初次接触到屏幕时触发
- ACTION_POINTER_UP：要抬起的手指如果抬起后仍然有手指在屏幕上，要抬起的手指离开屏幕时触发
- ACTION_MOVE：手指在屏幕上滑动时触发，一般会多次触发
- ACTION_CANCEL：当事件流被中断时触发

一个完整的事件序列是从 ACTION_DOWN 开始，到 ACTION_UP 或 ACTION_CANCEL 结束，其中间可能有多个 ACTION_MOVE / ACTION_POINTER_DOWN / ACTION_POINTER_UP

### ViewGroup 是如何将多个手指产生的事件准确派发给不同的子 view？

每个原始 MotionEvent 中都包含了当前屏幕所有触控点的信息

当一个子 view 消费了 down 事件之后，ViewGroup 会为该 view 创建一个 TouchTarget，这个 TouchTarget 就包含了该 view 的实例与触控点 id

这里的触控 id 可以是多个，也就是说一个 view 可接受多个触控点的事件序列

ViewGroup 中执行派发事件逻辑的函数是 `ViewGroup.dispatchTransformedTouchEvent()` 

在 `ViewGroup.dispatchTransformedTouchEvent()` 中，会先根据 TouchTarget 中记录的触控点 id 将原始 MotionEvent 进行拆分，这样拆分得到的新 MotionEvent 中就只包含指定触控点 id 所对应的触控点信息，拆分后再将新 MotionEvent 派发给子 View，从而达到精准发送触控点信息的目的

POINTER_DOWN 事件的分发逻辑如下图所示：

![](https://note.youdao.com/yws/api/personal/file/F9A0B60C63EA494F80B9DBD378945D61?method=download&shareKey=80e53f41994e7ef359d2e63b7ce2bf43)

### View 支持多指操作吗？ 

View默认是不支持的

在 `View.onTouchEvent()` 函数的默认实现中，并没有通过传入触控点索引来获取触控点信息

实现多指操作需要我们重写 `View.onTouchEvent()` 函数

### View 中是如何处理事件的？

DOWN 事件的处理逻辑如下图所示：

![](https://note.youdao.com/yws/api/personal/file/094DC0EACCB54F03AABB2A12D48EC25A?method=download&shareKey=d967ff561cf3e2608a5dd39a8412a8c8)

只要调用到了 `View.onTouchEvent()` 函数，无论该 view 是否是 enable，只要是 clickable，就是返回 true，相当于处理了事件

### 可以说说事件分发机制中各函数之间的关系吗？

正常分发流程：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cbc1a84caed84667a88e51250b0eee14~tplv-k3u1fbpfcp-zoom-1.image)

事件流被中断：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/684ef18db9064739b3927b9b70b76a58~tplv-k3u1fbpfcp-zoom-1.image)

## 嵌套滑动

[Android—有趣的嵌套滑动](https://juejin.cn/post/6991476769958133774)

### 如何有效处理嵌套滑动的场景？有了解过嵌套滑动机制吗？

嵌套滑动的场景可以通过嵌套滑动机制进行有效处理

嵌套滑动机制依托于事件分发机制，嵌套滑动机制就是为了解决嵌套滑动而设计的，可以说嵌套滑动是事件分发机制的一个补全

嵌套滑动机制是通过使用 NestedScrollingChild 和 NestedScrollingParent 接口来实现的

### 能大致说明一下 NestedScrollingChild 和 NestedScrollingParent 接口中函数的用途吗？

#### NestedScrollingChild

- `setNestedScrollingEnabled()`：启用或禁用嵌套滚动

  当设置为 true，且当前界面的 view 的层次结构是支持嵌套滚动的 (也就是需要 NestedScrollingParent 嵌套 NestedScrollingChild)，才会触发嵌套滚动

  一般这个方法内部都是直接代理给 NestedScrollingChildHelper 的同名函数即可

- `isNestedScrollingEnabled()`：判断当前 view 是否支持嵌套滑动

  一般也是直接代理给 NestedScrollingChildHelper 的同名函数即可

- `startNestedScroll()`：表示 view 开始滚动了
  
  一般是在 ACTION_DOWN 事件流程中调用，如果返回 true 则表示父 view 支持嵌套滚动
  
  一般也是直接代理给 NestedScrollingChildHelper 的同名函数即可
  
  正常情况下会回调父 view 的 `onStartNestedScroll()` 函数
  
  - axes：滚动方向
  
- `stopNestedScroll()`：一般是在事件结束比如 ACTION_UP / ACTION_CANCLE 事件流程中调用，告诉父 view 滚动结束

  一般也是直接代理给 NestedScrollingChildHelper 的同名函数即可

- `hasNestedScrollingParent()`：判断当前 view 是否有支持嵌套滑动的父 view

  一般也是直接代理给 NestedScrollingChildHelper 的同名函数即可

- `dispatchNestedScroll()`：在子 view 消费滚动距离之后
  
  通过调用该方法，把剩下的滚动距离传给父 view
  
  如果当前没有发生嵌套滚动，或者不支持嵌套滚动，调用该方法也没啥用
  
  内部一般也是直接代理给 NestedScrollingChildHelper 的同名函数即可
  
  返回 true 表示滚动事件分发成功；返回 fasle 表示分发失败
  
  - dxConsumed：被子 view 消费了的水平方向滑动距离
  - dyConsumed：被子 view 消费了的垂直方向滑动距离
  - dxUnconsumed：未被子 view 消费的水平滑动距离
  - dyUnconsumed：未被子 view 消费的垂直滑动距离
  
- `dispatchNestedPreScroll()`：在子 view 消费滚动距离之前把滑动距离传给父 view
  
  相当于把滑动的优先处理权交给父 view，内部一般也是直接代理给 NestedScrollingChildHelper 的同名函数即可
  
  返回 true 代表父 View 消费了滚动距离；返回 false 代表父 View 没有消费滚动距离
  
  - dx：当前水平方向滑动的距离
  - dy：当前垂直方向滑动的距离
  - consumed：输出参数，会将父 view 消费掉的距离封装进该参数。consumed[0] 代表水平方向，consumed[1] 代表垂直方向
  
- `dispatchNestedFling()`：将惯性滑动的速度分发给父 view
  
  内部一般也是直接代理给 NestedScrollingChildHelper 的同名函数即可
  
  返回 true 表示父 view 处理了 Fling 事件；返回 false 表示父 view 没有处理 Fling 事件
  
  - velocityX：表示水平滑动速度
  - velocityY：表示垂直滑动速度
  - consumed：如果传入 true 表示子 view 消费了滑动事件，否则传入 false
  
- `dispatchNestedPreFling()`：在子 view 自己处理惯性滑动前，先将 Fling 事件分发给父 view

  一般来说如果子 view 想自己处理 Fling 事件，就不应该调用该方法给父 view 处理

  如果给了父 view 并且返回 true，那表示父 view 已经处理了，子 view 就不应该再做处理

  返回 false，代表父 view 没有处理，但是不代表父 view 后面就不用处理了

  返回 true 表示父 view 处理了 Fling 事件；返回 false 表示父 view 没有处理 Fling 事件

#### NestedScrollingParent

- `onStartNestedScroll()`：当子 view 调用 `startNestedScroll()` 函数时，会回调该函数

  主要就是通过返回值告诉系统是否需要对后续的滚动进行处理

  返回 true 表示父 view 需要处理嵌套滑动；返回 false 表示父 view 不需要处理嵌套滑动

  - child：该 NestedScrollingParent 包含的直接子 view

    如果只有一层嵌套，那么和 target 是同一个 view

  - target：该 NestedScrollingParent 包含的直接 NestedScrollingChild

  - axes：滚动方向

- `onNestedScrollAccepted()`：在开始嵌套滑动之前，让父 view 或者它的父类执行一些配置的初始化

  如果 `onStartNestedScroll()` 函数返回的是 true，那么紧接着就会调用该函数

- `onStopNestedScroll()`：停止滚动了

  当子 view 调用 `stopNestedScroll()` 函数时会回调该函数

- `onNestedScroll()`：当子 view 调用 `dispatchNestedScroll()` 函数时，会回调该函数
  
  表示父 view 开始处理嵌套滑动了
  
  - dxConsumed：已经被 target 消费掉的水平方向的滑动距离
  - dyConsumed：已经被 target 消费掉的垂直方向的滑动距离
  - dxUnconsumed：tagert 未消费的水平方向的滑动距离
  - dyUnconsumed：tagert 未消费的垂直方向的滑动距离
  
- `onNestedPreScroll()`：当子 view 调用 `dispatchNestedPreScroll()` 函数时，会回调该函数
  
  也就是在 target 处理嵌套滑动之前，会先将机会交给父 view
  
  如果父 view 想先消费部分滚动距离，则将消费了的距离放入 consumed 中，传给 target
  
  - dx：水平滑动距离
  - dy：竖直滑动距离
  - consumed：表示父 view 需要消费的滚动距离，consumed[0] 和 consumed[1] 分别表示父 view 在 x 和 y 方向上消费的距离
  
- `onNestedFling()`：当子 view 调用 `dispatchNestedFling()` 函数时，会回调该函数
  
  父 view 可以捕获对内部 NestedScrollingChild 的 Fling 事件
  
  返回 true 表示子 view 消费了 Fling 事件；返回 false 表示子 view 没有消费 Fling 事件
  
  - velocityX：水平方向的滑动速度
  - velocityY：垂直方向的滑动速度
  - consumed：Fling 事件是否被 target 消费了
  
- `onNestedPreFling()`：当子 view 调用 `dispatchNestedPreFling()` 函数时，会回调该函数

  同 onNestedPreScroll 一样，也是给父 view 优先处理的权利

  返回 true 表示父 view 要处理 Fling 事件，子 view 就不要处理了

- `getNestedScrollAxes()`：返回当前嵌套滑动的方向

  一般直接返回 `NestedScrollingParentHelper.getNestedScrollAxes()` 函数即可

### 简单说明一下嵌套滑动机制的流程？

嵌套滑动机制整体流程如下图：

![](https://note.youdao.com/yws/api/personal/file/A3BB9F99217D4FA7AF0F58EBECA7C5D5?method=download&shareKey=8a79605a28dbea47117c3d6fd5bc83ca)

## View 基础

[Android全面解析之Window机制](https://juejin.cn/post/6888688477714841608)

[从 XML 到 View 显示在屏幕上，都发生了什么?](https://juejin.cn/post/6991483318625632286)

### 如何加载 xml 布局文件？

可以通过调用 `LayoutInflater.inflate()` 函数传入对应的 xml 文件资源 id 来加载对应的 xml 布局

在 `LayoutInflater.inflate()` 函数中，会先调用 `LayoutInflater.createViewFromTag()` 函数解析 xml 文件中的根标签

`LayoutInflater.inflate()` 函数运行逻辑如下图：

![](https://note.youdao.com/yws/api/personal/file/F470A3BE84C740259363601795763E3B?method=download&shareKey=628730c018777459e00fc05b45255d3b)

再调用 `LayoutInflater.rInflateChildren()` 函数遍历 xml 文件中的所有子标签并解析

`LayoutInflater.rInflateChildren()` 函数运行逻辑如下图：

![](https://note.youdao.com/yws/api/personal/file/0CB1914B27A84321AEBB7A68973CAEA3?method=download&shareKey=810a2fe6ae90f6923bdf00fac75c7198)

最终返回整个解析 xml 文件得到的视图树

### Activity 中的视图是在什么时候可见的 (View 的第一次绘制是怎么调到的) ？

根据 Activity 启动流程，最终启动的 Activity 会回调 `ActivityThread.handleResumeActivity()` 函数

在 `ActivityThread.handleResumeActivity()` 函数中，会调用 `WindowManager.addView()` 函数将 Activity 的 DecorView 添加到窗口中

最终会调用到 `ViewRootImpl.requestLayout()` 函数执行 view 树的绘制流程

绘制流程结束后，Activity 中的视图才真正可见

### View 的 onMeasure、onLayout、onDraw 都分别用来干什么？

onMeasure 用于确定 View 本身的所占空间和大小 (长宽)

onLayout 用于确定 View 在其父 View 中的具体位置并根据具体需求调整 View 的最终绘制大小

onDraw 用于将 View 在 Canvas 上绘制出来

一般情况下，在 onMeasure 中确定的 View 的大小就等同于最终绘制大小

### Window、Activity、PhoneWindow、ViewRootImpl、DecorView 之间的关系？

整体关系如下图所示：

![](https://note.youdao.com/yws/api/personal/file/B25B188BE14D4B1998316FC134BA5B8D?method=download&shareKey=21f3a1f083ae2b7c4c70fa3bcde2285b)

在 Android 中 window 是一个抽象的概念，本身并不存在，可以理解为一个视图树就是一个 window

所有的 window 都由系统服务 wms 管理，window 在 wms 中的存在形式是 windowStatus，每一个 windowStatus 会对应着一个 ViewRootImpl，每一个 ViewRootImpl 对应着管理一个视图树

Activity 持有了 PhoneWindow，PhoneWindow 持有了 DecorView 和一些 window 的属性参数，DecorView 是视图树的根

### View 的绘制流程？

Activity 中 View 的绘制流程是从 `ViewRootImpl.requestLayout()` 函数开始的

`ViewRootImpl.requestLayout()` 函数运行逻辑如下图：

![](https://note.youdao.com/yws/api/personal/file/3AEA27E9FC9E4D2CBA2F2ACF9145F6CA?method=download&shareKey=a7d17ffdd33196dd3e6fbce9f44c3899)

`performMeasure()`、`performLayout()`、`performDraw()` 函数会递归执行 view 树中所有 view 的 `onMeasure()`、`onLayout()`、`onDraw()` 函数

`performMeasure()` 函数运行逻辑如下图：

![](https://note.youdao.com/yws/api/personal/file/DE0969AB3D3249A9A7BD8A749F8EBA43?method=download&shareKey=570593d8955fff61cc1f10107c74e382)

`performLayout()` 函数运行逻辑如下图：

![](https://note.youdao.com/yws/api/personal/file/69413B01BE02438E886C8749BBB0905B?method=download&shareKey=24c964f7e81975669bc8ed11f0a97b4f)

`performDraw()` 函数运行逻辑如下图：

![](https://note.youdao.com/yws/api/personal/file/ABF7C614E6C2423A9DE00A61E1DA0014?method=download&shareKey=360fa8829304adbee5fd17aec3163bcc)

Activity 中 View 的绘制流程是在 `onResume()` 函数回调之后才开始的，因此在 `onResume()` 函数回调时 Activity 中的视图仍未被绘制出来

### invalidate 和 requestlayout 的区别？

requestlayout 会重新执行整个视图树的绘制流程 (onMeasure、onLayout、onDraw)

而 invalidate 只会重新执行整个视图树的 onDraw

### requestlayout 的作用范围是多大？

requestlayout 会重新执行整个视图树的绘制流程，而不仅仅局限于某个子 View

同理，invalidate 也是如此 (重新执行整个视图树的 onDraw)

### View 的生命周期？

整体生命周期如下图所示：

![](https://note.youdao.com/yws/api/personal/file/AB1103D66EE24C9DAEF5C367333EFAB3?method=download&shareKey=4964945fcf9ca7aa3dab3a849441f2f5)

### 为什么 View.post 可以获取到 View 的宽高？

View.post 首先会判断当前整个视图树的绘制是否完成 (判断 mAttachInfo 字段是否为空)

如果当前整个视图树的绘制还未开始，就将所要执行的 Runnable 先缓存起来 (HandlerActionQueue)，等待绘制流程结束后再执行此 Runnable

当绘制流程执行 (执行 `performTraversals()` 函数) 时，会将缓存在 HandlerActionQueue 中的 Runnable 添加到主线程的 MessageQueue 中等待执行，并赋值 mAttachInfo 字段

(此时主线程的 Looper 正在执行 `performTraversals()` 函数，因此缓存在 HandlerActionQueue 中的 Runnable 一定是在绘制流程结束后才会被执行)

由于 View.post 传入的 Runnable 在绘制流程结束后才会被执行，因此 View.post 可以获取到 View 的宽高

[View.post为什么可以拿到View的宽高？](https://juejin.cn/post/6844903842782380045)

### 在子线程中一定不能更新 UI 吗？为什么？

不一定

首先主线程更新 UI 的报错是在 `ViewRootImpl.checkThread()` 函数中触发的，只要此函数不被执行，那么主线程更新 UI 是不会报错的

当 requestLayout 被执行时，`ViewRootImpl.checkThread()` 函数会被执行

当 invalidate 被执行时，如果 APP 没有开启硬件加速，`ViewRootImpl.checkThread()` 函数会被执行

> 冷知识：target API 级别为 14 及更高级别，则硬件加速默认处于启用状态

因此在绘制流程开始执行前，是可以在子线程中更新 UI 的

所以就有在 Activity 的 `onCreate()` 函数中通过子线程更新 UI 不会报错的情况出现，因为 Activity 的 `onCreate()` 函数被回调时，ViewRootImpl 还未完成实例化，`ViewRootImpl.checkThread()` 函数不会被执行

[这次彻底搞明白子线程到底能不能更新 UI](https://juejin.cn/post/6915034015544115214)

## 图片加载

[Android中一张图片占据的内存大小是如何计算](https://www.cnblogs.com/dasusu/p/9789389.html)

[Glide 浅析（上）](https://juejin.cn/post/6980599769055887368)

[Glide 浅析（下）](https://juejin.cn/post/6981643538622578695)

### 一张图片所占据的内存大小如何计算？

图片大小的计算公式是：分辨率（图片宽×高）* 每个像素点的大小

但是在 Android 中，图片数据经过解码加载成 Bitmap 后，分辨率可能会发生变化，因此图片占用的实际内存大小就有可能发生变化

例如：通过 `Bitmap.decodeResource()` 函数加载 res 目录下的资源图片时，`Bitmap.decodeResource()` 函数内部会根据图片资源所存放的不同路径进行一次分辨率的转换，因此通过 `Bitmap.decodeResource()` 函数加载同一张资源图片有可能所占用内存的大小是不同的

### 实现一个图片加载框架，需要考虑什么？

- 异步加载：需要两个线程池，一个用于网络加载，一个用于磁盘和内存加载
- 线程切换：Handler
- 缓存策略：三级缓存
  - 第一级缓存：普通的 List 集合
  - 第二级缓存：LruCache
  - 第三级缓存：DiskLruCache
- 防止OOM：对加载的图片进行压缩 (降低分辨率)
- 防止内存泄漏：软引用 (第一级缓存)

###  LaunchActivity 同时加载的图片太多如何优化？

- 使用线程池
- 对图片的分辨率进行压缩
- 设置图片像素点的数据格式，减小像素点大小

###  view 的大小比图片小如何优化？

根据 view 的大小对于图片的分辨率进行压缩

### Glide 是如何实现图片加载优化的？

- 缓存优化：三级缓存
  - 第一级缓存：活动缓存，ArrayList，缓存 **仍被界面所使用** 的图片资源
  - 第二级缓存：内存缓存，LruCache，缓存 **已不被界面所使用却仍存在于内存中** 的图片资源
  - 第三级缓存：磁盘缓存，DiskLruCache，缓存 **根据策略写入磁盘** 的图片资源
  
  Glide 缓存机制的整体流程如下图：
  
  ![Glide 缓存机制](https://note.youdao.com/yws/api/personal/file/WEB9b560a8f311723207e80e585ae062968?method=download&shareKey=88e254d7494ff46006513640c503db1c)
  
- 加载优化：根据控件的宽高缩放图片的分辨率

  Glide 在执行加载图片的请求时，会先判断是否调用了 `override` 函数指定图片的分辨率；如果没有指定，就根据控件的 LayoutParams 缩放图片的分辨率

  所以 Glide 加载时是先对图片压缩再将图片加载进内存，通过  `BitmapFactory#decodeStream` 函数并传入 BitmapFactory.Options 实现

### 使用 Glide 偶尔会出现内存溢出问题，请说说大概是什么原因？

Glide 活动缓存中的资源跟随通过 `Glide#with` 函数传入的 context 的生命周期而变化

如果传入的 context 不合适，会导致 Glide 活动缓存中的资源得不到回收，就有可能导致内存溢出

### Glide 里的运行时缓存，为什么要设置两种不同的类型？

因为 Lru 算法的特点就是会将最近最少使用的资源回收，而对于实际的使用场景中，很有可能会在一个界面中使用大量的图片资源

如果使用 Lru 算法来直接缓存还在 UI 界面上显示的资源，就有可能会造成用户在回退浏览之前的图片时，出现重复网络请求原先图片资源的情况，这样不仅消耗了用户的流量还重复对图片进行压缩处理，占用多余内存的同时加载图片也很缓慢

活动缓存缓存的是仍在 UI 界面上显示的资源，Glide 设置三级缓存的目的就是 **最大程度上减少重复网络请求原先图片资源的情况发生**

### Glide 加载一个 100x100 的图片，是否会先缩放后再加载？如果再把这张图片放到一个 300x300 的 view 上会怎样？

Glide 在执行加载图片的请求时，会先通过计算确定图片的目标宽高。如果我们没有调用 `override()` 函数指定具体的目标宽高，那么 Glide 会根据所要显示图片的 view 的 LayoutParams 进行综合计算

当目标宽高得到后，就会根据目标宽高以及其他参数 (例如：数据源信息) 构建 Key

这个 Key 不仅唯一标识缓存中的资源，也唯一标识 Glide 加载图片的请求操作

也就是说当目标宽高改变之后，即使数据源信息相同，Glide 也会把同一张图片的加载请求看成两个不同的请求

所以如果将一张 100x100 的图片资源放到一个 300x300 的 view 上，Glide 会执行一个新的加载 300x300 的图片的请求，即使这张图片是同一张图片

Glide 不会直接将图片的完整尺寸全部加载到内存中

Glide 会先通过计算确定图片的最终目标宽高，这样通过 `BitmapFactory.decodeStream()` 函数解码的时候就降低了图片的分辨率，可以有效优化图片的占用内存，从而帮助我们节省内存开支

### 为什么选择 Glide 不选择其他的图片加载框架？

#### Glide VS Picasso

- Glide 更加省内存，可以通过 `format` 函数按需调整像素点大小，默认为 ARGB_8888；Picasso 固定为 ARGB_8888
- Glide 支持Gif；Picasso 并不支持

#### Glide VS Fresco

- Fresco 低版本有优势，占用部分 native 内存，但是高版本一样是 java 内存
- Fresco 加载对图片大小有限制；Glide 基本没有
- Fresco 推荐使用 SimpleDraweeView，涉及到 UI 界面，这就不得不考虑迁移的成本
- Fresco 有很多 native 的实现，想改源码成本要大的多；Glide 源码全由 java 编写
- Glide 提供 TransFormation 帮助处理图片；Fresco 并没有

### Glide 的几个显著的优点？

- 生命周期的管理

- 高效的缓存机制

- Bitmap 对象池

  Glide 提供了一个 BitmapPool 来保存 Bitmap 对象

  简单来说就是当需要加载一个 Bitmap 的时候，会根据图片的参数去池子里找到一个合适的 Bitmap 对象，如果没有就重新创建

  BitmapPool 同样是根据 Lru 算法来工作的，从而提高性能

## IPC

[Binder 浅析（上）](https://juejin.cn/post/6976198889833988127)

[Binder 浅析（下）](https://juejin.cn/post/6976202756704960548)

### Android 中的 IPC 方式有哪些？

* Bundle
* 文件共享
* Messenger
* ContentProvider
* Socket

### 为什么使用 Bundle 而不用 HashMap 传输数据？

|          | Bundle     | HashMap      |
| -------- | ---------- | ------------ |
| IPC 方式 | Parcelable | Serializable |

Serializable 通过字节流传输，基于硬盘，同时内部还使用到了反射，可能会触发 GC，性能较低，对于 Android 进程间小数据量通信来说有些得不偿失

而 Parcelable 通过 Binder 传输，基于内存，性能明显高于 Serializable 这一方式

### Intent 本身可以通过 putExtra 函数传输数据，为什么还需要有 Bundle？

```java
// Intent.java
private Bundle mExtras;

public Intent putExtra(String name, int value)
{
	if (mExtras == null)
    {
		mExtras = new Bundle();
	}
	mExtras.putInt(name, value);
	return this;
}

public Intent putExtras(Bundle extras)
{
	if (mExtras == null)
	{
		mExtras = new Bundle();
	}
	mExtras.putAll(extras);
	return this;
}
```

`putExtra()` 函数有多个重载形式，其实现都大同小异

可以看到 `putExtra()` 函数实际上也是通过 Bundle 进行传输，Intent 内部会持有一个 Bundle

当我们通过 `putExtras()` 函数设置我们自定义的 Bundle 时，会将我们 Bundle 中的数据全部添入 Intent 内部的 Bundle 中

### Bundle 的最大内存限制？Bundle 传输有什么要求？

Bundle 实现了 Parcelable，其底层基于 Binder 进行传输，Binder 对于传输的数据具有大小限制，其大小不能超过 1M-8K

经过 Bundle 传输的数据，其类型必须是基础数据类型、实现了 Serializable / Parcelable 的数据类型

### 文件共享可靠吗？为什么？

文件共享不可靠，因为 Android 是基于 Linux 内核的操作系统，Linux 对于文件读写的限制不同于 Windows

在 Windows 上，一个文件如果被加了排斥锁将会导致其他线程无法对其进行访问，包括读和写；而在 Linux 上，对于并发读 / 写文件没有作任何限制，甚至两个线程同时对同一个文件进行写操作都是被允许的，尽管这可能会出问题

因此文件共享不适用于高并发数据同步的 IPC 场景下，使用文件共享进行 IPC 通信，要妥善处理并发读 / 写的问题

### ContentProvider 如何实现 IPC？

ContentProvider 是 Android 提供的专门用于不同应用间进行数据共享的方式，从这一点来看，它天生就适合 IPC

我们可以通过自定义 ContentProvider 的方式，将需要进行 IPC 通信的数据通过自定义的 ContentProvider 发送给其他进程

ContentProvider 提供了 `query()` 、`insert()` 、`delete()` 、`update()` 等函数供我们对数据进行增删改查，我们可以创建一个数据库 (通过 SQLiteOpenHelper)，通过 ContentProvider 来操作这个数据库，从而实现 IPC 通信

#### 数据发送方

自定义 ContentProvider：

```java
public class DBOpenHelper extends SQLiteOpenHelper
{
    // 数据库文件名
    private static final String DB_NAME = "myprovider.db";
    // 表名
    public static final String STUDENT_TABLE_NAME = "student";
    // 数据库版本
    private static final int DB_VERSION = 1;
    // 建表的SQL语句
    private String mCreateTable = "create table if not exists " + STUDENT_TABLE_NAME 
        + "(id integer primary key," + "name TEXT,"+"sex TEXT)";

    public DBOpenHelper(@Nullable Context context)
    {
        super(context, DB_NAME, null, DB_VERSION);
    }

    @Override
    public void onCreate(SQLiteDatabase db)
    {
        // 执行SQL语句
        db.execSQL(mCreateTable);
    }

    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion){ }
}


public class MyProvider extends ContentProvider
{
    // ContentProvider的唯一标识，其他进程通过该标识找到该ContentProvider
    public static final String AUTHORITY = "com.chenjimou.ipc_demo.MyProvider";
    // 创建UriMatcher，用于匹配Uri，确定要访问的是哪一张数据表
    private static final UriMatcher mUriMatcher = new UriMatcher(UriMatcher.NO_MATCH);
    // student表的匹配码
    private static final int MATCH_STUDENT = 0;
    
    private SQLiteDatabase mDB;
    private Context mContext;
    
    static {
        // 注册Uri，匹配码为MATCH_STUDENT
        mUriMatcher.addURI(AUTHORITY, "student", MATCH_STUDENT);
    }

    @Nullable
    @Override
    public Bundle call(@NonNull String method, @Nullable String arg, 
                       @Nullable Bundle extras)
    {
        return super.call(method, arg, extras);
    }

    @Override
    public boolean onCreate()
    {
        mContext =  getContext();
        initProvider();
        return false;
    }

    private void initProvider()
    {
        mDB = new DBOpenHelper(mContext).getWritableDatabase();
    }

    @Nullable
    @Override
    public Cursor query(@NonNull Uri uri, @Nullable String[] projection, 
                        @Nullable String selection, @Nullable String[] selectionArgs, 
                        @Nullable String sortOrder)
    {
        // 匹配Uri，如果Uri访问的是student表
        if (mUriMatcher.match(uri) == MATCH_STUDENT)
        {
            String table = DBOpenHelper.STUDENT_TABLE_NAME;
        	Cursor mCursor = mDB.query(table, projection, selection, 
                                       selectionArgs, null, sortOrder, null);
        	return mCursor;
        }
        return null;
    }

    @Nullable
    @Override
    public String getType(@NonNull Uri uri)
    {
        return null;
    }

    @Nullable
    @Override
    public Uri insert(@NonNull Uri uri, @Nullable ContentValues values)
    {
        if (mUriMatcher.match(uri) == MATCH_STUDENT)
        {
            String table = DBOpenHelper.STUDENT_TABLE_NAME;
            mDB.insert(table, null, values);
        	mContext.getContentResolver().notifyChange(uri, null);
        	return uri;
        }
        return null;
    }

    @Override
    public int delete(@NonNull Uri uri, @Nullable String selection, 
                      @Nullable String[] selectionArgs)
    {
        if (mUriMatcher.match(uri) == MATCH_STUDENT)
        {
            String table = DBOpenHelper.STUDENT_TABLE_NAME;
            return mDB.delete(table, selection, selectionArgs);
        }
        return -1;
    }

    @Override
    public int update(@NonNull Uri uri, @Nullable ContentValues values, 
                      @Nullable String selection, @Nullable String[] selectionArgs)
    {
        if (mUriMatcher.match(uri) == MATCH_STUDENT)
        {
            String table = DBOpenHelper.STUDENT_TABLE_NAME;
            return mDB.update(table, values, selection, selectionArgs);
        }
        return -1;
    }
}
```

通过 UriMatcher 来对 Uri 进行匹配，省略了我们手动对 Uri 字符串进行过滤的操作，UriMatcher 本质上就是一个文本过滤器

使用 UriMatcher，我们需要先注册所要匹配的 Uri 及其匹配成功所返回的匹配码

注册完成后，我们就可以通过调用 `UriMatcher.match(uri)` 函数对传入的 Uri 进行匹配，如果匹配就会返回匹配码

注册 ContentProvider：

```xml
<provider
    android:authorities="com.chenjimou.ipc_demo.MyProvider"
    android:name=".MyProvider"
    android:process=":myprovider"
    android:permission="com.chenjimou.ipc_demo.permission.PROVIDER"
    android:exported="true">
</provider>
```

* authorities：ContentProvider 的唯一标识，其他进程通过该标识找到该 ContentProvider
* name：ContentProvider 的全类名
* process：ContentProvider 运行所在进程
* permission：自定义权限，其他应用想要访问该 ContentProvider 必须声明该权限
* exported：true 表示允许不同进程的组件访问

#### 数据请求方

```java
public class MainActivity extends AppCompatActivity
{
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        
        Uri uri = Uri.parse("content://com.chenjimou.ipc_demo.MyProvider/student");
        
		ContentValues contentValues = new ContentValues();
		contentValues.put("name", "张三");
		contentValues.put("sex", "男");
		getContentResolver().insert(uri, contentValues);
        
        getContentResolver().delete(uri, "name = ?", new String[]{"张三"});
        
        ContentValues contentValues1= new ContentValues();
		contentValues1.put(NAME, "张三");
		contentValues1.put(SEX, "女");
		getContentResolver().update(uri, contentValues1, "name = ?", 
                                    new String[]{"张三"});
        
        Cursor cursor = getContentResolver().query(uri, new String[]{"name", "sex"}, 
                                                   null, null, null);
		if (cursor != null)
        {
			List<Student> students = new ArrayList<>();
			while (cursor.moveToNext())
            {
                String name = cursor.getString(0);
                String sex = cursor.getString(1);
				Log.e("MainActivity", "name: " + name + " sex: " + sex);
			}
		}
    }
}
```

访问具体某个 ContentProvider，需要通过 ContentResolver 传入 Uri 进行访问

Uri 的组成为：**标准前缀 + authorities + 具体要访问的内容**

* 标准前缀：`content://`
* authorities：ContentProvider 在 AndroidManifest 文件中声明的唯一标识
* 具体要访问的内容：例如数据库中的哪一张数据表

### AIDL 原理？AIDL 函数是在 Service 线程池中调用的，这个线程池是 Service 自己创建的吗？

我们可以在 aidl 文件中定义用于 IPC 通信的接口

经过编译后，Android Studio 会动态为我们生成一个与 aidl 文件同名的 Java 类，其中含有两个静态内部类，Stub 和 Proxy

Stub 是一个静态抽象类，对应服务端，服务端需要继承 Stub 实现对应的函数

Proxy 对应客户端，其定义了对应服务端的代理函数，代理函数会通过 Binder 机制与服务端进行 IPC 通信，从而调用到服务端中对应的同名函数

AIDL 最常见的搭配就是 Activity 与 Service 的绑定，通过 AIDL，让 Activity 能够 IPC 调用 Service 中的函数

通过 AIDL 调用 Service 中的函数，函数会运行在一个由 Binder 驱动控制的线程中，这种线程被称为 Binder 线程

Binder 线程由 Binder 驱动通过 Binder 线程池管理，Binder 线程池在进程启动的时候就会被创建

### 什么是 Messenger？Messenger 如何使用？

如果说 AIDL 对于 IPC 来说还是不够简便，谷歌还为我们设计出了 Messenger

Messenger 等同在 AIDL 之上再进行了一层封装，并使用 Handler 来接收来自其他进程发来的信息

#### 服务端

```java
public class MessengerService extends Service
{
    /**
     * 服务端的 Messenger
     */
    Messenger mServiceMessenger = new Messenger(new Handler()
	{
        @Override
        public void handleMessage(final Message msg)
        {
            if (msg != null && msg.arg1 == ConfigHelper.MSG_ID_CLIENT)
            {
                if (msg.getData() == null)
                {
                    return;
                }
                // 接收来自客户端的信息
                String content = (String) msg.getData().get(ConfigHelper.MSG_CONTENT);
                // 回复信息给客户端
                Message replyMsg = Message.obtain();
                replyMsg.arg1 = ConfigHelper.MSG_ID_SERVER;
                Bundle bundle = new Bundle();
                bundle.putString(ConfigHelper.MSG_CONTENT, "收到你的消息了，态度好点");
                replyMsg.setData(bundle);

                try
                {
                    // 发送回信给客户端
                    msg.replyTo.send(replyMsg);
                }
                catch (Exception e)
                {
                    e.printStackTrace();
                }
            }
        }
    });

    @Nullable
    @Override
    public IBinder onBind(final Intent intent)
    {
        return mServiceMessenger.getBinder();
    }
}
```

#### 客户端

```java
public class MainActivity extends AppCompatActivity
{
    /**
     * 客户端的 Messenger
     */
    Messenger mClientMessenger = new Messenger(new Handler()
	{
        @Override
        public void handleMessage(final Message msg)
        {
            if (msg != null && msg.arg1 == ConfigHelper.MSG_ID_SERVER)
            {
                if (msg.getData() == null)
                {
                    return;
                }
				// 接收来自服务端的信息
                String content = (String) msg.getData().get(ConfigHelper.MSG_CONTENT);
            }
        }
    });

    // 服务端的 Messenger
    private Messenger mServerMessenger;

    private ServiceConnection mServiceConnection = new ServiceConnection()
    {
        @Override
        public void onServiceConnected(final ComponentName name, final IBinder service)
        {
            mServerMessenger = new Messenger(service);
        }

        @Override
        public void onServiceDisconnected(final ComponentName name)
        {
            mServerMessenger = null;
        }
    };

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

		Intent intent = new Intent(this, MessengerService.class);
        bindService(intent, mServiceConnection, BIND_AUTO_CREATE);

        Message message = Message.obtain();
        message.arg1 = ConfigHelper.MSG_ID_CLIENT;
        Bundle bundle = new Bundle();
        bundle.putString(ConfigHelper.MSG_CONTENT, "能不能收到，给爷说句话");
        message.setData(bundle);

        // 指定回信方是客户端的 Messenger
        message.replyTo = mClientMessenger;     

        try
        {
            // 发送消息给服务端
            mServerMessenger.send(message);
        }
        catch (Exception e)
        {
            e.printStackTrace();
        }
    }

    @Override
    protected void onDestroy()
    {
        super.onDestroy();
        unbindService(mServiceConnection);
    }
}
```

### 了解 mmap 吗？

Linux 中通过将一个虚拟内存区域与一个磁盘上的物理内存区域关联起来，以初始化这个虚拟内存区域的内容，这个过程称为 **内存映射 (memory mapping)**

mmap 是 Linux 中一种实现内存映射的方式

mmap 简单的讲就是 **将指定的一块虚拟内存区域与指定的一块物理内存区域建立映射关系**

映射关系建立后，对这块虚拟内存区域的修改可以直接反应到物理内存上；反之物理内存中对这段区域的修改也能直接反应到虚拟内存上

Binder IPC 通信的一次拷贝就是通过 mmap 来实现的

### 了解 Binder 机制吗？说一下其优缺点？

Binder 通信最大的特点就是 **只需要一次数据拷贝**

在所有 Linux 系统里的跨进程通信方式中，除了共享内存，都需要进行两次数据拷贝：将发送方进程中的数据先拷贝到内核空间中，再将内核空间中的数据拷贝到接收方进程

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9ad5f9e0a47a44f7972a71b8209c0d29~tplv-k3u1fbpfcp-zoom-in-crop-mark:717:0:0:0.awebp)

而 Binder 通信使用了 **mmap** 技术，实现了内存映射，所以 Binder 通信只需要进行一次数据拷贝 —— 将发送方进程中的数据拷贝到内核空间中

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f7fcbd1436c14b84a7872299991621a9~tplv-k3u1fbpfcp-zoom-in-crop-mark:717:0:0:0.awebp)

Binder 通信的缺点是：

- 需要占用线程
- 无法传输大数据

### Android 访问文件需要经历几次拷贝？

**两次拷贝**

常规文件操作使用页缓存机制

这样造成读文件时需要先将文件页从磁盘拷贝到页缓存中，由于页缓存处在内核空间，不能被用户进程直接寻址，所以还需要将页缓存中数据页再次拷贝到内存对应的用户空间中

反之写文件时需要先将数据拷贝到内核缓存中，再由系统调用，将数据拷贝到文件中，才能完成写入

### Linux OS 中 IPC 手段有哪些？

* 管道
* 信号
* 消息队列
* 共享内存
* 信号量
* Socket

### 了解 Socket 如何进行 IPC 吗？说一下其优缺点？

使用 Socket 进行 IPC 主要是通过网络来实现的

Socket 封装了 TCP / IP 协议，使得我们可以直接通过调用 Socket 的 API 来实现对于网络的访问

#### 服务端

```java
package com.example.chenjimou.socketserver;

public class ServerActivity extends AppCompatActivity
{
    Thread mServerThread;
    PrintWriter mPrintWriter;
    BufferedReader mBufferedReader;
    ServerSocket mServerSocket;
    boolean mIsConnectClient = false;

    @Override
    protected void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_Server);

        mServerThread = new Thread()
        {
			@Override
			public void run()
            {
				connectClient();
			}
		}.start();
    }

    @Override
    protected void onResume()
    {
        super.onResume();
        mPrintWriter.println("收到你的消息了，态度好点");
    }

    @Override
    public void onDestroy()
    {
        super.onDestroy();
        disConnectClient();
    }
    
    void connectClient()
    {
        // 防止重复连接
        if (mIsConnectClient)
            return;
        
        try
        {
            // 创建 ServerSocket，监听 8088 端口
            mServerSocket = new ServerSocket(8088);
        }
        catch (IOException e)
        {
            e.printStackTrace();
        }

        // 循环监听端口直至连接成功
        while (mClientSocket == null)
        {
            try
            {
                // 接收 Socket，与客户端建立连接
                mClientSocket = mServerSocket.accept();
                mPrintWriter = new PrintWriter(new BufferedWriter(
                    	new OutputStreamWriter(mClientSocket.getOutputStream())), true);
                mIsConnectClient = true;
            }
            catch (IOException e)
            {
                e.printStackTrace();
            }
        }
        
        try
        {
            mBufferedReader = new BufferedReader(
                	new InputStreamReader(mClientSocket.getInputStream()));
            do {
                // 接收客户端的消息
                String msg = bufferedReader.readLine();
                // 延时一秒后再获取客户端消息，避免服务端压力过大
                SystemClock.sleep(1000);
            } while (!Thread.currentThread().isInterrupted());
        }
        catch (IOException e)
        {
            e.printStackTrace();
        }
        catch (InterruptedException e)
        {
            // 如果当前线程被中断就直接退出
            return;
        }
    }

    void disConnectClient()
    {
        mIsConnectServer = false;
        if (mClientSocket != null)
        {
            try
            {
                mClientSocket.shutdownOutput();
                mClientSocket.shutdownInput();
                mClientSocket.close();
            }
            catch (IOException e)
            {
                e.printStackTrace();
            }
        }
        if (mServerSocket != null)
        {
            try
            {
                mServerSocket.close();
            }
            catch (IOException e)
            {
                e.printStackTrace();
            }
        }
        // 中断监听线程
        mServerThread.interrupt();
    }
}
```

#### 客户端

```java
package com.example.chenjimou.socketclient;

public class ClientActivity extends AppCompatActivity
{
    Thread mClientThread;
    PrintWriter mPrintWriter;
    BufferedReader mBufferedReader;
    Socket mClientSocket;
    boolean mIsConnectServer = false;

    @Override
    protected void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_client);
        
        mClientThread = new Thread()
        {
			@Override
			public void run()
            {
				connectServer();
			}
		}.start();
    }
    
    @Override
    protected void onResume()
    {
        super.onResume();
        mPrintWriter.println("能不能收到，给爷说句话");
    }

    @Override
    protected void onDestroy()
    {
        super.onDestroy();
        disConnectServer();
    }

    void connectServer()
    {
        // 防止重复连接
        if (mIsConnectServer)
            return;

		// 连接失败累计次数
        int count = 0;

        // 重复发起连接请求直至连接成功
        while (mClientSocket == null)
        {
            try
            {
                // 创建 Socket，与服务端建立连接
                mClientSocket = new Socket("10.10.14.160", 8088);
                mPrintWriter = new PrintWriter(new BufferedWriter(
                    	new OutputStreamWriter(mClientSocket.getOutputStream())), true);
                mIsConnectServer = true;
            }
            catch (IOException e)
            {
                e.printStackTrace();
                // 延时一秒后再发起连接请求，避免服务端压力过大
                SystemClock.sleep(1000);
                count++;
                // 失败超过五次就直接退出
                if (count == 5)
                {
                    return;
                }
            }
        }

        try
        {
            mBufferedReader = new BufferedReader(
                	new InputStreamReader(mClientSocket.getInputStream()));
            do {
                // 接收服务端的消息
                String msg = bufferedReader.readLine();
                // 延时一秒后再获取服务端消息，避免客户端压力过大
                SystemClock.sleep(1000);
            } while (!Thread.currentThread().isInterrupted());
        }
        catch (IOException e)
        {
            e.printStackTrace();
        }
        catch (InterruptedException e)
        {
            // 如果当前线程被中断就直接退出
            return;
        }
    }

    void disConnectServer()
    {
        mIsConnectServer = false;
        if (mClientSocket != null)
        {
            try
            {
                mClientSocket.shutdownOutput();
                mClientSocket.shutdownInput();
                mClientSocket.close();
            }
            catch (IOException e)
            {
                e.printStackTrace();
            }
        }
        // 中断监听线程
        mClientThread.interrupt();
    }
}
```

Socket 的优点在于并不仅仅可以用于跨进程通信，还可以用于跨设备通信

因为 Socket 基于网络，所以相较于其他 IPC 方式而言，Socket 是一种较为复杂的通信方式

通常客户端需要开启单独的监听线程来接收从服务端发过来的数据

客户端发送数据给服务端，服务端也需要开启单独的监听线程来接收从客户端发过来的数据

并且由于 Socket 基于网络，导致数据传输具有较大的延时，客户端和服务端都需要对接收到的数据进行同步，这是一件很麻烦的事情

### Socket IPC 通信的使用场景？

由于 Socket 最大的优点在于可以用于跨设备通信，因此 Socket 更多的是用在物联网场景中

利用蓝牙、WiFi等硬件模块与手机设备进行 Socket 连接并通信

### 常见 IPC 类型之间的比较？

|        | Binder                                     | 共享内存                                 | Socket                                                |
| ------ | ------------------------------------------ | ---------------------------------------- | ----------------------------------------------------- |
| 性能   | 需要拷贝一次                               | 无需拷贝                                 | 需要拷贝两次                                          |
| 特点   | 基于 C/S 架构，易用性高                    | 控制复杂，易用性差                       | 基于 C/S 架构，作为一款通用接口，其传销效率低，开销大 |
| 安全性 | 为每个APP分配UID，同时支持实名和匿名，安全 | 依赖上层协议，访问接入点是开放的，不安全 | 依赖上层协议，访问接入点是开放的，不安全              |

## 序列化

[序列化/反序列化，我忍你很久了](https://juejin.cn/post/6844904176263102472)

[解析Serializable原理](https://juejin.cn/post/6844904049997774856)

[Android Protobuf应用及原理](https://juejin.cn/post/6844903582743920648)

[更小、更快、更简单Google ProtoBuf 跨语言通信协议](https://juejin.cn/post/6844903481040437255)

### 介绍一下 Serializable？Serializable 的原理是什么？

Serializable 是 Java 原生提供的标准序列化方案

Java 原生的序列化方案是 **将对象序列化后的二进制串通过字节流进行传输**，而 Serializable 则是用来标志当前类可以被序列化 / 反序列化

Java 原生的序列化是通过 `ObjectOutputStream.writeObject()` 函数 和 `ObjectInputStream.readObject()` 函数来实现的

被 Serializable 标志的类需要注意以下几点：

- 被 static 修饰的字段和被 transient 修饰的字段不会参与序列化

  当对象被反序列化时，其中不参与序列化的字段会通过无参构造函数重新进行初始化 (赋值为默认值)

  因此，实现 Serializable 接口的类的无参构造函数必须可以访问

- 实现了 Serializable 接口的类，其子类也是可序列化的

  如果子类实现了 Serializable 接口但是父类没有实现，那么父类的的无参构造函数必须可以访问

- 在反序列化时，类的字段发生变化，不会发生错误。与原来相比新增的字段会通过无参构造函数重新进行初始化 (赋值为默认值)

Serializable 序列化需要通过 IO 流进行传输，基于磁盘，同时还使用到了反射，可能会触发 GC，性能开销较大；由于其基于 IO 流，因此更适合用于网络传输和持久化

### serialVersionUID 是什么？作用是什么？

serialVersionUID 是序列化 / 反序列化前后的唯一标识符，用于反序列化时检验类是否一致

在反序列化时，JVM 会把字节流中的 serialVersionUID 和反序列化类中的 serialVersionUID 做比对，只有两者一致，才能进行反序列化，否则就会报异常来终止反序列化的过程

如果类中没有主动定义一个 serialVersionUID 的话，那么 Java 运行时环境会根据该类的各方面信息自动地为它生成一个默认的 serialVersionUID

一旦类的结构或者信息发生了更改，那么这个默认生成的 serialVersionUID 也会跟着变化

因此在实现了 Serializable 接口的类中，建议主动定义一个确定的 serialVersionUID，避免后续开发中因为修改了类的结构而发生不必要的错误

### 介绍一下 Parcelable？Parcelable 的原理是什么？

Parcelable 是 Android 原生提供的序列化方案。Parcelable 主要用于 Android 四大组件间通信以及进程间通信

Parcelable 的序列化方案是 **将对象中的数据写入 Parcel，将 Parcel 通过 binder 通信传输到目标进程中**

在实现了 Parcelable 接口的类中，需要重写 `Parcelable.writeToParcel()` 函数，完成将对象中的数据写入 Parcel (调用 `Parcel.writeXXX()` 函数)

还需要创建一个 Creator 类型的字段，用于反序列化，调用参数类型是 Parcel 的单参构造函数创建对象

因此还需要创建一个参数类型是 Parcel 的单参构造函数，将 Parcel 中的数据读取出来对字段进行赋值 (调用 `Parcel.readXXX()` 函数)

Parcelable 序列化需要通过 Binder 进行传输，基于内存，性能开销较低；由于 Binder、Parcel 都是只存在于 Android 系统中的概念，并且 Binder 是基于内存的 IPC 传输方式，因此更适合用于 Android 系统中的 IPC 通信

### 两者各自的优缺点是什么？应用场景呢？

|          | Serializable                                                 | Parcelable                                                   |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 优点     | 使用简单，只需要实现接口即可；通过字节流传输，适合网络传输和持久化。 | 通过 binder 传输，基于内存，性能较高。                       |
| 缺点     | 通过字节流传输，基于硬盘，同时内部还使用到了反射，可能会触发 GC，性能较低。 | 使用复杂，需要重写函数；同时 Parcelable 不能保证，当外部设备（硬盘）发生变化时数据的持续性。 |
| 应用场景 | 网络传输和持久化                                             | IPC 通信                                                     |

### Serializable 序列化为什么会破坏单例模式？如何解决？

通过 Serializable 进行序列化传输，需要调用 `ObjectInputStream.readObject()` 函数来反序列化对象

在 `ObjectInputStream.readObject()` 函数中，会通过反射调用单例类中的私有构造函数来创建一个新的对象，导致序列化前的对象与反序列化后的对象不一致，致使单例模式失效

如何解决？

在 `ObjectInputStream.readObject()` 函数中，会尝试通过反射调用单例类中的私有 `readObject()` 和 `readResolve()` 函数 (不是重写，是我们自己定义的名为 readObject / readResolve 的函数)，如果我们没有定义，就会根据默认的逻辑来进行反序列化

根据 `ObjectInputStream.readObject()` 函数内部的调用栈，我们知道 `readObject()` 函数会优先于 `readResolve()` 函数执行，所以解决方案就是 —— 在单例类中定义一个 `readResolve()` 函数，使其返回单例对象

```java
class Single implements Serializable
{
	private static final long serialVersionUID = 1234567890L;
    private volatile static Single single;
    
    private Single() { }
    
    public static Single getInstance()
    {
        if (single == null)
        {
            synchronized (Single.class)
            {
                if (single == null)
                {
                    single = new Single();
                }
            }
        }
        return single;
    }
    
    // 如果不定义 readResolve 函数，就会导致单例模式在反序列化后失效
    private Object readResolve()
    {
        return single;
    }
}
```

[为什么序列化会破化单例？](https://blog.csdn.net/qq_33709508/article/details/105356327)

### 有了解过 protobuf 吗？简单介绍一下？

Google Protocol Buffer (简称 Protobuf) 是 Google 公司提出的一种灵活高效可序列化的数据协议，相较于传统的 Json、XML 等通讯数据协议，具有更快、更简单、更轻量级等特性

支持多种语言，只需定义好数据结构，利用 Protobuf 框架生成源代码，就可很轻松地实现数据结构的序列化和反序列化；一旦需求有变，可以更新数据结构，而不会影响已部署程序

protobuf 具有以下优点：

* 代码生成机制

  protobuf 序列化通过编写 proto 文件，指定 Java 类的包名、类名等信息，指定需要进行序列化的字段，再通过 Protobuf 提供的 Gradle Plugin 就可以在指定目录下自动编译生成用于序列化的类，其含有用于序列化 / 反序列化等函数

* 高效性

  序列化时间效率对比：

  | 数据协议 | 1000条数据 | 5000条数据 |
  | -------- | ---------- | ---------- |
  | Protobuf | 195ms      | 647ms      |
  | Json     | 515ms      | 2293ms     |

  序列化空间效率对比：

  | 数据协议 | 5000条数据 |
  | -------- | ---------- |
  | Protobuf | 22MB       |
  | Json     | 29MB       |

  从上面的数据可以看出来，使用 Protobuf 序列化时，与使用 Json 相比，不管在时间和空间上都是更加高效

* 支持向后兼容和向前兼容

  当客户端和服务器同时使用一个协议时，当客户端在协议中增加一个字节，并不会影响客户端的使用

* 支持多种编程语言

  在 Google 官方发布的源代码中包含了 c++、java、Python、Go 等多种语言

Protobuf 的缺点也很明显，就是可读性差

为了追求性能上的提升，Protobuf 采用了二进制格式进行编码，这直接导致了可读性差，要是不配合 proto 结构文件人类根本看不懂

## SharedPreferences

[反思｜官方也无力回天？Android SharedPreferences的设计与实现](https://juejin.cn/post/6884505736836022280)

[SharedPreferences 源码解析：自带的轻量级 K-V 存储库](https://juejin.cn/post/6844904036714430472)

[今日头条 ANR 优化实践系列 - 告别 SharedPreference 等待](https://juejin.cn/post/6961961476047568932)

### SharedPreferences 的缺陷？

#### 不支持多进程场景

在第一次通过 `context.getSharedPreferences(String name, int mode)` 函数获取 SharedPreferences 对象时，会对 xml 文件进行一次读取，并将文件中所有内容（即所有的键值对）缓存到一个 Map 集合（mMap）中

SharedPreferences 对象创建后会被缓存在一个 Map 集合中，以 xml 文件名作为 Key，再次调用 `context.getSharedPreferences(String name, int mode)` 函数得到的是缓存中的对象，因此 mMap 集合在内存中只会被初始化一次

由于进程间的内存隔离，不同进程中的 mMap 集合无法做到数据同步，通过 `getXXX` 函数获取数据实际上是访问 mMap 集合，而不是读取 xml 文件，这就导致了 SharedPreferences 多进程下读写不同步

#### 容易导致 ANR

##### 问题一

第一次通过 `context.getSharedPreferences()` 函数获取 SharedPreferences 对象时，会单独使用一个线程来读取对应的 xml 文件

若此时 UI 线程通过 `getXXX()` 函数尝试访问 SharedPreferences 中的内容，如果读取还未结束，则 UI 线程会被阻塞，直到读取完成

如果 xml 文件中的内容过多，就会导致 UI 线程被阻塞的时间过长，最终造成 ANR

所以 xml 文件中的内容不宜过多，xml 文件大小不宜过大，Google 推荐我们将 **不同业务模块的数据分文件存储**，根据业务逻辑将数据存放在不同的 xml 文件中

##### 问题二

当通过 `apply()` 函数提交修改时，会先向 QueuedWork 提交一个 awaitCommit 任务，将 mModified 集合 (Editor 中保存 `putXXX()` 函数修改记录的集合) 与 mMap 集合合并，将合并后得到的结果 (MemoryCommitResult) 封装成 Runnable (writeToDiskRunnable) 交给子线程 queued-work-looper 执行

若在 writeToDiskRunnable 任务被执行前，Activity 处于 Pause / Stop 状态，就会执行 `QueuedWork.waitToFinish()` 函数，执行先前提交的 awaitCommit 任务

awaitCommit 任务会通过 `CountDownLatch.await()` 阻塞 UI 线程，直至 writeToDiskRunnable 任务被执行，SharedPreferences 写入完成

writeToDiskRunnable 任务会将 MemoryCommitResult **覆盖** 写入 xml 文件，并通过 `CountDownLatch.countDown()` 函数唤醒 UI 线程

根据 SharedPreferences 设计的初衷，这种等待行为不会有什么问题，但在实际开发过程中由于 `apply()` 函数被过度使用 (SPUtils.putXXX() 等这类设计的粗暴使用)，UI 线程所等待的时间被不可控的拉长，最终导致 ANR，这个问题越在业务繁重的 App 上体现得越明显

### SharedPreferences 是进程安全的吗？为什么？

不是，SharedPreferences 本身 [不支持多进程场景](#不支持多进程场景)

### SharedPreferences 是线程安全的吗？commit 和 apply 的区别？

SharedPreferences 是线程安全的，其内部使用了三把锁来保证线程安全：

* mLock：用于保证 mMap 集合线程安全，所有涉及 mMap 的操作都需要使用 mLock 锁
* mWritingToDiskLock：用于保证读写 xml 文件线程安全，所有涉及读写 xml 文件的操作都需要使用 mWritingToDiskLock 锁
* mEditorLock：用于保证 mModified 集合线程安全，所有涉及 mModified 的操作都需要使用 mEditorLock 锁

commit 和 apply 操作都会将 mModified 和 mMap 集合进行合并，二者的区别在于，在将合并后得到的结果写入 xml 文件时，commit 是**同步**写入（会阻塞 UI 线程）；apply 是**异步**写入（不会阻塞 UI 线程）

###  apply 之后立即获取数据能够获取得到吗？

在 `apply()` 函数中，会调用 `commitToMemory()` 函数将 mModified 和 mMap 集合进行合并

通过 `getXXX()` 函数获取数据，实际上是访问 mMap 集合，此时的 mMap 集合已经与 mModified 集合进行了合并，是能够立即获取到新提交的数据的

### 使用 SharedPreferences 的 get 和 put 函数读写数据会面临什么问题？IO 性能方面怎么解决？

使用 SharedPreferences 的 get 和 put 函数读写数据都有可能会 [造成 ANR](#容易导致 ANR)

SharedPreferences 中涉及 IO 的操作有：

1. 内存中第一次初始化 SharedPreferences 对象时，会对 xml 文件进行一次读取，并将文件内容缓存到 mMap 集合中

2. SharedPreferences.Editor 调用 `commit()` / `apply()` 函数提交修改时，会将 mMap 集合合并后的结果覆盖写入 xml 文件

针对第一种 IO 操作的优化，可以采取预加载的方式来触发 xml 文件的加载和解析，这样在真正使用的时候大概率 SharedPreferences 已经加载解析完毕了

但最重要的核心场景中的 SharedPreferences 一定不能太大，Google 官方的声明还是有必要遵守一下，**轻量级的数据持久化存储方式**，不要存太多数据，避免文件过大，导致前期的加载解析耗时过久

针对第二种 IO 操作的优化，核心思路是 —— 在 `QueuedWork.waitToFinish()` 函数触发之前移除 awaitCommit 任务

`QueuedWork.waitToFinish()` 函数是由 ActivityThread 触发的，ActivityThread 中有一个 Handler 变量，我们可以通过 Hook 拿到此变量，给此 Handler 设置一个 callback，此 callback 会优先于 handleMessage 处理

在 callback 中通过反射获取 QueuedWork.sFinishers 集合 (Android 8.0 以下此集合叫 sPendingWorkFinishers，Android 8.0 以上此集合叫 sFinishers)，并将此集合清空

还有一种就是全局替换写入方式，通过插桩的方式，替换所有的 API 实现，采用其他的存储方式，这种方式修复成本和风险较大，但是后期可以随机的替换存储方式，使用比较灵活

### SharedPreferences 产生 ANR 的原因？如何解决？

随着 App 业务变得越来越繁重，SharedPreferences 很 [容易导致 ANR](#容易导致 ANR) 的产生

字节跳动技术团队针对其原因提出了两套解决方案：

* 问题一：

  针对加载慢出现 ANR 的问题，使用较多的是采用预加载的方式来触发 xml 文件的加载和解析，这样在真正使用的时候大概率 SharedPreferences 已经加载解析完毕了；

  核心场景中的 SharedPreferences 一定不能太大，Google 官方的声明还是有必要遵守一下，**轻量级的数据持久化存储方式**，不要存太多数据，避免文件过大，导致前期的加载解析耗时过久

* 问题二：

  解决问题二的核心在于 —— 在 `QueuedWork.waitToFinish()` 函数触发之前移除 awaitCommit 任务

  `QueuedWork.waitToFinish()` 函数是由 ActivityThread 触发的，ActivityThread 中有一个 Handler 变量，我们可以通过 Hook 拿到此变量，给此 Handler 设置一个 callback，此 callback 会优先于 handleMessage 处理

  在 callback 中通过反射获取 QueuedWork.sFinishers 集合 (Android 8.0 以下此集合叫 sPendingWorkFinishers，Android 8.0 以上此集合叫 sFinishers)，并将此集合清空

  还有一种就是全局替换写入方式，通过插桩的方式，替换所有的 API 实现，采用其他的存储方式，这种方式修复成本和风险较大，但是后期可以随机的替换存储方式，使用比较灵活

### 有了解过 MMKV、Data Store 等框架吗？相较于 SharedPreferences，他们的优点是什么？

MMKV 是腾讯开发团队推出的基于 mmap 内存映射的 key-value 开源框架，底层序列化 / 反序列化使用 protobuf 实现，性能高，稳定性强。从 2015 年中至今在微信上使用，其性能和稳定性经过了时间的验证

DataStore 是 Google Jetpack 推出的一种数据存储解决方案，允许我们使用 protobuf 存储键值对或类型化对象。使用 Kotlin 协程和 Flow 以异步、一致的事务方式存储数据

在轻量级数据持久化存储的场景上，一直以来我们的选择都是 SharedPreferences；但随着业务场景和开发技术的不断发展，SharedPreferences 所存在的缺陷已经不能被忽视了，以上的两个框架，都一定程度上解决了 SharedPreferences 所存在的缺陷：

* SharedPreferences 不支持多进程场景；MMKV 基于 mmap 内存映射持久化数据，完美解决了多进程下数据同步问题；DataStore 依旧不支持多进程场景 (可能 Google 认为一个轻量级数据持久化存储框架不需要多进程吧......)
* SharedPreferences 会阻塞 UI 线程导致 ANR；MMKV 基于 mmap 内存映射持久化数据，由操作系统负责将数据写入文件，不必担心 App Crash 导致数据丢失，自然就没有必要阻塞 UI 线程了；DataStore 内部使用 Kotlin 协程和 Flow 通过挂起的方式来避免阻塞 UI 线程，避免产生 ANR

| 功能                  | MMKV | JetPack DataStore | SharedPreferences |
| --------------------- | ---- | ----------------- | ----------------- |
| **是否阻塞 UI 线程**  | ×    | ×                 | √                 |
| 是否线程安全          | √    | √                 | √                 |
| **是否支持多进程**    | √    | ×                 | ×                 |
| **是否支持 protobuf** | √    | √                 | ×                 |
| 是否类型安全          | ×    | √                 | ×                 |
| 是否能监听数据变化    | ×    | √                 | √                 |

##  Jetpack

[如何看待Android的Jetpack这一系列库？](https://www.zhihu.com/question/302221538?sort=created)

[Jetpack 是什么？](https://zhuanlan.zhihu.com/p/334350927)

[Android ViewModel，再学不会你砍我](https://juejin.cn/post/6844903919064186888)

[将Room的使用方式塞到脑子里](https://juejin.cn/post/6992875656707211271)

### 说一下你对于 Jetpack 的理解？

按照 Google 的解释，jetpack 是一套组件库，为了能让开发者更好地进行开发

>  Jetpack 是一套组件库，可帮助开发人员遵循最佳实践，减少样板代码并编写可在 Android 版本和设备上一致工作的代码，以便开发人员可以专注于他们关心的代码

Google 将所有 **目前仍被使用且打算继续维护的组件** 都归到了 jetpack 的范畴中，包括我们熟知的 ViewPager、Fragment、RecyclerView 等等

当然 jetpack 中也出现了很多新的组件，例如：Lifecycle、ViewModel、LiveData、Room 等等

使用这些新的组件可以更好地完成我们之前需要自己去实现的 **非业务操作**，例如：管理 Activity 的生命周期 (Lifecycle) 、视图层和逻辑层的解耦 (MVVM，ViewModel + LiveData) 、数据库交互 (Room) 等等

我们在实现这些 **非业务操作** 时，往往都是八仙过海，各显神通，因为没有规范，代码就得不到统一，容易出现冲突的情况。而 jetpack 相当于制定了这样的一种规范

我认为 jetpack 是一定要使用的，因为这是 Android 将来的大方向；但不是 jetpack 中的每一个组件都要使用，因为就目前而言，jetpack 中的组件大多数都还没有开发完全，仍处于测试的版本

### 你了解 MVVM 架构吗？简单说一下如何实现？

MVVM 架构如下图所示

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9f1fc5a47d584e4ca78fda24565cc160~tplv-k3u1fbpfcp-watermark.awebp)

- Model 层：业务相关的数据 (如网络请求数据、本地数据库数据等) 及其对数据的处理
- View 层：页面视图 (Activity / Fragment)，负责接收用户输入、发起数据请求及展示结果页面
- ViewModel 层：M 与 V 之间的桥梁，负责业务逻辑

MVVM 架构的特点在于：ViewModel 不会持有 View 层的引用，而是 View 层会通过观察者模式监听 ViewModel 层的数据变化；当有新数据时，View 层能自动收到新数据并刷新界面

为了实现上面的 MVVM 架构模式，Jetpack 提供了多种组件来实现，具体来说有 Lifecycle、LiveData、ViewModel 等

Lifecycle 负责生命周期相关

LiveData 赋予类可观察，同时还是生命周期感知的 (其内部使用了 Lifecycle)

ViewModel 旨在以注重生命周期的方式存储和管理界面相关的数据

实现举例：

#### View 层

```java
public class MainActivity extends AppCompatActivity
{
    ActivityMainBinding mBinding;
    MainViewModel mViewModel;

    final List<Student> students = new ArrayList<>();

    @Override
    protected void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        mBinding = ActivityMainBinding.inflate(getLayoutInflater());
        setContentView(mBinding.getRoot());
        init();
    }

    private void init()
    {
        // 创建 ViewModel
        mViewModel = new ViewModelProvider(this, 
				new ViewModelProvider.NewInstanceFactory()).get(MainViewModel.class);
        // 观察 ViewModel 中的 LiveData，如果 LiveData 有变化就会回调
        mViewModel.data.observe(this, students :: addAll);
    }

    @Override
    protected void onResume()
    {
        super.onResume();
        // 调用 ViewModel 中的函数，使 ViewModel 请求 Model 获取数据
        mViewModel.getData();
    }
}
```

#### ViewModel 层

```java
public class MainViewModel extends ViewModel
{
    // LiveData
    public MutableLiveData<List<Student>> data = new MutableLiveData<>();

    // Model
    MainModel model = MainModel.getInstance();

    public void getData()
    {
        // 请求 Model 获取数据
        List<Student> result = model.findAll();
        // 将返回的数据放入 LiveData 中，通知 View 更新 UI
        data.setValue(result);
    }
}
```

#### Model 层

```java
public class MainModel
{
    // 获取 Dao 实例
    StudentDao studentDao = StudentDatabase.getInstance().getStudentDao();

    public List<Student> findAll()
    {
        // 从 Room 数据库中查找数据并返回
        return studentDao.findAll();
    }
}
```

### Jetpack 架构组件和 MVVM 架构的关系？

MVVM 架构是一种开发模式，而 Jetpack 架构组件是用于实现 MVVM 架构的工具

### ViewModel 是怎么做到在 Activity 被销毁重建新实例之后还能保持不变的呢？

当 Activity 被销毁重建后，会重走一遍生命周期函数，从 `onCreate()` 函数开始

通常我们会在 `onCreate()` 函数中获取 ViewModel，Google 也是推荐这么做的

```java
public class MainActivity
{
    MainViewModel mViewModel;
    
    @Override
    protected void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        mBinding = ActivityMainBinding.inflate(getLayoutInflater());
        setContentView(mBinding.getRoot());

        // 获取 ViewModel
        mViewModel = new ViewModelProvider(this, BaseApplication.sViewModelFactory)
										   .get(MainViewModel.class);
    }
}
```

#### ViewModelProvider.get()

```java
public <T extends ViewModel> T get(@NonNull Class<T> modelClass)
{
    // 获取 Class 对象的类名
	String canonicalName = modelClass.getCanonicalName();
	if (canonicalName == null)
    {
		throw new IllegalArgumentException(
            	"Local and anonymous classes can not be ViewModels");
	}
	return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
}

public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass)
{
    // 从 ViewModelStore 中搜索 ViewModel 是否已被创建过
	ViewModel viewModel = mViewModelStore.get(key);

    // 将搜索结果与传入的 Class 对象进行比对，
    // 如果一致就返回 ViewModelStore 中保存的 ViewModel 对象
	if (modelClass.isInstance(viewModel))
    {
		return (T) viewModel;
	}
    else
    {
		if (viewModel != null) { }
	}

    // 如果 ViewModelStore 中搜索不到，
    // 就通过 ViewModelProvider.Factory 创建一个 ViewModel 对象
	if (mFactory instanceof KeyedFactory)
    {
		viewModel = ((KeyedFactory) (mFactory)).create(key, modelClass);
	}
    else
    {
		viewModel = (mFactory).create(modelClass);
	}

    // 将新创建的 ViewModel 对象存入 ViewModelStore
	mViewModelStore.put(key, viewModel);

    // 返回新创建的 ViewModel 对象
	return (T) viewModel;
}
```

ViewModelStore 本质上就是一个 HashMap，以 ViewModel 的实现类名为 key，ViewModel 对象为 value，缓存所有创建过的 ViewModel

那 ViewModelStore 是什么时候被创建的呢？

#### new ViewModelProvider()

```java
public ViewModelProvider(@NonNull ViewModelStoreOwner owner, @NonNull Factory factory)
{
	this(owner.getViewModelStore(), factory);
}
```

owrer 就是我们传入的 Activity / Fragment 上下文，以 Activity 为例

```java
public ViewModelStore getViewModelStore()
{
	......

	// 如果 ViewModelStore 没有被创建
	if (mViewModelStore == null)
    {
        // 获取 NonConfigurationInstances，还原 ViewModelStore
		NonConfigurationInstances nc =
				(NonConfigurationInstances) getLastNonConfigurationInstance();
		if (nc != null)
        {
            mViewModelStore = nc.viewModelStore;
		}
        // 如果 ViewModelStore 没有被保存就重新创建
		if (mViewModelStore == null)
        {
			mViewModelStore = new ViewModelStore();
		}
	}
    // 返回 ViewModelStore 对象
	return mViewModelStore;
}

// frameworks/base/core/java/android/app/Activity.java
public Object getLastNonConfigurationInstance()
{
	return mLastNonConfigurationInstances != null
			? mLastNonConfigurationInstances.activity : null;
}
```

既然 ViewModelStore 通过 NonConfigurationInstances 保存，那 NonConfigurationInstances 是什么时候被创建的？

#### ActivityThread.handleDestroyActivity()

在 Activity 被销毁时，AMS 会通知 APP 进程并回调 `ActivityThread.handleDestroyActivity()` 函数

```java
// frameworks/base/core/java/android/app/ActivityThread.java
public void handleDestroyActivity(IBinder token, boolean finishing, 
                                  int configChanges, boolean getNonConfigInstance, 
                                  String reason
){
    ......

	ActivityClientRecord r = performDestroyActivity(token, finishing,
                configChanges, getNonConfigInstance, reason);
    
    ......
}

ActivityClientRecord performDestroyActivity(IBinder token, boolean finishing, 
                                            int configChanges, 
                                            boolean getNonConfigInstance, // 默认为 true
                                            String reason
){
    ......

	if (r != null)
    {
        ......

        // getNonConfigInstance 默认为 true，if 命中
		if (getNonConfigInstance)
        {
			try
            {
                // 执行 Activity 的 retainNonConfigurationInstances 函数
                // 创建 NonConfigurationInstances
				r.lastNonConfigurationInstances
                            = r.activity.retainNonConfigurationInstances();
			}
            catch (Exception e)
            {
				if (!mInstrumentation.onException(r.activity, e))
                {
					throw new RuntimeException(
                                "Unable to retain activity "
                                + r.intent.getComponent().toShortString()
                                + ": " + e.toString(), e);
				}
			}
		}
        
        ......
    }
    
    ......
}

// frameworks/base/core/java/android/app/Activity.java
NonConfigurationInstances retainNonConfigurationInstances()
{
    // 调用 onRetainNonConfigurationInstance 函数
	Object activity = onRetainNonConfigurationInstance();

    ......

	// 创建 NonConfigurationInstances
	NonConfigurationInstances nci = new NonConfigurationInstances();
	nci.activity = activity;

    ......

	return nci;
}

public final Object onRetainNonConfigurationInstance()
{
	......

	ViewModelStore viewModelStore = mViewModelStore;

	......

	// 创建 NonConfigurationInstances
	NonConfigurationInstances nci = new NonConfigurationInstances();
    // 将 ViewModelStore 保存在 NonConfigurationInstances 中
	nci.viewModelStore = viewModelStore;
	return nci;
}
```

#### ActivityThread.handleLaunchActivity()

当 Activity 重建后，AMS 会通知 APP 进程并回调 `ActivityThread.handleLaunchActivity()` 函数

```java
// frameworks/base/core/java/android/app/ActivityThread.java
public Activity handleLaunchActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions, Intent customIntent)
{
    ......

    final Activity a = performLaunchActivity(r, customIntent);

    ......
}

private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent)
{
    ......

	// 执行 Activity 的 attach 函数进行初始化
	activity.attach(appContext, this, getInstrumentation(), r.token,
			r.ident, app, r.intent, r.activityInfo, title, r.parent,
			r.embeddedID, r.lastNonConfigurationInstances, config,
			r.referrer, r.voiceInteractor, window, r.configCallback);
    
    ......
}

// frameworks/base/core/java/android/app/Activity.java
final void attach(Context context, 
				  ......
                  NonConfigurationInstances lastNonConfigurationInstances, 
                  ......
){
    ......

	// 对 mLastNonConfigurationInstances 字段赋值
	mLastNonConfigurationInstances = lastNonConfigurationInstances;

    ......
}
```

整体流程图：

![](https://note.youdao.com/yws/api/personal/file/WEB6aaea138bc4e4620e79466fdc305cb07?method=download&shareKey=e9571f6e4cfbfd5fac2b5aab5e432120)

所以 ViewModel 是通过 NonConfigurationInstances 进行保存的

在 Activity 销毁之前将 ViewModel 保存在 NonConfigurationInstances 中

当 Activity 重新创建时再将 ViewModel 从 NonConfigurationInstances 中取出

### LiveData 的 onChanged 回调时机？

当 LiveData 被设置观察者 (observe) 时，会立即检查 LiveData 中的数据是否发生过变化 (给 LiveData 设置了数据也算发生变化)

如果 LiveData 中的数据发生过变化，就会通知观察者 (Activity / Fragment)，回调 onChanged

之后 LiveData 中的数据一发生变化，就会通知观察者 (Activity / Fragment)，回调 onChanged

[Jetpack—LiveData组件的缺陷以及应对策略](https://juejin.cn/post/7054363342080573471)

### Room 使用的基本流程了解吗？

1. 添加依赖库

   ```groovy
   dependencies {
       def room_version = "2.3.0"
   
       implementation "androidx.room:room-runtime:$room_version"
       annotationProcessor "androidx.room:room-compiler:$room_version"
   }
   ```

2. 定义数据库表结构

   我们不需要手动去创建表，而是定义一个实体类并且使用 @Entity 注解

   这样 Room 就会根据类的字段去创建相应的数据库表，注意每个表必须都有一个主键

   例如：

   ```java
   @Entity
   public class User
   {
   	@PrimaryKey
   	public int uid;
   
   	@ColumnInfo(name = "first_name")
   	public String firstName;
   
   	@ColumnInfo(name = "last_name")
   	public String lastName;
   }
   ```

3. 创建 Dao

   Dao 是 Room 框架中的数据库访问接口，里面定义了多个函数，让用户能够以对象的方式实现对数据库表的增删改查

   在 Room 中定义一个 Dao 异常简单，只需要声明一个接口，然后使用 @Dao 注解即可

   例如：

   ```java
   @Dao
   public interface UserDao
   {
   	@Query("SELECT * FROM user")
   	List<User> getAll();
   
   	@Query("SELECT * FROM user WHERE uid IN (:userIds)")
   	List<User> loadAllByIds(int[] userIds);
   
   	@Query("SELECT * FROM user WHERE first_name LIKE :first AND " +
   			"last_name LIKE :last LIMIT 1")
   	User findByName(String first, String last);
   
   	@Insert
   	void insertAll(User... users);
   
   	@Delete
   	void delete(User user);
   }
   ```

4. 创建数据库

   数据库实例需要继承 RoomDatabase，并且声明成抽象类，以及获取 Dao 的几个抽象方法

   数据库对象需要使用 @Database 注解，并且声明参数 version 和 entities 

   version 代表的是数据库的版本号，每次数据库表有改动的时候，都需要增加版本号并且添加对应的 Migration

   entities 是一个数组，内容是每个表所对应的实体类

   例如：

   ```java
   @Database(entities = {Student.class}, version = 1)
   public abstract class AppDatabase extends RoomDatabase
   {
       public abstract UserDao userDao();
   
       private static volatile AppDatabase mInstance;
   
       public static AppDatabase getInstance()
       {
           if (mInstance == null)
           {
               synchronized (AppDatabase.class)
               {
                   if (mInstance == null)
                   {
                       mInstance = Room.databaseBuilder(getApplicationContext(),
                               AppDatabase.class, "database-name")
                               .build();
                   }
               }
           }
           return mInstance;
       }
   }
   ```

### 使用 Room 和直接使用 SQLite 有什么区别？

Room 本身只是对 SQLite 在上层进行了一层封装，实际上使用 Room 还是在使用 SQLite

Room 对 SQLite 的基本操作进行了抽象，使得操作 SQLite 变得更加简便，这样我们就无需花费精力在操作 SQLite 上，更专注于实现业务逻辑

同时 Room 支持 Kotlin 协程、Flow 和 LiveData 相结合，使得使用更加方便

##  OkHttp

### 请求流程？

使用 OkHttp 发起一次请求时，会创建三个角色：

* OkHttpClient：OkHttp 框架的使用接口，其内部做了大量的复杂逻辑处理，隐藏了整体框架的复杂性（外观设计模式）
* Request：对于 Http 请求的封装，记录 Http 请求的 URL、请求类型、请求头、请求体等信息
* Call：对于 Request 的再一层封装，表示 Request 请求已准备好执行。具体实现为 RealCall，其内部处理了分发器和拦截器之间的工作逻辑
  * Call 的 `execute()` 函数代表了 OkHttp 的同步请求模式，在该模式下 OkHttp 框架会立即执行该请求，阻塞当前执行操作的线程直至返回 Response，**无法并发执行请求 (需要外部自行处理并发)**
  * Call 的 `enqueue(Callback responseCallback)` 函数代表了 OkHttp 的异步请求模式，在该模式下 OkHttp 框架会将该请求提交给其内部维护的线程池来执行，不会阻塞当前执行操作的线程，**可以并发执行请求 (无需外部处理并发)**

请求总体流程如下图：

![](https://note.youdao.com/yws/api/personal/file/2CA541D9990E45A7BE36BBC96C544C02?method=download&shareKey=c1e5873a846506d5ec506926ffdbe200)

###  讲解一下 OkHttp 中的分发器？

Dispatcher (分发器) 用于调配请求任务，其内部维护了一个线程池，用于 OkHttp 的异步请求模式。我们可以在构建 OkHttpClient 时，通过传递我们自己创建的 Dispatcher 对象来自定义线程池

Dispatcher 中定义的成员属性：

```java
// runningAsyncCalls 大小的最大值
private int maxRequests = 64;
// runningAsyncCalls 中域名相同请求的最大值
private int maxRequestsPerHost = 5;
// 闲置任务（当所有请求执行完成后执行，由使用者设置）
private @Nullable Runnable idleCallback;

// 异步请求使用的线程池
private @Nullable ExecutorService executorService;

// 异步请求等待执行队列
private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

// 异步请求正在执行队列
private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

// 同步请求正在执行队列
private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
```

#### 同步请求

Dispatcher 对于同步请求的整体工作流程如下图：

![](https://note.youdao.com/yws/api/personal/file/WEB13f62edbdd770cfa9551cc2571bb52ed?method=download&shareKey=ca0bbafc91ca6b0232ca4fc31714af8a)

由于同步请求没有使用到 Dispatcher 内部的线程池，一次只能执行一个，即 runningSyncCalls 队列中至多存在一个请求，因此当同步请求返回 response 后整个流程就结束了

如果我们要使用 OkHttp 的同步请求模式来执行并发操作，我们需要在外部自行定义线程池去执行 OkHttp 的同步请求

#### 异步请求

Dispatcher 对于异步请求的整体工作流程如下图：

![](https://note.youdao.com/yws/api/personal/file/WEBaa837f19ac776ae18a5e46c59f942d2a?method=download&shareKey=1713e3f5211430728c129104cb7614f6)

Dispatcher 使用了两个阻塞队列 (runningAsyncCalls、readyAsyncCalls) 来管理异步请求，并使用了线程池来执行异步请求，实现了并发执行异步请求

runningAsyncCalls 和 readyAsyncCalls 主要用于记录所有提交的异步请求，基于发送请求的客户端和接受请求的服务端两者的性能考虑，并不是所有提交的异步请求都能被放入 runningAsyncCalls 立即被提交给线程池执行

只有 **当 runningAsyncCalls 的大小没有超过 maxRequests (64) 且 runningAsyncCalls 中与当前请求域名相同的请求数量没有超过 maxRequestsPerHost (5)** 时，异步请求才能被放入 runningAsyncCalls 立即执行；否则异步请求被放入 readyAsyncCalls 中等待执行

对于 runningAsyncCalls 的大小不能超过 maxRequests 的限制，**是基于客户端性能考虑的**，runningAsyncCalls 中有多少个异步请求，就意味着 APP 进程需要开启多少个线程去执行请求；一个 APP 进程所能开启的线程数量是有限的，开启的线程太多会导致占用大量应用进程资源，造成 APP 运行卡顿甚至崩溃

OkHttp 基于各大浏览器的内核源码，综合确定了 maxRequests = 64

对于 runningAsyncCalls 中与当前请求域名相同的请求数量不能超过 maxRequestsPerHost 的限制，**是基于服务端性能考虑的**，通常一个服务器拥有多个端口，我们的应用只占用了其中一个端口来发送请求；对于服务器而言，如果仅仅是一个端口就同时收到数十个请求，那么服务器肯定会因为资源紧张而崩溃的

OkHttp 基于市面上常见服务器的性能指标，综合确定了 maxRequestsPerHost = 5

##### 最大并发线程池

如果我们在构建 OkHttpClient 时，没有传递自定义的线程池，那么 Dispatcher 会为我们创建一个默认的最大并发线程池

```java
public synchronized ExecutorService executorService()
{
	if (executorService == null)
    {
		executorService = new ThreadPoolExecutor(
            0, // corePoolSize：核心线程数
            Integer.MAX_VALUE, // maximumPoolSize：最大线程数
            60, // keepAliveTime：普通线程空闲时的存活时间
            TimeUnit.SECONDS, // TimeUnit：keepAliveTime的时间单位
			new SynchronousQueue<Runnable>(), // workQueue：任务等待队列
            Util.threadFactory("OkHttp Dispatcher",false)); // threadFactory：线程创建工厂
	}
	return executorService;
}
```

我们先回顾一下 Java 线程池的工作原理，如下图：

![](https://note.youdao.com/yws/api/personal/file/WEB9e14bdcf6385ee1fce9d3f0e160327fe?method=download&shareKey=b5ecf83b5b6a655195db9e4be3fa4740)

根据 Dispatcher 创建线程池传入的参数，我们可以看出这个线程池具有高并发、最大吞吐量的特点

首先核心线程数为 0，表示该线程池不会缓存任何线程，该线程池中所有线程都是在 60s 内没有工作就会被回收

SynchronousQueue 是一个不存储元素的阻塞队列，因此最大线程数 Integer.MAX_VALUE 与任务等待队列 SynchronousQueue 的组合就能得到最大并发量

即当该线程池执行任务时，如果不存在空闲线程，任务不需要等待，线程池会马上新建线程执行任务

> 向该线程池提交任务时，因为核心线程数为 0，所以任务会由池中的空闲线程执行或被添加到 SynchronousQueue；
>
> 由于 SynchronousQueue 不存储元素的特性，当任务被添加到 SynchronousQueue 时，任务不会被放入等待队列，因此线程池就会检查当前池中线程数是否超过 maximumPoolSize；
>
> 由于设置了 maximumPoolSize = Integer.MAX_VALUE，池中线程数不可能超过 maximumPoolSize，因此线程池就会新建线程执行当前提交的任务

Dispatcher 通过这样的参数设置得到了最大并发线程池，但是该线程池的最大并发数是无上限的，新建线程过多可能会导致我们的 APP 应用出现 OOM

由于每一个任务被放入 runningAsyncCalls 后会立即被提交给线程池执行，所以 OkHttp 在任务提交时添加了一个限制：runningAsyncCalls 的大小不能超过 maxRequests (64)，这样既解决了这个问题同时也获得了最大并发

###  讲解一下 OkHttp 中的线程池？任务执行的最大并发数？

OkHttp 中会默认创建一个[最大并发线程池](#最大并发线程池) 来执行异步请求，当该线程池中没有空闲线程时，对于新提交的任务会立即新建线程来执行，从而做到最大并发

OkHttp 默认创建的最大并发线程池其最大并发数是没有上限的，但是 APP 应用进程所能创建的线程数量是有限的

由于每一个任务被放入 runningAsyncCalls 后会立即被提交给线程池执行，所以 OkHttp 在任务提交时添加了一个限制：runningAsyncCalls 的大小不能超过 maxRequests (64)，这样既解决了这个问题同时也获得了最大并发

### 为什么 OkHttp 限制对于每个 Host 至多只能有五个请求？

**这是基于服务端性能考虑的**，通常一个服务器拥有多个端口，我们的应用只占用了其中一个端口来发送请求；对于服务器而言，如果仅仅是一个端口就同时收到数十个请求，那么服务器肯定会因为资源紧张而崩溃的

OkHttp 基于市面上常见服务器的性能指标，将对同一域名正在执行请求的最大个数限制为 5

### OkHttp 中的任务有没有优先级？

OkHttp 中的同步请求和被放入 runningAsyncCalls 的异步请求都是立即被执行的，不存在优先级的说法

当 runningAsyncCalls 满足了 OkHttp 设置的限制后，异步请求会被放入 readyAsyncCalls 中等待执行

readyAsyncCalls 本质上只是一个 ArrayDeque 阻塞队列，不具备优先级的功能

OkHttp 提供了一个闲置任务的功能，这个闲置任务会在所有已提交请求执行完成后执行，我们可以通过 Dispatcher 向外公开的 API —— `setIdleCallback(Runnable idleCallback)` 进行设置

### 拦截器和责任链思想？

请求经过 Dispatcher 分发后，Dispatcher 会执行 `RealCall.getResponseWithInterceptorChain()` 函数，将请求移交给责任链处理

责任链思想的核心是 **解决一个流程中的先后执行关系**，例如发送 Http 请求的流程，OkHttp 通过责任链将拦截器代码与流程代码进行解耦，代码结构变得更加清晰，拦截器之间的先后执行关系更加明了

OkHttp 责任链的整体流程如下图：

![](https://note.youdao.com/yws/api/personal/file/WEB7f4d14a87066a646139020080a7b7dbf?method=download&shareKey=5dce35f127a6294c45e3c3ef173af8f0)

OkHttp 中有五大默认拦截器：

* RetryAndFollowUpInterceptor：判断是否需要重试与重定向
  * 重试与重定向执行的前提是出现 RouteException 或 IOException 这两类异常，否则不会发生重试与重定向
* BridgeInterceptor：添加或删除 Request 的相关头部信息，使 Request 符合网络请求规范，能够进行网络请求
* CacheInterceptor：发起请求前查询缓存，根据缓存策略判断缓存是否可用 (仅对于 GET 请求)
  * 如果缓存可用，直接返回缓存中的 response 结果，中断责任链
  * 如果缓存不可用，将请求移交给下一个拦截器执行，并在 response 返回后进行缓存
* ConnectInterceptor ：与服务器完成 Socket 连接
* CallServerInterceptor：与服务器通信，发送请求，封装请求数据与解析响应数据 (如：HTTP 报文等)

##  性能优化

### 在开发中如何定位应用的卡顿问题？

大多数用户感知到的卡顿等性能问题的最主要根源都是因为 **渲染卡顿**

Android 系统每隔大概 16.6 ms 发出 VSYNC (垂直同步) 信号，触发对于 UI 的渲染，如果每次渲染都成功，这样就能够达到流畅画面所需要的 60 fps

一旦程序的每帧运行时间超出了 16 ms，也就是帧率低于 60 fps，我们称之为丢帧现象

出现丢帧现象，就会造成肉眼可见的卡顿

#### Systrace

[在命令行上捕获系统跟踪记录](https://developer.android.google.cn/topic/performance/tracing/command-line)

[Systrace工具使用](https://www.jianshu.com/p/e73768e66b8d)

Systrace 是 Android 平台提供的一款工具，用于记录短期内的设备活动情况。该工具会生成一份报告，其中汇总了 Android 内核中的数据，例如 CPU 调度程序、磁盘活动和应用线程等占用情况

Systrace 对于应用开发者来说，能看的并不多。主要用于看是否丢帧，以及丢帧时系统以及我们应用大致的一个状态

执行 Systrace 可以选择配置自己感兴趣的 category，常用的有：

| 标签   | 描述                                                         |
| ------ | ------------------------------------------------------------ |
| gfx    | Graphics 图形系统，包括 SerfaceFlinger，VSYNC 消息，Texture，RenderThread 等 |
| input  | Input 输入系统，按键或者触摸屏输入；分析滑动卡顿等           |
| view   | View 绘制系统的相关信息，比如 onMeasure，onLayout 等；分析 View 绘制性能 |
| am     | ActivityManager 调用的相关信息；分析 Activity 的启动、跳转   |
| dalvik | 虚拟机相关信息；分析虚拟机行为，如 GC 停顿                   |
| sched  | CPU 调度的信息，能看到 CPU 在每个时间段在运行什么线程，线程调度情况，锁信息。 |
| dick   | IO 信息                                                      |
| wm     | WindowManager 的相关信息                                     |
| res    | 资源加载的相关信息                                           |

```
python systrace.py -t 5 -o \路径\a.html gfx input view am dalvik sched wm disk res -a 应用包名
```

-t：记录设备活动的时间

-o：报告的输出路径

**我们在使用 Systrace 获取报告时，切记不要抓取太长时间，也不要进行太多复杂的操作**

在生成的 html 文件中，我们需要关注 Frames 这一栏的内容

在 Frames 这一栏中，将每一帧的运行情况使用颜色来进行标识，绿色表示该帧运行正常，黄色表示该帧运行稍卡顿，红色表示该帧运行卡顿严重

如果只是单独存在一个红色或者黄色的都是没关系的。关键是连续的红 / 黄色或者两帧间隔非常大那就需要我们去仔细观察

在 UIThread (主线程) 上面有一条很细的线，表示线程状态

Systrace 会用不同的颜色来标识不同的线程状态，在每个函数上面都会有对应的线程状态来标识目前线程所处的状态

通过查看线程状态我们可以知道目前的瓶颈是什么，是 CPU 执行慢还是因为 Binder 调用，又或是进行 IO 操作，又或是拿不到 CPU 时间片

线程状态主要有以下几种：

* 绿色：表示正在运行
  * 需要考虑是否频率不够？(CPU处理速度)
  * 需要考虑是否运行在小核上？(通常来说不可控，但其实很多手机都会有游戏模式，如果我们的应用是手游，那系统会优先把手游中的任务放到大核上运行)
* 蓝色：表示可以运行，但是 CPU 在执行其他线程
  * 需要考虑是否后台有太多的任务要运行？Runnable 状态的线程状态持续时间越长，则表示 cpu 的调度越忙，没有及时处理到这个任务
  * 需要考虑没有及时处理是因为频率太低？
* 紫色：表示阻塞，一般表示执行 IO 操作
* 白色：表示休眠
  * 需要考虑可能是因为线程在互斥锁上被阻塞 ，如 Binder 堵塞 / Sleep / Wait 等

#### [Android CPU Profiler](https://developer.android.google.cn/studio/profile/cpu-profiler)

我们可以使用 CPU 性能剖析器在与应用交互时实时检查应用的 CPU 使用率和线程活动，也可以检查记录的方法跟踪数据、函数跟踪数据和系统跟踪数据的详情

### 线上如何监控卡顿问题？

目前市面上比较成熟的监控线上卡顿的方案有两种：

* 利用 UI 线程的 Looper 打印的日志匹配
* 设置 Choreographer.FrameCallback

第一种方式比较适合在发布前进行测试或者小范围灰度测试然后定位问题

第二种方式比较适合监控线上环境的 app 的掉帧情况来计算 app 在某些场景的流畅度然后有针对性的做性能优化

#### 利用 UI 线程的 Looper 打印的日志匹配 (BlockCanary)

更新 UI 本质上还是通过 UI 线程中的 Looper 进行驱动的，如果在 handler 的 `dispatchMesaage` 函数里有耗时操作，就会发生卡顿

只要检测 `msg.target.dispatchMessage(msg)` 的执行时间，就能检测到部分 UI 线程是否有耗时的操作

在 `Looper.loop()` 函数中，有一个日志系统

```java
// Looper.java
public static void loop()
{ 
	...... 
	for (;;)
	{ 
		......
		Printer logging = me.mLogging;
		if (logging != null)
		{
			logging.println(">>>>> Dispatching to " + msg.target + " " + 
					msg.callback + ": " + msg.what);
		}
		
		msg.target.dispatchMessage(msg);
		
		if (logging != null)
		{
			logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
		}
		...... 
	}
}
```

注意到 `msg.target.dispatchMessage(msg)` 这行代码的执行前后，先后执行了两次 `Printer.println()` 函数

因此我们可以通过在 `Printer.println()` 函数中获取系统时间来计算时间差值，来得到 `msg.target.dispatchMessage(msg)` 的执行时间，从而设置阈值判断是否发生了卡顿

Looper 提供了 `setMessageLogging(Printer printer)` 函数来让我们设置自定义的 Printer

这也是 [BlockCanary](https://github.com/markzhai/AndroidPerformanceMonitor) 的核心原理

#### 设置 Choreographer.FrameCallback

Android 系统每隔 16ms 发出 VSYNC 信号，来通知界面进行重绘、渲染，每一次同步的周期约为 16.6 ms，代表一帧的刷新频率

通过 Choreographer 类设置它的 `FrameCallback()` 函数，当每一帧被渲染时会触发回调 `FrameCallback.doFrame(long frameTimeNanos)` 函数

frameTimeNanos 是底层 VSYNC 信号到达的时间戳

```java
public class ChoreographerHelper
{
    public static void start()
    {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN)
        { Choreographer.getInstance().postFrameCallback(new Choreographer.FrameCallback() 
			{
                long lastFrameTimeNanos = 0;
                @Override public void doFrame(long frameTimeNanos)
                {
                    //上次回调时间
                    if (lastFrameTimeNanos == 0)
                    {
                        lastFrameTimeNanos = frameTimeNanos; 
                        Choreographer.getInstance().postFrameCallback(this);
                        return;
                    }
                    long diff = (frameTimeNanos - lastFrameTimeNanos) / 1_000_000;
                    if (diff > 16.6f)
                    {
                        //掉帧数 
                        int droppedCount = (int) (diff / 16.6);
                    }
                    lastFrameTimeNanos = frameTimeNanos; 
                    Choreographer.getInstance().postFrameCallback(this); 
                } 
            }); 
        } 
    } 
}
```

通过 ChoreographerHelper 可以实时计算帧率和掉帧数，实时监测 App 页面的帧率数据，发现帧率过低，还可以自动保存现场堆栈信息

### 如何解决卡顿问题？

一般来说解决卡顿就是要优化 UI 渲染所要花费的时间，大体上有这三个方向：

* 布局层级优化
* 减少过度渲染
* 布局加载优化 (异步加载)

#### 布局层级优化

measure、layout、draw 这三个过程需要自顶向下地遍历 ViewTree，如果视图层级太深自然需要更多的时间来完成整个绘测过程，从而造成启动速度慢、卡顿等问题

##### Layout Inspector

我们可以使用 Layout Inspector 这个工具来帮助我们检查应用运行时的视图层次结构

![](https://note.youdao.com/yws/api/personal/file/1FDC61FFDA9E46D0BD2636DE79D68C34?method=download&shareKey=72e696b8529853cd7652089777c8ff6d)

我们应该尽量减少其层级，可以使用 ConstraintLayout 来使得布局尽量扁平化，移除非必需的UI组件

##### 使用 merge / include 标签

当我们有一些布局元素需要被多处使用时，这时候我们会考虑将其抽取成一个单独的布局文件。在需要使用的地方通过 include 加载

也可以使用 merge 标签将一些控件直接添加到布局中

##### 使用 ViewStub 标签

当我们布局中存在一个 View / ViewGroup，在某个特定时刻才需要他的展示时，我们通常的做法是把这个元素在 xml 中定义为 invisible / gone，在需要显示时再设置为 visible 可见

但如果只是将这个元素定义为 invisible / gone，在加载布局时，它仍然会被初始化，占用资源

ViewStub 是一个轻量级的 view，它不可见，不用占用资源，只有设置 ViewStub 为 visible 或调用其 `inflater()` 函数时，其对应的布局文件才会被初始化

因此我们可以通过使用 ViewStub 标签来包裹此类 View / ViewGroup，避免此类 View / ViewGroup 不显示却又需要加载，造成资源的浪费

#### [减少过度渲染](https://developer.android.google.cn/topic/performance/rendering/overdraw?hl=zh_cn)

过度绘制是指系统在渲染单个帧的过程中多次在屏幕上绘制某一个像素

例如，如果我们有若干界面卡片堆叠在一起，每张卡片都会遮盖其下面一张卡片的部分内容。但是，系统仍需要绘制其中被遮盖的部分

##### 检查过度渲染

手机开发者选项中能够显示过度渲染检查功能，通过对界面进行彩色编码来帮我们识别过度绘制

开启步骤如下：

1. 进入手机的 **开发者选项 (Developer Options)**
2. 找到 **调试 GPU 过度绘制 (Debug GPU overdraw)**
3. 在弹出的对话框中，选择 **显示过度绘制区域 (Show overdraw areas)**

Android 将按如下方式为界面元素着色，以确定过度绘制的次数：

- **真彩色**：没有过度绘制
- ![img](https://developer.android.google.cn/topic/performance/images/gpu/overdraw-blue.png?hl=zh_cn) **蓝色**：过度绘制 1 次
- ![img](https://developer.android.google.cn/topic/performance/images/gpu/overdraw-green.png?hl=zh_cn) **绿色**：过度绘制 2 次
- ![img](https://developer.android.google.cn/topic/performance/images/gpu/overdraw-pink.png?hl=zh_cn) **粉色**：过度绘制 3 次
- ![img](https://developer.android.google.cn/topic/performance/images/gpu/overdraw-red.png?hl=zh_cn) **红色**：过度绘制 4 次或更多次

> 需要注意，这些颜色是半透明的，因此您在屏幕上看到的确切颜色取决于界面内容
>
> 有些过度绘制是不可避免的
>
> 因此在优化应用的界面时，应尽可能尝试达到大部分显示真彩色或仅有 1 次过度绘制（蓝色）的视觉效果

##### 解决过度渲染

我们可以采取以下几种策略来减少甚至消除过度绘制：

- 移除布局中不需要的背景
  
  - 默认情况下，布局没有背景，这表示布局本身不会直接渲染任何内容
  
    但是，当布局具有背景时，其有可能会导致过度绘制
  
    移除不必要的背景可以快速提高渲染性能
  
    不必要的背景可能永远不可见，因为它会被应用在该视图上绘制的任何其他内容完全覆盖
  
    例如，当系统在父视图上绘制子视图时，可能会完全覆盖父视图的背景
- 使视图层次结构扁平化
  
  - 可以通过优化视图层次结构来减少重叠界面对象的数量，从而提高性能
- 降低透明度
  
  - 对于不透明的 view ，只需要渲染一次即可把它显示出来
  
    但是如果这个 view 设置了 alpha 值，则至少需要渲染两次
  
    这是因为使用了 alpha 的 view 需要先知道混合 view 的下一层元素是什么，然后再结合上层的 view 进行 Blend 混色处理
  
    透明动画、淡入淡出和阴影等效果都涉及到某种透明度，这就会造成了过度绘制
  
    可以通过减少要渲染的透明对象的数量，来改善这些情况下的过度绘制
  
    例如，如需获得灰色文本，可以在 TextView 中绘制黑色文本，再为其设置半透明的透明度值
  
    但是，简单地通过用灰色绘制文本也能获得同样的效果，而且能够大幅提升性能

#### 布局加载优化（异步加载）

LayoutInflater 加载 xml 布局的过程会在主线程使用 IO 读取 XML 布局文件进行 XML 解析，再根据解析结果利用反射创建布局中的控件对象。这个过程随着布局的复杂度上升，耗时自然也会随之增大

因此对于布局加载，我们是否可以将其异步执行呢？

##### [AsyncLayoutInflater](https://developer.android.google.cn/jetpack/androidx/releases/asynclayoutinflater?hl=zh_cn)

Android 为我们提供了 Asynclayoutinflater 把耗时的加载操作在异步线程中完成，最后把加载结果再回调给主线程

```java
dependencies { 
    implementation "androidx.asynclayoutinflater:asynclayoutinflater:1.0.0" 
}

new AsyncLayoutInflater(this)
    .inflate(R.layout.activity_main, null, new AsyncLayoutInflater.OnInflateFinishedListener() 
	{ 
        @Override 
        public void onInflateFinished(@NonNull View view, int resid, @Nullable ViewGroup parent) 
        { 
            setContentView(view); 
            ......
        } 
    });
```

使用 AsyncLayoutInflater，我们还需要注意以下几点：

* 使用异步 inflate，那么需要保证这个布局中 parent 的 generateLayoutParams 函数是线程安全的

* 所有构建的 view 中必须不能创建 Handler 或者是调用 `Looper.myLooper()` 函数（因为是在异步线程中加载的，异步线程默认没有调用 `Looper.prepare()` 函数进行初始化）

* 不支持设置 LayoutInflater.Factory 或 LayoutInflater.Factory2

* 不支持加载包含 Fragment 的布局

* 如果 AsyncLayoutInflater 加载失败，那么会自动回退到 UI 线程来加载布局

##### [X2C](https://github.com/iReaderAndroid/X2C/blob/master/README_CN.md)

### 如何实现 Crash 的监控？

Cras (应用崩溃) 是由于代码异常而导致 App 非正常退出，导致应用程序无法继续使用，所有工作都停止的现象

发生 Crash 后需要重新启动应用 (有些情况会自动重启)，而且不管应用在开发阶段做得多么优秀，也无法避免 Crash 发生，特别是在 Android 系统中，系统碎片化严重、各 ROM 之间的差异，甚至系统 Bug，都可能会导致 Crash 的发生

在 Android 应用中发生的 Crash 有两种类型，Java 层 Crash 和 Native 层 Crash

这两种 Crash 的监控和获取堆栈信息有所不同

#### Java 层 Crash

Java 层 Crash 监控非常简单，Java 中的 Thread 定义了一个接口 —— UncaughtExceptionHandler，用于处理未捕获的异常导致线程的终止

**注意：已经被 catch 了的是捕获不到的**

当我们的应用发生 crash 的时候，就会走 `UncaughtExceptionHandler.uncaughtException()` 函数，在该函数中可以获取到异常的信息，我们通过 `Thread.setDefaultUncaughtExceptionHandler()` 该函数来设置线程默认的异常处理器，我们可以将异常信息保存到本地或者是上传到服务器，方便我们快速的定位问题

```java
public class CrashHandler implements Thread.UncaughtExceptionHandler
{
    private static final String FILE_NAME_SUFFIX = ".trace";
    private static Thread.UncaughtExceptionHandler mDefaultCrashHandler;
    private static Context mContext;
    
    private CrashHandler(){ }
    
    public static void init(@NonNull Context context)
    { 
        // 默认为：RuntimeInit#KillApplicationHandler 
        mDefaultCrashHandler = Thread.getDefaultUncaughtExceptionHandler(); 
        Thread.setDefaultUncaughtExceptionHandler(this); 
        mContext = context.getApplicationContext(); 
    }
    
    /**
     * 当程序中有未被捕获的异常，系统将会调用这个方法 
     *
     * @param t 出现未捕获异常的线程 
     * @param e 得到异常信息 
     */ 
    @Override public void uncaughtException(Thread t, Throwable e)
    { 
        try
        {
            // 自行处理：保存本地 
            File file = dealException(e); 
            // 上传服务器 
            //...... 
        }
        catch (Exception e1)
        { 
            e1.printStackTrace(); 
        }
        finally
        { 
            // 交给系统默认程序处理
			if (mDefaultCrashHandler != null)
            { 
                mDefaultCrashHandler.uncaughtException(t, e); 
            } 
        } 
    }
    
    /**
     * 导出异常信息到SD卡 
     *
     * @param e 
     */ 
    private File dealException(Thread t,Throwable e) throw Exception
    { 
        String time = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date());
        File f = new File(context.getExternalCacheDir().getAbsoluteFile(),"crash_info"); 
        if (!f.exists())
        { 
            f.mkdirs();
        }
        File crashFile = new File(f, time + FILE_NAME_SUFFIX);
        File file = new File(PATH + File.separator + time + FILE_NAME_SUFFIX);
        // 往文件中写入数据
        PrintWriter pw = new PrintWriter(new BufferedWriter(new FileWriter(file))); 
        pw.println(time);
        pw.println("Thread: "+ t.getName());
        pw.println(getPhoneInfo());
        e.printStackTrace(pw); // 写入crash堆栈 
        pw.close();
        return file;
    }
    
    private String getPhoneInfo() throws PackageManager.NameNotFoundException
    {
        PackageManager pm = mContext.getPackageManager();
        PackageInfo pi = pm.getPackageInfo(mContext.getPackageName(), 
				PackageManager.GET_ACTIVITIES);
        StringBuilder sb = new StringBuilder(); 
        // App版本
        sb.append("App Version: ");
        sb.append(pi.versionName);
        sb.append("_");
        sb.append(pi.versionCode + "\n");
        // Android版本号
        sb.append("OS Version: ");
        sb.append(Build.VERSION.RELEASE);
        sb.append("_");
        sb.append(Build.VERSION.SDK_INT + "\n");
        // 手机制造商
        sb.append("Vendor: ");
        sb.append(Build.MANUFACTURER + "\n");
        // 手机型号
        sb.append("Model: ");
        sb.append(Build.MODEL + "\n");
        // CPU架构
        sb.append("CPU: ");
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP)
        {
            sb.append(Arrays.toString(Build.SUPPORTED_ABIS));
        }
        else
        {
            sb.append(Build.CPU_ABI);
        }
        return sb.toString();
    }
}
```

#### Native 层 Crash

实现 Native 层 Crash 的监控，我们需要借助 Linux 系统中的信号机制

信号机制是 Linux 进程间通信的一种重要方式，Linux 信号一方面用于正常的进程间通信和同步，另一方面它还负责监控系统异常及中断

当应用程序运行异常时，Linux 内核将产生错误信号并通知当前进程。当前进程在接收到该错误信号后，可以有三种不同的处理方式：

* 忽略该信号

* 捕捉该信号并执行对应的信号处理函数 (信号处理程序)

* 执行该信号的缺省操作 (如终止进程)

当 Linux 应用程序在执行时发生严重错误，一般会导致程序崩溃

其中，Linux 专门提供了一类 Crash 信号，在程序接收到此类信号时，缺省操作是将崩溃的现场信息记录到核心文件 (墓碑)，然后终止进程

常见崩溃信号列表：

| 信号    | 描述                           |
| ------- | ------------------------------ |
| SIGSEGV | 内存引用无效                   |
| SIGBUS  | 访问内存对象的未定义部分       |
| SIGFPE  | 算术运算错误，如：除以零       |
| SIGILL  | 非法指令，如执行垃圾或特权指令 |
| SIGSYS  | 系统调用错误                   |
| SIGXCPU | 超过 CPU 时间限制              |
| SIGXFSZ | 文件大小限制                   |

一般的，如果出现崩溃信号，Android 系统默认缺省操作是直接退出我们的程序

但是系统允许我们给某一个进程的某一个特定信号注册一个相应的处理函数 (signal)，即对该信号的默认处理动作进行修改

因此 Native 层 Crash 的监控可以采用这种信号机制，捕获崩溃信号执行我们自己的信号处理函数从而捕获 Native 层 Crash

这也是 BreakPad 实现监控 Native 层 Crash 的核心原理

##### [BreakPad](https://github.com/google/breakpad)

Google BreakPad 是一个跨平台的崩溃转储和分析框架和工具集合，BreakPad 在 Linux 中的实现就是借助了 Linux 信号捕获机制实现的

因为其实现为 C++，因此在 Android 中使用时，必须借助 NDK 工具

###### 引入 BreakPad

将 Breakpad 源码下载解压，首先查看 README.ANDROID 文件

> If you're using the ndk-build build system, you can follow
> these simple steps:
>
>   1/ Include android/google_breakpad/Android.mk from your own
>      project's Android.mk
>
>      This can be done either directly, or using ndk-build's
>      import-module feature.
>
>   2/ Link the library to one of your modules by using:
>
>      LOCAL_STATIC_LIBRARIES += breakpad_client
>
> NOTE: The client library requires a C++ STL implementation,
>       which you can select with APP_STL in your Application.mk
>
>       It has been tested succesfully with both STLport and GNU libstdc++

按照文档中的介绍，如果我们使用 Android.mk 文件就可以非常简单地把 BreakPad 引入到我们工程中，但是目前 NDK 默认的构建工具为：CMake，因此我们需要做一次移植

查看 android/google_breakpad/Android.mk

```makefile
LOCAL_PATH := $(call my-dir)/../..

include $(CLEAR_VARS)

# 最后编译出 libbreakpad_client.a
LOCAL_MODULE := breakpad_client
# 指定 c++ 源文件后缀名
LOCAL_CPP_EXTENSION := .cc

# 强制构建系统以 32 位 arm 模式生成模块的对象文件
LOCAL_ARM_MODE := arm

# 需要编译的源码 LOCAL_SRC_FILES := \
	src/client/linux/crash_generation/crash_generation_client.cc \ 
	src/client/linux/dump_writer_common/thread_info.cc \ 
	src/client/linux/dump_writer_common/ucontext_reader.cc \ 
	src/client/linux/handler/exception_handler.cc \ 
	src/client/linux/handler/minidump_descriptor.cc \ src/client/linux/log/log.cc \ 
	src/client/linux/microdump_writer/microdump_writer.cc \ 
	src/client/linux/minidump_writer/linux_dumper.cc \ 
	src/client/linux/minidump_writer/linux_ptrace_dumper.cc \ 
	src/client/linux/minidump_writer/minidump_writer.cc \ 
	src/client/minidump_file_writer.cc \ src/common/convert_UTF.cc \ src/common/md5.cc \ 
	src/common/string_conversion.cc \ src/common/linux/breakpad_getcontext.S \ 
	src/common/linux/elfutils.cc \ src/common/linux/file_id.cc \ 
	src/common/linux/guid_creator.cc \ src/common/linux/linux_libc_support.cc \ 
	src/common/linux/memory_mapped_file.cc \ src/common/linux/safe_readlink.cc

# 导入头文件
LOCAL_C_INCLUDES := $(LOCAL_PATH)/src/common/android/include \ 
					$(LOCAL_PATH)/src \ 
					$(LSS_PATH) #注意这个目录 

# 导出头文件
LOCAL_EXPORT_C_INCLUDES := $(LOCAL_C_INCLUDES)
# 使用 android ndk 中的日志库
log LOCAL_EXPORT_LDLIBS := -llog 

# 编译 static 静态库 -> 类似 java 的 jar 包
include $(BUILD_STATIC_LIBRARY)
```

对照 Android.mk 文件，我们在自己项目的 cpp (工程中 C / C++ 源码) 目录下创建 breakpad 目录，并将下载 

的 breakpad 源码根目录下的 src 目录全部复制到新建的 breakpad 目录下

接下来在 breakpad 目录下创建 CMakeList.txt 文件

```cmake
cmake_minimum_required(VERSION 3.4.1)

# 对应 android.mk 中的 LOCAL_C_INCLUDES
include_directories(src src/common/android/include)

# 开启 arm 汇编支持，因为在源码中有 .S 文件（汇编源码）
enable_language(ASM)

# 生成 libbreakpad.a 并指定源码，对应 android.mk 中的 LOCAL_SRC_FILES + LOCAL_MODULE
add_library(breakpad STATIC 
	src/client/linux/crash_generation/crash_generation_client.cc 
	src/client/linux/dump_writer_common/thread_info.cc 
	src/client/linux/dump_writer_common/ucontext_reader.cc 
	src/client/linux/handler/exception_handler.cc
	src/client/linux/handler/minidump_descriptor.cc 
	src/client/linux/log/log.cc 
	src/client/linux/microdump_writer/microdump_writer.cc 
	src/client/linux/minidump_writer/linux_dumper.cc 
	src/client/linux/minidump_writer/linux_ptrace_dumper.cc 
	src/client/linux/minidump_writer/minidump_writer.cc 
	src/client/minidump_file_writer.cc src/common/convert_UTF.cc src/common/md5.cc 
	src/common/string_conversion.cc src/common/linux/breakpad_getcontext.S 
	src/common/linux/elfutils.cc src/common/linux/file_id.cc 
	src/common/linux/guid_creator.cc src/common/linux/linux_libc_support.cc 
	src/common/linux/memory_mapped_file.cc src/common/linux/safe_readlink.cc)

# 链接 log 库，对应 android.mk 中 LOCAL_EXPORT_LDLIBS
target_link_libraries(breakpad log)
```

这个 CMakeList.txt 文件的作用是编译 breakpad 的源码，使其成为一个引入库

我们还需要在我们自己项目的 CMakeList.txt 中引入 breakpad 这个库

```cmake
# ......

# 引入 breakpad 的头文件（api的定义）
include_directories(breakpad/src breakpad/src/common/android/include)

# 引入 breakpad 的 cmakelist，执行并生成 libbreakpad.a（api 的实现，类似 java 的 jar 包）
add_subdirectory(breakpad)

target_link_libraries(
		# ......
		breakpad) # 引入 breakpad 的库文件（api的实现）
		
# ......
```

此时执行编译，会在 #include "third_party/lss/linux_syscall_support.h" 报错，无法找到头文件

此文件从：https://chromium.googlesource.com/external/linux-syscall-support/+/refs/heads/master 下载 (需要翻墙)

下载好后放到工程中对应目录下，重新编译运行即可

###### BreakPad 简单使用

C / C++ 代码

```c++
#include <jni.h>
#include "breakpad/src/client/linux/handler/minidump_descriptor.h"
#include "breakpad/src/client/linux/handler/exception_handler.h"

bool DumpCallback(const google_breakpad::MinidumpDescriptor &descriptor, 
                  void *context, 
                  bool succeeded)
{
    __android_log_print(ANDROID_LOG_ERROR, "ndk_crash", 
                       "Dump path: %s", descriptor.path());
    // 如果回调返回 true，Breakpad 将把异常视为已完全处理，禁止任何其他处理程序收到异常通知。
    // 如果回调返回 false，Breakpad 会将异常视为未处理，并允许其他处理程序处理它。
    return false;
}

extern "C"
JNIEXPORT void JNICALL
Java_com_enjoy_crash_CrashReport_initBreakpad(JNIEnv *env, jclass type, jstring path_)
{
    const char *path = env->GetStringUTFChars(path_, 0);
    // 开启 crash 监控
    google_breakpad::MinidumpDescriptor descriptor(path);
    static google_breakpad::ExceptionHandler eh(descriptor, NULL, DumpCallback, 
                                                NULL, true, -1);
    env->ReleaseStringUTFChars(path_, path);
}
```

Java 代码

```java
public class CrashReport
{
    static {
        System.loadLibrary("bugly");
    }
    
    public static void init(Context context)
    {
		// 开启 ndk 监控
        File file = new File(context.getExternalCacheDir(), "native_crash");
        if (!file.exists())
        {
            file.mkdirs();
        }
        initBreakpad(file.getAbsolutePath());
    }
    
    private static native void initBreakpad(String path);
}
```

如果出现了 NDK Crash，就会在我们指定的目录：`/sdcard/Android/Data/[packageName]/cache/native_crash` 下生成 NDK Crash 的信息文件

###### Crash 分析

BreakPad 采集到的 Crash 信息记录在 minidump 文件中

minidump 是由微软开发的用于崩溃上传的文件格式

我们可以将此文件上传到服务器完成上报，但是此文件没有可读性可言，要将文件解析为可读的崩溃堆栈，需要按照 breakpad 文档编译 minidump_stackwalk 工具，而 Windows 系统编译个人不会

不过好在，无论你是 Mac、windows 还是 ubuntu，在 Android Studio 安装目录下的 `bin\lldb\bin` 目录中就存在一个对应平台的 minidump_stackwalk 工具

使用这里的命令行执行

```
minidump_stackwalk xxxx.dump > crash.txt
```

crash.txt 中的内容大致为

```c
Operating system: Android 0.0.0 Linux 4.4.124+ #1 SMP PREEMPT Wed Jan 30 07:13:09 UTC 

2019 i686
CPU: x86 // abi 类型
	 GenuineIntel family 6 model 31 stepping 1
	 3 CPUs
	 
GPU: UNKNOWN

Crash reason: SIGSEGV // 内存引用无效的信号
Crash address: 0x0
Process uptime: not available

Thread 0 (crashed) // crashed：出现 crash 的线程
 0 libbugly.so + 0x1feab // 出现 crash 的 so 文件和出错的寄存器地址
 	eip = 0xd5929eab esp = 0xffa85f30 ebp = 0xffa85f38 ebx = 0x0000000c
 	esi = 0xd71a3f04 edi = 0xffa86128 eax = 0xffa85f5c ecx = 0xefb19400
 	edx = 0x00000000 efl = 0x00210286
 	Found by: given as instruction pointer in context
 1 libart.so + 0x5f6a18
 	eip = 0xef92ea18 esp = 0xffa85f40 ebp = 0xffa85f60
 	Found by: previous frame's frame pointer
 	
Thread 1
......
```

接下来需要使用 Android NDK 里面提供的 addr2line 工具将寄存器地址转换为对应符号

addr2line 要使用对应 ABI 匹配的版本，同时进行分析的 so 文件需要带有符号信息 (一般在 `[项目根目录]\app\build\intermediates\cmake\debug\obj\` 目录下就可以找到有符号信息的 so 文件)

最后我们就可以出错的寄存器地址转换成具体的程序行号了

```
i686-linux-android-addr2line.exe -f -C -e libbugly.so 0x1feab
```

### 列举一下常见的内存泄漏场景？如何解决？

* 资源性对象未关闭

  对于资源性对象不再使用时，应该立即调用它的 `close()` 函数，将其关闭，然后再置为 null

  例如 Bitmap 等资源未关闭会造成内存泄漏，此时我们应该在 Activity 销毁时及时关闭

* 注册对象未注销

  例如 BraodcastReceiver、EventBus 未注销造成的内存泄漏，我们应该在 Activity 销毁时及时注销

* 类的静态变量持有大数据对象

  尽量避免使用静态变量存储数据，特别是大数据对象，建议使用数据库存储大数据对象

* 单例造成的内存泄漏

  优先使用 Application 的 Context，如需使用 Activity 的 Context，可以在传入 Context 时使用弱引用进行封装，然后，在使用到的地方从弱引用中获取 Context，如果获取不到，则直接 return 即可

* 非静态内部类的静态实例

  该实例的生命周期和 Application 一样长，这就导致该静态实例一直持有 Activity 的引用，Activity 的内存资源不能正常回收。

  此时，我们可以将该内部类设为静态内部类或将该内部类抽取出来封装成一个单例，如果需要使用 Context，尽量使用 ApplicationContext；

  如果需要使用 Activity Context，就记得用完后置空让 GC 可以回收，否则还是会内存泄漏

* Handler 临时性内存泄漏

  Message 发出之后存储在 MessageQueue 中，在 Message 中存在一个 target，它是 Handler 对象的一个引用，Message 在 MessageQueue 中存在的时间过长，就会导致 Handler 对象无法被回收；如果 Handler 对象是非静态的，则会导致 Activity 或 Service 无法被回收

  并且消息队列是在一个 Looper 线程中不断地轮询处理消息，当这个 Activity 退出时，消息队列中还有未处理的消息或者正在处理的消息，并且消息队列中的 Message 持有 Handler 对象的引用，Handler 对象又持有 Activity 的引用，所以导致该 Activity 的内存资源无法及时回收，引发内存泄漏

  解决方案： 

  * 使用一个静态 Handler 内部类，然后对 Handler 持有的对象 (一般是 Activity) 使用弱引用，这样在回收时，Handler 持有的对象也可以被回收

  * 在 Activity 的 onDestroy 或 onStop 时，应该移除消息队列中的消息，避免 Looper 线程的消息队列中有待处理的消息需要处理

    需要注意的是，AsyncTask 内部也是 Handler 机制，同样存在内存泄漏风险，但其一般是临时性的

    对于类似 AsyncTask 或是线程造成的内存泄漏，我们也可以将 AsyncTask 和 Runnable 类独立出来或者使用静态内部类

* 容器中的对象没清理造成的内存泄漏

  在退出程序之前，将集合 clear，然后将集合对象置为 null，再退出程序

* WebView

  WebView 也存在内存泄漏的问题，在应用中只要使用一次 WebView，内存就不会被释放掉

  我们可以为 WebView 开启一个独立的进程，使用 AIDL 与应用的主进程进行通信，WebView 所在的进程可以根据业务的需要选择合适的时机进行销毁，达到正常释放内存的目的

* 使用 ListView 时造成的内存泄漏

  在构建 Adapter 时，使用缓存的 convertView

以上这些常见的内存泄漏的 bug，我们可以通过 Android Studio 中自带的代码检查工具找出来

通过点击 Android Studio 中的 Analyze > Inspect Code... 来开启 AS 的代码检查 (如图示)

![](https://note.youdao.com/yws/api/personal/file/AB3DC28A01674645BF77187308D386AC?method=download&shareKey=7cf346b4197fe384670accdd9409825d)

[AndroidStudio使用inspect code帮助优化代码](https://blog.csdn.net/u013564742/article/details/81671533)

### 如何分析内存泄漏？

目前分析内存泄漏常见的方式有四种：

* Memory Analyzer Tools (MAT)
* Android Memory Profiler
* LeakCanary
* Koom (快手提出的最新解决方案)

#### Memory Analyzer Tools (MAT)

Android 系统为了方便我们定位内存泄漏，提供了一种捕获当前应用运行时内存状况的方案 —— 堆转储

堆转储显示的是应用运行时哪些对象正在使用内存，特别是在应用长时间运行后，堆转储可能会显示出我们认为不应再位于内存中却仍在内存中的对象，从而帮助我们识别内存泄漏

我们可以使用 MAT 对堆转储进行分析，从而快速地计算出对象在内存中所占用的大小，观察是哪一个对象没有被 GC 所回收，并可以通过报表更直观的展示出来

然而堆转储只存在于应用运行时，当我们退出应用时，会丢失堆转储。因此，我们需要将应用运行时的堆转储导出为 HPROF 文件后，才能使用 MAT 进行分析

Android SDK 为我们提供了一个 API 用于捕获堆转储并导出为 HPROF 文件 —— `android.os.Debug.dumpHprofData(String fileName)`

此时导出的 hprof 文件并不能直接进行分析，还需要将其从 Android 格式转换为 Java SE HPROF 格式 

我们可以通过使用 `android_sdk/platform-tools/` 目录中提供的 `hprof-conv` 工具执行转换操作

```shell
hprof-conv heap-original.hprof heap-converted.hprof
```

运行需要两个参数，即原始 HPROF 文件和转换后 HPROF 文件的写入位置

使用 MAT 工具打开 HPROF 文件后，我们需要点击 Create a histogram from an arbitrary set of objects（如图示）

![](https://note.youdao.com/yws/api/personal/file/A90FDCB8540F44918A7E27DFB385041B?method=download&shareKey=c0ddf316027bf7f255161dad907524b7)

将 HPROF 文件通过直方图的形式显示

为了更好地分辨我们的程序所占用的内存，我们可以点击 Group result by... > Group by package（如图示）

![](https://note.youdao.com/yws/api/personal/file/89185D386CCB43F9B334B5B82566B50D?method=download&shareKey=deb6f89dfadaab0c174c932057be9434)

这样我们就可以更清晰地查看到自己程序中所占用的内存

![](https://note.youdao.com/yws/api/personal/file/24297AA234DE43FB97B6A2234AAC063D?method=download&shareKey=3cef6e24da07f3d94a731eea5d45f231)

##### Shallow Heap 和 Retained Heap

* Shallow Heap 指的是对象 **本身** 所占用的内存大小
* Retained Heap 指的是当一个对象被回收后 **可以被级联回收** 的总内存大小

[Eclipse MAT 里面的SHALLOW HEAP和RETAINED HEAP是什么意思？](https://blog.csdn.net/goldenfish1919/article/details/94378369)

##### Incoming References 和 Outgoing References

* 对于某一个对象来说，拥有其引用的所有对象都称为 Incoming References
* 对于某一个对象来说，其引用的所有对象都称为 Outgoing References

[JVM 内存分析神器 MAT: Incoming Vs Outgoing References 你真的了解吗？](https://blog.csdn.net/weixin_45410925/article/details/102740403)

我们可以通过右键点击想要查看的对象，选择 List objects > with incoming references / with outgoing references (如图示)

![](https://note.youdao.com/yws/api/personal/file/3A0D3DF9E2AE49B6B50E442F10D3D487?method=download&shareKey=4d222c668dc67c28e37a53365b65dd42)

##### 查看 GC Roots 引用

GC Roots 包括：

* 作用域为函数内部的局部变量所引用的对象
* 被 static 修饰的变量 (静态变量) 所引用的对象
* 被 final 修饰的变量 (常量) 所引用的对象
* native 函数中所引用的对象

我们可以通过右键点击想要查看的对象，选择 Merge hortet Paths to GC Roots > exclude all phantom/weak/soft etc. references (如下图示)

![](https://note.youdao.com/yws/api/personal/file/571A71180B694CBDB605E860ED5495B7?method=download&shareKey=5ea60b50337c13940bee001364fd62fd)

#### [Android Memory Profiler](https://developer.android.google.cn/studio/profile/memory-profiler)

Android Memory Profiler 是 Android Profiler 中的一个组件，可帮助我们识别可能会导致应用卡顿、冻结甚至崩溃的内存泄漏和内存抖动

它显示一个应用内存使用量的实时图表，让我们可以捕获堆转储、强制执行垃圾回收以及跟踪内存分配

#### [LeakCanary](https://github.com/square/leakcanary)

要使用 LeakCanary，需要将 leakcanary-android 依赖项添加到应用程序 build.gradle 文件中：

```
dependencies {
  // debugImplementation because LeakCanary should only run in debug builds.
  debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.7'
}
```

**就是这样，不需要更改代码！**

可以通过过滤 Logcat 中的 LeakCanary 标签来确认 LeakCanary 在启动时正在运行：

```
D LeakCanary: LeakCanary is running and ready to detect leaks
```

#### [Koom](https://github.com/KwaiAppTeam/KOOM/blob/master/README.zh-CN.md)

KOOM (Kwai OOM，Kill OOM) 是快手性能优化团队在处理移动端 OOM 问题的过程中沉淀出的一套完整解决方案

### 内存抖动如何解决？

使用 Android Memory Profiler 定位到造成内存抖动的位置，将出现内存抖动的代码进行修改

内存抖动多半是由于对象的频繁创建和回收导致的，所以我们在解决内存抖动时应当注意：

* 避免在循环和频繁调用的函数中创建对象
* 如果需要频繁创建对象，使用对象池，如 Handler、Glide 中的对象池

### 内存优化总体思路是什么？

[「性能优化系列」不使用第三方库，Bitmap的优化策略](https://juejin.cn/post/6844904099297624077)

[「性能优化系列」APP内存优化理论与实践](https://juejin.cn/post/7001827852534579214)

#### 设备分级

所谓设备分级，指的是根据不同设备环境来考虑不同的内存优化策略

目前市场上手机层出不穷，几乎每一年都会对手机性能进行提升，但是对于性能较差的手机，app 应用的运行状况就会较差

> 对于低端机用户可以关闭复杂的动画，或者是某些功能；使用 565 格式的图片，使用更小的缓存内存等
>
> 在现实环境下，不是每个用户的设备都跟我们的测试机一样高端，在开发过程我们要学会思考功能要不要对低端机开启、在系统资源吃紧的时候能不能做降级
>
> —— 张邵文

针对不同设备的不同性能，我们能做的优化大致就是这几个方面：

- 是否关闭动画
- 图片质量分级
- 为低性能设备设计简版应用

Facebook 开发了一个 [Device Year Class](https://github.com/facebookarchive/device-year-class) (设备年份类库)，它使用简单的算法将设备的 RAM、CPU 内核和时钟速度与这些特性被认为是高端的年份相匹配，使得我们能够根据手机的硬件功能编写不同的逻辑

通过 Gradle 将库引用到项目中

```groovy
implementation 'com.facebook.device.yearclass:yearclass:2.1.0'
```

通过其提供的 `YearClass#get` 函数得到计算后当前设备所对应的年份

```java
int year = YearClass.get(getApplicationContext());
if (year >= 2013)
{
    // Do advanced animation
}
else if (year > 2010)
{
    // Do simple animation
}
else
{
    // Phone too slow, don't do any animations
}
```

#### Bitmap 优化

对于 Bitmap 的优化主要分为以下几种：

* 针对不同密度的设备合理的分配资源 (合理分配 drawable 目录)
* 压缩 (尺寸压缩、质量压缩)
* 缓存 (内存缓存、磁盘缓存)

##### 合理分配 drawable 目录

总所周知，drawable 目录是放置本地图片资源的地方

在 Android Studio 中将 drawable 目录分为了 mdpi、hdpi、xhdpi 等等不同的等级，对应着不同的设备屏幕分辨率 (dpi)

当 drawable 目录下的图片资源被加载时，其实是调用了 `BitmapFactory.decodeResource()` 函数将资源 id 转换成了 bitmap

在转换的过程中，会根据图片资源存放的不同 dpi 等级做一次分辨率的转换，转换的规则是：

```
新图宽高 = 原图宽高 * (当前设备的dpi / 目录对应的dpi)
```

因此当图片资源被不小心放在了低于自身 dpi 的 drawable 等级目录下时，其被加载时，图片会被放大，图片被放大也就意味着图片所占内存增大，长此以往就有可能造成 OOM

当图片资源被不小心放在了高于自身 dpi 的 drawable 等级目录下时，在其被加载时会被缩小，从而影响用户的体验

因此我们应该将不同 dpi 的图片资源放置在对应的 drawable 等级目录下，以对不同 dpi 的设备进行适配

##### 尺寸压缩

当装载图片的容器例如 ImageView 只有 100 * 100，而图片的分辨率为 800 * 800，此时将图片直接放置在容器上，很容易 OOM，同时这也是对图片和内存资源的一种浪费

当容器的宽高都远小于图片的宽高时，需要对图片进行尺寸上的压缩，将图片的分辨率调整为容器宽高的大小，这样一方面不会对图片的质量有影响，同时也可以很大程度上减少内存的占用

Android 开发者官网也给我们提供了尺寸压缩的 [实现方案](https://developer.android.google.cn/topic/performance/graphics/load-bitmap?hl=zh_cn)，使用 BitmapFactory.Options 中的 inSampleSize 参数

```java
/**
 * 计算用于缩放的inSampleSize值
 */
public static int calculateInSampleSize(
                BitmapFactory.Options options, int reqWidth, int reqHeight)
{
	// 图片的原始宽高
	final int height = options.outHeight;
	final int width = options.outWidth;
	int inSampleSize = 1;

    // 如果当前图片的高或者宽大于所需的高或宽
	if (height > reqHeight || width > reqWidth)
    {
		final int halfHeight = height / 2;
		final int halfWidth = width / 2;
        // 进行inSampleSize的2的幂次方自增运算，直到图片宽高符合所需要求
		while ((halfHeight / inSampleSize) >= reqHeight
                    && (halfWidth / inSampleSize) >= reqWidth)
        {
			inSampleSize *= 2;
		}
	}

	return inSampleSize;
}

/**
 * 对图片进行解码缩放
 */
public static Bitmap decodeSampledBitmapFromResource(Resources res, int resId,
            int reqWidth, int reqHeight)
{
	// inJustDecodeBounds设置为true，解码器将返回一个空的Bitmap，
    // 系统不会为此Bitmap分配内存，只能用于查询图片的宽高、类型信息等
	final BitmapFactory.Options options = new BitmapFactory.Options();
	options.inJustDecodeBounds = true;
	BitmapFactory.decodeResource(res, resId, options);

    // 计算inSampleSize
	options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);

	// 在真正要将图片解码加载进入内存时，需要将inJustDecodeBounds设置为false
	options.inJustDecodeBounds = false;
	return BitmapFactory.decodeResource(res, resId, options);
}
```

##### 质量压缩

一般情况下质量压缩是不推荐的一种优化手法，此手法压缩后图片将会失真，但不排除有项目对图片的清晰度没有过高的要求

直至 API 29，将像素点分为六个等级：

| Bitmap.Config  | 占用字节大小 |                             说明                             |
| :------------: | :----------: | :----------------------------------------------------------: |
|  ALPHA_8（1）  |      1       |                  单透明通道，不存储颜色信息                  |
|  RGB_565（3）  |      2       | 简易 RGB 色调，仅存储 RGB 通道，如果对 Bitmap 色彩没有高要求，可以使用该设置 |
| ARGB_4444（4） |      4       |                            已废弃                            |
| ARGB_8888（5） |      4       |      24位真彩色，保持高质量的色彩保真度，默认使用该设置      |
| RGBA_F16（6）  |      8       |    Android 8.0 新增 (更丰富的色彩表现)，适合宽色域和 HDR     |
| HARDWARE（7）  |      --      | Android 8.0 新增，一种特殊设置，减少了内存占用同时也加快了 Bitmap 的绘制 |

每个设置所对应每个像素所占用的字节也都不一样，所存储的色彩信息也不同

同一张 100 像素的图片，ARGB_8888 就占了 400 字节，RGB_565 才占 200 字节，RGB_565 在内存上取得了优势，但是 Bitmap 的色彩值以及清晰度却不如 ARGB_8888 设置下的 Bitmap

质量压缩说到底就是用清晰度来换内存

我们可以通过使用 BitmapFactory.Options 中的 inPreferredConfig 参数在解码时进行质量压缩

```java
options.inPreferredConfig = Bitmap.Config.ARGB_8888;
```

##### 缓存策略

不管是从网络上下载图片，还是直接从 USB 中读取图片，缓存对于图片加载的优化起到了至关重要的作用

当我们首次从网络上或者 USB 读取图片，会对图片进行相应的压缩处理，如果在处理过后不加入缓存，下一次请求图片还是直接从网络上或者 USB 中直接读取，不仅消耗了用户的流量还重复对图片进行压缩处理，占用多余内存的同时加载图片也很缓慢

对于缓存，目前的策略是内存缓存和磁盘缓存

Android 开发者官网也给我们提供了两种缓存的 [实现方案](https://developer.android.google.cn/topic/performance/graphics/cache-bitmap?hl=zh_cn)

##### 总结

一句话，能用 Glide 就用 Glide，这可是 Google 给广大开发者的忠告

因为 Glide 完全符合 Google 对于 Bitmap 优化所制定的各项标准

> 在大多数情况下，我们建议您使用 [Glide](https://github.com/bumptech/glide) 库获取、解码和显示应用中的位图。
>
> 在处理这些任务以及与位图和 Android 上的其他图片相关的其他任务时，Glide 会将大部分的复杂工作抽象出来。
>
> 如需了解如何使用和下载 Glide，请访问 GitHub 上的 [Glilt 代码库](https://github.com/bumptech/glide)
>
> 原话出处 —— [处理位图](https://developer.android.google.cn/topic/performance/graphics?hl=zh_cn)

#### 减少内存泄漏

减少内存泄漏的最主要途径就是 [分析 HPROF 文件](#如何分析内存泄漏？)，找到内存泄漏所在位置，进行修改

#### 解决内存抖动

使用 Android Memory Profiler 定位到造成内存抖动的位置，将出现内存抖动的代码进行修改

内存抖动多半是由于对象的频繁创建和回收导致的，所以我们在解决内存抖动时应当注意：

* 避免在循环和频繁调用的函数中创建对象
* 如果需要频繁创建对象，使用对象池，如 Handler、Glide 中的对象池

### ANR 产生的条件是什么？ANR 类型有哪些？

- KeyDispatchTimeout：输入事件 (触摸事件等) 在 5s 内没有被处理完成
  - 对于 KeyDispatchTimeout 类型来说，即便某次事件执行时间超过 5s 时长，但只要用户后续没有再生成输入事件，则不会触发 ANR
- BroadcastTimeout： 
  - 前台 Broadcast：onReceiver 函数在 10s 内没有执行结束
  - 后台 Broadcast：onReceiver 函数在 60s 内没有执行结束
- ServiceTimeout： 
  - 前台 Service：onCreate、onStart、onBind 等生命周期函数在 20s 内没有执行结束
  - 后台 Service：onCreate、onStart、onBind 等生命周期函数在 200s 内没有执行结束
- ContentProviderTimeout：ContentProvider 在 10s 内没有处理完成当前事务

### 常见的会造成 ANR 的操作，列举几个？

* 主线程频繁进行耗时的 IO 操作，如数据库读写
* 多线程操作的死锁，主线程被阻塞
* 主线程被 Binder 对端阻塞
* System Server 进程中的 WatchDog 出现 ANR
* Binder 的连接数达到上限无法和 System Server 进行通信
* 系统资源已耗尽，如管道、CPU、IO

### 如果发生了 ANR，如何分析？

当出现 ANR 时，在系统的日志系统中会出现相应的日志信息，我们可以通过 adb 命令将日志信息抓取下来导出到文件中进行分析

```shell
> adb logcat -v long -d > log.txt
```

关于 Android 日志系统 —— [骇极干货｜第2期：Android日志系统分析](https://www.sohu.com/a/365755232_675300) (大致了解就可以了)

关于 logcat 命令的使用 —— [adb logcat命令行用法](https://blog.csdn.net/u011793251/article/details/51741048)

因为我们抓取的整个系统运行期间的日志信息，所以在这个文件中有许多是我们不需要的日志信息

我们可以通过关键字 `ANR IN` 进行搜索，定位到我们想要看的 ANR 日志信息

```
[ 10-17 18:45:54.031  3320: 3769 E/ActivityManager ]
ANR in com.chenjimou.anrtestdemo (com.chenjimou.anrtestdemo/.MainActivity)
PID: 996
Reason: Input dispatching timed out (Waiting to send non-key event because the touched window has not finished processing certain input events that were delivered to it over 500.0ms ago.  Wait queue length: 4.  Wait queue head age: 5645.6ms.)
```

从上面的这条 log 我们可以知道以下信息：

* ANR 发生时间：07-20 15:36:36.472
* 进程pid：996
* 进程名：com.chenjimou.anrtestdemo
* ANR 类型：KeyDispatchTimeout (Input dispatching timed out)

现在我们知道了当前 ANR 的类型是 KeyDispatchTimeout，我们可以通过关键字 `PID: 996` 向上进行搜索，查看该进程在前 5s 的时间内做了什么事情

或者通过关键字 `CPU usage` 向下进行搜索，查看各个进程占用 cpu 的情况 (不一定会有)

但是仅仅通过日志信息，我们只能知道 ANR 的类型以及发生 ANR 时系统的一些状态信息，这些是不足以定位到具体的错误位置的，因此我们还需要对 traces.txt 文件进行分析

当发生 ANR 时，系统就会 dump 出一个 traces.txt 文件，存放在文件目录：data/anr/

> 对于 Android 原生系统来说，每产生一次 ANR，就会将本次 ANR 的相关信息单独存入一个 txt 文件中，并以 ANR 发生时的时间戳对 txt 文件进行命名，因此在 data/anr/ 目录下存在多个 txt 文件
>
> 但是也有一些手机厂商对原生系统进行魔改之后，会将每一次的 ANR 信息存入一个名称是 traces.txt 的文件，并覆盖文件中原先的内容，也就是上一次的 ANR 信息，因此在 data/anr/ 目录下有可能只存在一个名称是 traces.txt 文件
>
> 虽然不同系统之间生成 traces.txt 文件的方式有可能不一样，但是其文件的内容格式都是一样的

通过 traces.txt 文件，我们可以获取到线程名、堆栈信息、线程当前状态、binder call 等信息

我们可以通过 adb 命令将 traces.txt 文件导出到电脑上进行分析

```shell
> adb pull /data/anr/traces.txt
```

对于 traces.txt 文件，我们不需要查看所有的内容，我们只需要关注几个重点的地方即可

```
----- pid 25240 （pid）at 2021-10-17 17:50:00 （发生 ANR 的时间）-----
Cmd line: com.chenjimou.anrtestdemo（发生 ANR 的进程名）
......
ABI: 'arm64'（手机的 cpu 架构类型）
......
Heap: 12% free, 11MB/12MB; 56685 objects（已分配堆内存大小12MB，其中11M已用，总分配56685个对象）
......

DALVIK THREADS (14):（当前进程总14个线程）
......
"main" prio=5 tid=1 Sleeping（主线程调用栈）
  | group="main" sCount=1 dsCount=0 obj=0x756b3320 self=0x7f80097e00
  | sysTid=25240 nice=-10 cgrp=default sched=0/0 handle=0x7f84a30a98
  | state=S schedstat=( 186292451 2604946 188 ) utm=17 stm=1 core=2 HZ=100
  | stack=0x7fd8c7d000-0x7fd8c7f000 stackSize=8MB
  | held mutexes=
  （以下就是调用栈，顺序从下到上）
  at java.lang.Thread.sleep!(Native method)
  - sleeping on <0x08f9de52> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:371)
  - locked <0x08f9de52> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:313)
  at com.chenjimou.anrtestdemo.MainActivity$1.onClick(MainActivity.java:29)
  at android.view.View.performClick(View.java:5642)
  at com.google.android.material.button.MaterialButton.performClick(MaterialButton.java:967)
  at android.view.View$PerformClick.run(View.java:22491)
  at android.os.Handler.handleCallback(Handler.java:751)
  at android.os.Handler.dispatchMessage(Handler.java:95)
  at android.os.Looper.loop(Looper.java:154)
  at android.app.ActivityThread.main(ActivityThread.java:6306)
  at java.lang.reflect.Method.invoke!(Native method)
  at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:1108)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:998)

......
```

线程调用栈中的属性含义：

* main：main 标识是主线程，如果是其他线程，那么命名成 Thread-X 的格式，X 表示线程 id，逐步递增
* prio：线程优先级，默认是 5
* tid：线程内部 id
* group：线程组名
* sCount：线程被挂起的次数
* dsCount：debug 模式下，线程被挂起的次数
* obj：线程关联的 java 线程对象的地址
* self：线程在内核中的地址
* sysTid：线程在内核中的 tid
* nice：线程的调度优先级，nice 值越小则优先级越高
* cgrp：进程所属的进程调度组
* sched：线程的调度策略
* handle：线程处理函数的地址
* state：线程状态
* schedstat：CPU 调度时间统计，括号中的 3 个数字依次是 Running、Runable、Switch，紧接着的是 utm 和 stm
  * Running 时间：CPU 运行时间，单位 ns
  * Runable 时间：RQ 队列的等待时间，单位 ns
  * Switch 次数：CPU 调度切换次数
  * utm：线程在用户态所执行的时间，单位是 jiffies，默认等于 10 ms
  * stm：线程在内核态所执行的时间，单位是 jiffies，默认等于 10 ms

* core：最后执行该线程的 CPU 核的序号
* HZ：时钟频率
* stack：线程栈的地址区间
* stackSize：线程栈的大小
* mutexes：所持有 mutex (锁) 的类型，有独占锁 exclusive 和共享锁 shared 两类，为空就是没有持有锁

从上面的线程调用栈中，我们可以查看到是由于在 MainActivity 的点击事件回调中对主线程进行了睡眠，导致了我们的进程 com.chenjimou.anrtestdemo 发生了 ANR

这时候可以对着源码查看，找到出问题，并且解决它

所以分析 ANR 的整体流程大致为：

1. 通过 adb 命令抓取日志信息，找出发生 ANR 的时间点、进程 PID、ANR 类型，并通过前后日志，找出发生 ANR 的进程前几秒干了什么，以及发生 ANR 时，各个进程占用 cpu 的情况，排除 ANR 是由于我们自己的进程运行造成的还是由于系统 CPU 占用太高而导致的
2. 通过 adb 命令导出 traces.txt 文件，找到主线程调用栈，根据主线程的堆栈信息定位代码位置，最后找到源码位置进行修改即可

### 如何在线上监控 ANR？

目前市面上常见的实现方案有两种：

* FileObserver
* WatchDog

#### FileObserver

我们知道当发生 ANR 的时候，系统会在 data/anr/ 目录下生成 traces.txt 文件

Android SDK 为我们提供了一个抽象类 FileObserver 能够用于监听文件 / 文件夹的变化，我们可以用其去监听 data/anr/ 目录下文件的变化从而达到监控 ANR 的目的

```java
public class ANRFileObserver extends FileObserver
{
    public ANRFileObserver(String path) //data/anr/
    {
        super(path);
    }

    public ANRFileObserver(String path, int mask)
    {
        super(path, mask);
    }

    @Override
    public void onEvent(int event, @Nullable String path)
    {
		switch (event)
        {
            case FileObserver.ACCESS: // 文件被访问
                break;
            case FileObserver.ATTRIB: // 文件属性被修改，如 chmod、chown、touch 等
                break;
            case FileObserver.CLOSE_NOWRITE: // 不可写文件被 close
                break;
            case FileObserver.CLOSE_WRITE: // 可写文件被 close
                break;
            case FileObserver.CREATE: // 创建新文件
                break;
            case FileObserver.DELETE: // 文件被删除，如 rm
                break;
            case FileObserver.DELETE_SELF: // 自删除，即一个可执行文件在执行时删除自己
                break;
            case FileObserver.MODIFY: // 文件被修改
                break;
            case FileObserver.MOVE_SELF: // 自移动，即一个可执行文件在执行时移动自己
                break;
            case FileObserver.MOVED_FROM: // 文件被移走，如 mv
                break;
            case FileObserver.MOVED_TO: // 文件被移来，如 mv、cp
                break;
            case FileObserver.OPEN: // 文件被 open
                break;
            default:
                // CLOSE：文件被关闭，等同于(IN_CLOSE_WRITE | IN_CLOSE_NOWRITE)
                // ALL_EVENTS：包括上面的所有事件
                break;
        }
    }
}
```

使用 FileObserver 我们需要注意以下几点：

* 启动监控使用 startWatching 函数，停止监控使用 stopWatching 函数。在 startWatching 和 stopWatching 函数内部都存在加锁操作，因此使用这两个函数时不要进行加锁操作，否则很可能会引发死锁，从而导致 ANR
* 由于监听需要保证 FileObserver 对象的存活，我们最好在 Application 创建的时候创建 FileObserver 对象（在 Application 的 onCreate 回调中），并使其生命周期跟随 Application，防止其被 GC 回收（使用静态字段进行引用）

[android FileObserver的用法与避坑指南](https://blog.csdn.net/J_bing/article/details/47146953)

[FileObserver文件监听](https://www.jianshu.com/p/8f47c699b742)

#### WatchDog

WatchDog 翻译过来就是看门狗，有保护的意思

这个概念最先是在单片机上提出的，由于单片机的工作环境容易受到外界磁场的干扰，造成整个系统无法正常工作，因此，引入了一个“看门狗”，对单片机的运行状态进行实时监测，针对运行故障做一些保护处理，譬如让系统重启

硬件都有可能会出现程序跑飞的状况，更何况是软件呢？所以 Android 系统在 FrameWork 层面也引入了一个 WatchDog 用来监控线程的运行状态

Android WatchDog 的工作流程如下图

![](https://note.youdao.com/yws/api/personal/file/3A539195C0644A6D804E588A23FD1E6E?method=download&shareKey=acddc0aba383bef0cccb4b4e55de5a4d)

Android WatchDog 利用了 MessageQueue 处理耗时任务时会被阻塞的特点，周期性地在所要监听线程的 MessageQueue 中插入 (插入到 MessageQueue 的最前方，保证其优先执行) 一个用于检测的 Runnable，根据 Runnable 是否在规定时间内执行完成来判断所监听的线程是否卡顿、发生 ANR

根据这个思路我们可以自己定义一个 WatchDog 用来监控 ANR

```java
public class ANRWatchDog extends Thread
{
    private static final String TAG = "ANR";
    private int timeout = 5000;

    private volatile static ANRWatchDog sWatchdog;

    private Handler mainHandler = new Handler(Looper.getMainLooper());

    private ANRChecker anrChecker = new ANRChecker();

    private class ANRChecker implements Runnable
    {
        private boolean mCompleted;
        private long mStartTime;
        private long executeTime = SystemClock.uptimeMillis();

        @Override
        public void run()
        {
            synchronized (ANRWatchDog.this)
            {
                // 设置 mCompleted 标志位
                mCompleted = true;
                executeTime = SystemClock.uptimeMillis();
            }
        }

        void schedule()
        {
            mCompleted = false;
            mStartTime = SystemClock.uptimeMillis();
            // 将 Runnable 插入 MessageQueue 的最前方，保证其优先执行
            mainHandler.postAtFrontOfQueue(this);
        }

        boolean isBlocked()
        {
            return !mCompleted || executeTime - mStartTime >= 5000;
        }
    }

    public static ANRWatchDog getInstance()
    {
        if (sWatchdog == null)
        {
            synchronized (ANRWatchDog.class)
            {
                if (sWatchdog == null)
                {
                    sWatchdog = new ANRWatchDog();
                }
            }
        }
        return sWatchdog;
    }

    @TargetApi(Build.VERSION_CODES.JELLY_BEAN)
    @Override
    public void run()
    {
        // 设置为后台线程
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        try
        {
            while (!isInterrupted())
            {
                synchronized (this)
                {
                    // post 任务 Runnable 到所要监控线程的 MessageQueue 开启监控
                    anrChecker.schedule();
                    // 防止假唤醒，确保休眠 5s
                    long waitTime = timeout;
                    long start = SystemClock.uptimeMillis();
                    while (waitTime > 0)
                    {
                        try
                        {
                            wait(waitTime);
                        }
                        catch (InterruptedException e)
                        {
                            Log.e(TAG, e.toString());
                        }
                        waitTime = timeout - (SystemClock.uptimeMillis() - start);
                    }
                    // 检测 Runnable 的 mCompleted 标志位和执行完成所耗时间
                    if (!anrChecker.isBlocked())
                    {
                        continue;
                    }
                    // 发生异常，获取堆栈信息
                    String stackTraceInfo = getStackTraceInfo();
                    // 保存堆栈信息，上传后台
                    // dumpStackTraces(stackTraceInfo)
                    // ......
                }
            }
        }
        catch (InterruptedException e)
        {
            Log.e(TAG, e.toString());
        }
    }

    private String getStackTraceInfo()
    {
        StringBuilder stringBuilder = new StringBuilder();
        for (StackTraceElement stackTraceElement : 
             		Looper.getMainLooper().getThread().getStackTrace())
        {
            stringBuilder
                    .append(stackTraceElement.toString())
                    .append("\r\n");
        }
        return stringBuilder.toString();
    }
}
```

# Java

[重温数据结构：哈希 哈希函数 哈希表](https://blog.csdn.net/u011240877/article/details/52940469)

[面试旧敌之 HashMap ：主要特点和关键方法源码解读](https://juejin.cn/post/6844903454951866375)

## HashMap

### HashMap 的原理？

HashMap 是一个采用哈希表实现的 key-value 键值对集合，其底层实现是数组 + 链表 + 红黑树 (JDK 8 之后新添加)

当调用 get / put 时，会先通过 `hash()` 函数计算 key 所对应的哈希值 (将 key 的 hashCode 进行无符号右移 16 位，再进行按位异或)

再通过哈希值计算 ( `(当前哈希表长度 - 1) & 哈希值` ) 得到目标元素在哈希表中的位置

当发生哈希碰撞 (冲突) 时，HashMap 采用拉链法将所有计算得到同一位置的元素链接成一个单链表 (JDK 7 之前是单链表，JDK 8 之后是单链表 + 红黑树)

在概念上，哈希表中所储存的单链表 / 红黑树称之为哈希桶

### HashMap 为什么会形成环？

因为 HashMap 是线程不安全的

当 HashMap 发生扩容 (哈希表的容量 > 最大容量 * 加载因子) 时，会调用 `resize()` 函数

`resize()` 函数会将原来的哈希表容量和阈值扩大一倍，并将原来的元素重新计算哈希值后再保存

由于 HashMap 线程不安全，在多线程下并发调用 `resize()` 函数时，在将原来的元素重新保存的过程中，就有可能使链表形成一个闭环

当调用 get / put 访问到这个闭环时，就会使运行陷入死循环，严重占用 CPU 资源

[HashMap原理及线程不安全详解](https://my.oschina.net/muziH/blog/1596801)

### HashMap 是如何解决哈希碰撞 (冲突) 的？

当发生哈希碰撞 (冲突) 时，HashMap 采用拉链法将所有计算得到同一位置的元素链接成一个单链表

在 JDK 7 之前哈希桶结构只是单链表

在 JDK 8 之后哈希桶结构采用单链表 + 红黑树

当哈希桶内元素个数小于 8 时，仍采用单链表的结构；当哈希桶内元素个数大于 8 时，则采用红黑树的结构

(当哈希桶内元素个数逐渐增多时，在搜索速度上，单链表比不上红黑树)

###  HashMap 的扩容机制？

每次调用 put 时，HashMap 会比较当前哈希表的容量是否超出阈值 (最大容量 * 加载因子)

如果超出阈值，就会调用 `resize()` 函数，对哈希表进行扩容

`resize()` 函数会将原来的哈希表容量和阈值扩大一倍，并将原来的元素重新计算哈希值后再保存

HashMap 的扩容开销会很大，需要遍历所有的元素，重新计算哈希值，再赋值，还得保留原来的数据结构 (如果扩容前哈希桶结构已经变成了红黑树，则扩容后哈希桶结构不会变回单链表，即保留数据结构)

因此在使用 HashMap 时，最好设置一个合理的初始容量，尽量避免频繁出现扩容

### 如何解决 HashMap 线程不安全？

* 使用 ConcurrentHashMap

  ```java
  ConcurrentHashMap<Integer, Integer> chm = new ConcurrentHashMap<>();
  ```

* 通过 Collections 封装 HashMap

  ```java
  HashMap<Integer, Integer> hm = new HashMap<>();
  Map<Integer, Integer> synchronizedMap = Collections.synchronizedMap(hm);
  ```

### 使用 HashMap 时，key 重写了 equals 但没有重写 hashcode 会怎么样？

hashcode 默认会根据对象的内存地址生成的一个 int 整数

equals 默认是用来比较对象地址是否相同 (相当于 == 运算符)

在使用 HashMap 时，如果重写了 equals 但没有重写 hashcode，就有可能导致 HashMap 存储重复的元素

因为 HashMap 内部会通过比较哈希值、key 和 value 是否相同来判断元素是否相同

如果重写了 equals 但没有重写 hashcode，就有可能导致两个元素比较的时候，key 和 value 都是相同的 (通过 equals 比较)，但是哈希值不同 (通过 hashcode 比较)，因此 HashMap 不会发生替换

这就会导致 HashMap 储存了两个 key 和 value 都是相同的元素，造成重复

## HashSet

### HashSet 和 HashMap 的区别？

* HashSet 实现了 Set 接口，仅存储对象

  HashMap 实现了 Map 接口, 存储的是 key-value 键值对

* HashSet 底层其实通过 HashMap 实现存储的，HashSet 封装了一系列 HashMap 的函数，依靠 HashMap 来存储对象 (通过 key 进行存储，而 value 默认为 Object 对象)

  所以 HashSet 也不允许出现重复，判断标准和 HashMap 判断标准相同 (两个元素的 hashCode 相等且 equals 返回 true)

## HashTable

### HashTable 和 HashMap 的区别？

* HashTable 继承自 Dictionary，HashMap 继承自 AbstractMap，但二者都实现了 Map 接口

* HashTable 线程安全，而 HashMap 线程不安全

  HashTable 几乎可以等价于 HashMap，除了 HashTable 是线程安全的 (其内部使用了 synchronized 锁)

## ConcurrentHashMap

###  ConcurrentHashMap 的分段锁？

在 JDK 1.7 的版本中，ConcurrentHashMap 在内部使用了一个叫做 Segment 的结构

一个 Segment 可以理解为是一个 HashTable，Segment 内部维护了一个链表数组，其整体结构如下图所示：

![](https://itqiankun.oss-cn-beijing.aliyuncs.com/picture/blogArticles/2020-04-16/1587000999.png)

因此 ConcurrentHashMap 定位一个元素需要进行两次 Hash 操作，第一次 Hash 定位到 Segment，第二次 Hash 定位到元素所在的哈希桶

这样带来的好处是进行写操作时可以只对元素所在的 Segment 进行加锁即可，从而不会影响到其他的 Segment

在最理想的情况下，ConcurrentHashMap 最高可以同时支持 Segment 数量大小的写操作，大大提升 ConcurrentHashMap 的并发能力

而在 JDK 1.8 的版本中，ConcurrentHashMap 又再进行了优化，废弃了 Segment 这一结构，直接将数组中的每个元素都作为锁

当调用 put 时，如果哈希桶中没有元素，则直接通过 CAS 操作来插入元素

同时使用 synchronized 来保证并发安全，如果哈希桶中已存在元素，则使用 synchronized 关键字对元素加锁

[ConcurrentHashMap分段加锁机制简析](https://blog.csdn.net/LO_YUN/article/details/106358362)

###  HashMap、Hashtable、ConcurrentHashMap 的区别？

|          |         HashMap         |        HashTable        |    ConcurrentHashMap    |
| :------: | :---------------------: | :---------------------: | :---------------------: |
| 底层实现 |  数组 + 链表 + 红黑树   |       数组 + 链表       |  数组 + 链表 + 红黑树   |
|   空值   | key 和 value 都可以为空 | key 和 value 都不能为空 | key 和 value 都不能为空 |
| 线程安全 |       线程不安全        |    线程安全，效率低     |    线程安全，效率高     |

## ArrayList

### ArrayList 的原理？

ArrayList 的底层实现是数组，并且是一个定长数组

每次调用 add 时，ArrayList 会判断此次添加是否会造成数组越界

如果会发生数组越界，就会调用 `ensureCapacityInternal()` 函数，对数组进行扩容

`ensureCapacityInternal()` 函数会将原来的数组长度扩大 1.5 倍

但扩容、删除等操作都是有代价的，在一些极端的情况下，ArrayList 会将大量的元素进行移位，影响性能

因此 ArrayList 适用于读多写少的场景

同时 ArrayList 线程不安全，多线程场景下可以使用 CopyOnWriteArrayList

[ArrayList 从源码角度剖析底层原理](https://juejin.cn/post/6986833022427349005)

## LinkedList

### LinkedList 和 ArrayList 的区别？

* ArrayList 基于动态数组实现的非线程安全的集合

  LinkedList 基于链表实现的非线程安全的集合

* 对于随机索引访问的 `get()` 和 `set()` 函数，一般 ArrayList 的性能要优于 LinkedList

  因为 ArrayList 可以直接通过数组下标直接找到元素

  而 LinkedList 需要移动指针遍历每个元素直到找到为止

* 新增和删除元素，一般 LinkedList 的性能要优于 ArrayList

  因为 ArrayList 在新增和删除元素时，可能会对数组进行扩容和复制

  而 LinkedList 除了实例化对象需要时间外，只需要修改指针即可

* LinkedList 不支持随机访问 (RandomAccess)

* ArrayList 的空间浪费主要体现在 list 列表的结尾需要预留一定的容量空间

  而 LinkedList 的空间花费则体现在它的每一个元素都需要消耗相当的空间

## String

### String 使用 equals 和 == 比较的区别？

== 运算符比较的是两个对象所在的内存地址是否相同，即比较是否为同一个对象

`equals()` 函数是根类 Object 中定义的函数，其默认实现是将当前对象与参数对象通过 == 运算符进行比较

而 String 类重写了 `equals()` 函数，将其实现为比较两个 String 对象中的字符内容是否相同

所以 String 类使用 `equals()` 函数比较的是内容，使用 == 比较的是内存地址

### String 是否能被重写？

String 类不能被重写，String 类是一个 final 类

## 访问限制符

### public protect default private 四个限制符的作用？

- public：对所有类可见
- private：只对类内部的函数 / 内部类可见
- default (默认)：对同一包内其他类可见
- protected：对继承子类可见

## 类

### 4种内部类，特别注意每个 class 编译后都会产生一个 class 文件，不管静态或非静态



###  lambda 的本质



### 抽象类和接口的区别



## 异常

### 异常体系、分类、机制



### 与error的区别



## IO

### 主要还是问NIO的原理以及优缺点。建议把缓冲流的原理也得学一学并进行比较。



## 线程池

### 线程池的工作原理？

![](https://note.youdao.com/yws/api/personal/file/WEB9e14bdcf6385ee1fce9d3f0e160327fe?method=download&shareKey=b5ecf83b5b6a655195db9e4be3fa4740)

### 线程池构造参数的作用？应当如何配置？

* corePoolSize：线程池中的核心线程数

  当提交一个任务时，线程池会创建一个新线程执行任务，直到池中线程数等于 corePoolSize

* maximumPoolSize：线程池中允许的最大线程数

  当提交一个任务时，如果任务等待队列已满且当前没有空闲线程，线程池会创建一个新线程执行任务，直到池中线程数等于 maximumPoolSize

* keepAliveTime：普通线程空闲时的存活时间

  线程池中的线程分为核心线程和普通线程

  当池中线程数 < corePoolSize 时，线程池中的所有线程都是核心线程

  当 corePoolSize < 池中线程数 < maximumPoolSize 时，线程池中的所有线程都是普通线程

* TimeUnit：keepAliveTime 的时间单位

* workQueue：任务等待队列 (阻塞队列)

  当提交一个任务时，如果池中线程数 > corePoolSize 且当前没有空闲线程，任务会进入等待队列

  一般来说，我们应该尽量使用有界队列，因为使用无界队列会导致线程池某些参数无效

  更重要的是，使用无界队列可能会消耗大量的系统资源，这就违背了使用线程池的初衷

* ThreadFactory：创建线程的工厂类

  通过自定义线程工厂可以给每个新建的线程设置一个具有识别度的线程名，当然还可以更加自由的对线程做更多的设置，例如设置新建的线程为守护线程

* RejectedExecutionHandler：线程池的拒绝策略

  当提交一个任务时，如果任务等待队列已满且池中线程数 > maximumPoolSize，线程池会拒绝执行此任务，并通过拒绝策略处理此任务

  线程池提供了四种拒绝策略：

  * AbortPolicy：直接抛出异常 (默认)
  * CallerRunsPolicy：用调用者所在的线程来执行此任务
  * DiscardOldestPolicy：丢弃任务等待队列中位于头部的任务，并向线程池重新提交此任务
  * DiscardPolicy：直接丢弃此任务，即不处理

线程池构造参数的配置需要根据具体业务场景进行分析

例如在 OkHttp 框架中，需要高并发执行 Http 请求

因此 OkHttp 将线程池的 corePoolSize 参数设置为 0，将 maximumPoolSize 设置为 Integer.MAX_VALUE，将 workQueue 设置为 SynchronousQueue

这样 OkHttp 就得到了一个具有最大并发量的线程池

### 线程池的作用？

使用线程池有三个好处：

* 降低资源消耗：通过重复利用已创建的线程来减少线程创建和销毁所造成的资源消耗

* 提高响应速度：当任务到达时，任务可以不需要等到线程创建就能立即执行

* 提高线程的可管理性：线程是稀缺资源，如果无限制地创建，不仅会消耗系统资源，还会降低系统的稳定性

  使用线程池可以进行统一分配、调优和监控线程

## 并发

### synchronized 的底层原理？

在 JVM 中，每一个对象 (包括 Class 对象) 都关联着一个 Monitor 对象

synchronized 底层是通过 Monitor 对象来实现锁机制的

当代码运行到 synchronized 关键字时，线程会执行 monitorenter 指令 (JVM 指令) 去获取锁

如果线程可以获取到锁，则将 Monitor 对象中的 owner 字段设置 (CAS 操作) 为当前线程 id (owner 表示持有锁的线程)，并将 Monitor 对象中的 state 字段值加一 (state 表示持有锁的线程获取锁的次数，保证了锁的可重入性)

如果线程没有获取到锁，则进入 Monitor 对象的 EntryList 队列进行等待 (阻塞)

当持有锁的线程退出 synchronized 范围时，会执行 monitorexit 指令 (JVM 指令)，将 Monitor 对象中的 state 字段值减一；如果 state = 0，则释放锁，重置 Monitor 对象中的 owner 字段，并唤醒 EntryList 队列中的线程继续抢夺锁

### synchronized 修饰静态函数和修饰非静态函数有什么区别？

synchronized 修饰静态函数相当于是类锁，锁住的是当前类的 Class 对象

synchronized 修饰非静态函数相当于是对象锁，锁住的是当前类对象自身 (this)

### synchronized 修饰代码块和修饰函数有什么区别？

当 synchronized 修饰代码块时，代码经过编译后，编译器会给代码块的前后插入 monitorenter 和 monitorexit 指令，进行锁的获取和释放

当 synchronized 修饰函数时，代码经过编译后，编译器会给函数设置 ACC_SYCHRONIZED 访问标识，用于声明此函数是一个同步函数

当线程执行函数时，如果发现函数有 ACC_SYCHRONIZED 标识，则会去获取锁；函数执行完成后会释放锁

相当于在函数主体的前后隐式插入 monitorenter 和 monitorexit 指令

### synchronized 的类锁和对象锁有什么区别？

类锁也就是 Class 对象锁，所有对象共享同一个类锁

不同的对象使用的对象锁不是同一个

### synchronized 同步函数递归调用自身，这样可行吗？为什么？

可行，synchronized 是可重入锁

synchronized 底层是通过 Monitor 对象来实现锁机制的，Monitor 对象中的 state 字段表示持有锁的线程获取锁的次数

synchronized 同步函数每次递归调用自身时，会将 Monitor 对象中的 state 字段值加一，记录锁的重入次数

### synchronized 和 ReentrantLock 的区别？

* **实现原理上**

  synchronized 是依赖于 JVM 实现的，而 ReenTrantLock 是 JDK 实现的

* **性能上**

  在 JDK 1.6 优化 synchronized 之前，synchronized 的性能是比 ReenTrantLock 差很多的

  但自从 synchronized 引入了偏向锁、轻量级锁、自旋锁后，二者的性能就差不多了

  在两种方法都可用的情况下，官方甚至建议使用 synchronized，其实 synchronized 的优化感觉就是借鉴了 ReenTrantLock 中的 CAS 技术

  都是试图在用户态就把加锁问题解决，避免进入内核态的线程阻塞

* **便利性**

  很明显 synchronized 的使用比较方便简洁，由编译器去保证加锁和释放锁

  而 ReenTrantLock 需要开发者手动声明来加锁和释放锁，为了避免忘记释放锁造成死锁，所以最好在 finally 中释放锁

* **锁的细粒度和灵活度**

  很明显 ReenTrantLock 优于 synchronized

但 ReentrantLock 有几个独有的功能：

* ReenTrantLock 可以设置锁是公平锁还是非公平锁，而 synchronized 只能是非公平锁

* ReenTrantLock 提供了一个 Condition 类，用于实现分组唤醒线程

  而 synchronized 唤醒线程是随机的，无法设置

* ReenTrantLock 提供了一种能够中断等待锁的线程的机制，通过 lock.lockInterruptibly() 实现

[Synchronized 与 ReentrantLock 的区别！](https://cloud.tencent.com/developer/article/1458822)

### volatile 的作用？

volatile 是 Java 中最轻量的同步机制，可以解决线程可见性的问题

根据 JMM 内存模型我们可知，每个 CPU 执行单元都会有一个高速缓存，从主存中读取的数据会在高速缓存中保留数据副本

由于 CPU 执行速度远大于内存访问速度，因此当 CPU 执行单元运行线程执行代码时，会先从高速缓存中访问数据副本，之后再将数据副本更新到主存中

这就导致在多线程下对同一变量进行操作时，各线程对该变量的修改都只在各自的高速缓存中，对于其他线程来说是不可见的

而 volatile 会开启 CPU 缓存一致性协议 (MESI)，该协议的作用是当变量被修改后，会将此修改立即更新到主存中，并使该变量在其他高速缓存中的数据副本失效

当其他线程访问这一变量时，由于高速缓存中的数据副本已经失效，因此就需要重新访问主存读取数据，从而保证了数据的可见性

在多线程下，出于优化的原因，CPU 可能会乱序执行字节码，例如造成对象的半初始化

volatile 可以禁止指令重排，防止半初始化对象的出现

### volatile 能否保证原子性？

volatile 只能保证单一的读取 / 赋值是原子性的

可当多个读取 / 赋值操作集中到一行代码中时 (例如 i++)，那么这行代码就不是原子操作了，volatile 也就无法保证原子性

### synchronized 和 volatile 的区别？

volatile 保证了不同线程对变量进行操作时的可见性，即一个线程修改了某个变量的值，这新值对其他线程来说是立即可见的

但是 volatile 不能保证操作的原子性，因此多线程下的写复合操作会导致线程安全问题

synchronized 保证了多个线程在同一个时刻，只能有一个线程可以执行函数 / 代码块，它保证了线程对变量访问的可见性和排他性

### CAS 机制？ABA 问题？

CAS 即 Compare And Swap 的缩写，翻译成中文就是比较并交换

其作用是让 CPU 比较内存中某个值是否和预期值相同，如果相同则将这个值更新为新值，如果不相同则不更新 (如果内存中的值与预期值不相同说明有其他线程更改了内存中的值)

CAS 是通过 C / C++ 调用 CPU 指令实现的，所以效率很高

由于 CAS 在更新值时，会检查内存中的值是否发生变化，如果没有发生变化 (内存中的值 = 预期值) 则更新

但如果内存中的值从 A --> B --> A，那么当 CAS 进行检查时会发现内存中的值没有发生变化，但实际上内存中的值变化了，这就是 ABA 问题

ABA 问题的解决方案就是使用版本号

在变量前面追加上版本号，每次变量更新时变量的版本号都加一，即 A --> B --> A 就变成了 1A --> 2B --> 3A

### 什么是死锁？产生死锁的条件是什么？

死锁是指有两个或多个线程由于竞争资源而造成阻塞的现象，如果无外力作用 (例如关机)，这种局面就会一直持续下去，极度耗费系统资源，时间长了还有可能会导致设备硬件损坏

产生死锁的四个必要条件：

* 互斥条件：一个资源每次只能被一个线程所使用
* 请求与保持条件：当一个线程因请求资源而被阻塞时，对已获得的资源继续持有而不释放
* 不剥夺条件：线程已获得的资源，在未使用完之前，不能被其他线程强行剥夺
* 循环等待条件：若干线程之间形成一种头尾相接的循环等待资源关系 (闭环)

这四个条件是死锁的必要条件，只要发生死锁，这些条件必然成立；而只要上述条件有一个不满足，就不会发生死锁

### ThreadLocal 的原理？

在 Java 的 Thread 类中，有一个 threadLocals 字段，其类型是 ThreadLocalMap

ThreadLocalMap 是一个采用数组实现的 key-value 键值对集合

通过 ThreadLocal 保存的数据会被保存在每个线程的 threadLocals 字段中，以当前 ThreadLocal 对象为 key 进行保存

原理如下图所示：

![](https://note.youdao.com/yws/api/personal/file/WEBbe4680977743e6705e86596ccc3605e8?method=download&shareKey=0468cb3981e820d06a12c68baaf01a3b)

### ThreadLocal 发生内存泄漏的原因？

由于 ThreadLocalMap 中的键值对以 ThreadLocal 对象为 key，且通过弱引用 ThreadLocal 对象

当 ThreadLocal 对象不存在任何强引用时，ThreadLocal 对象势必会被 GC 所回收，这就会导致 ThreadLocalMap 中的键值对的 key 变为 null

即使键值对的 key 变为 null，但键值对依旧强引用着 value；但由于 key 为 null，就无法再通过 key 访问到 value

这样 ThreadLocalMap 中的键值对的 value 就存在着一条强引用链：

Thread --> ThreaLocalMap --> Entry --> value

只要线程不退出，value 就无法被 GC 所回收，造成内存泄漏

只有当线程退出后，value 的强引用链才会断掉

### synchronized 和 ThreadLocal 的区别？

ThreadLocal 和 synchronized 都用于解决线程安全问题，可 ThreadLocal 与 synchronized 有着本质的差别

synchronized 是利用锁机制，使变量在某一时间段内只能被一个线程访问

而 ThreadLocal 为每个线程都提供了变量的副本，使得每个线程所访问到的并非同一个对象，这样就隔离了多个线程对数据的数据共享

## JVM

### 什么是 GC ？

GC 是垃圾回收的意思

对于大部分 Java 对象而言，其内存分配在堆当中，其是否存活由 JVM 进行管理

当堆中的对象越来越多时，其未分配内存就会越来越小，甚至可能会导致没有足够的内存分配给新对象，出现 OOM

于是 JVM 设计了一套用于回收堆中对象的机制，这就是 GC

### 简述 GC 机制？

JVM 将堆内存分为了新生代和老年代两部分 (新生代 : 老年代 ≈ 1 : 2) (分代模型)

又将新生代内存分为了 Eden、From 和 To 三部分 (Eden : From : To ≈ 8 : 1 : 1)

所有新创建的对象都被分配在新生代中的 Eden 区域

每当发生 GC 时，垃圾回收器会通过可达性分析算法判断堆中的对象是否应该被回收

对于新生代，垃圾回收器会把所有存活的对象复制到 From / To 区域，并清空除 From / To 之外的其他区域 (复制算法)

> 当对象每次经过 GC 后仍存活时，JVM 会增加对象的 GC 分代年龄 (指数式增长)
>
> 当对象的 GC 分代年龄达到一定数值时 (这个数值一般是 15)，JVM 就会认为此对象是长期存活的对象，将其移入老年代

对于老年代，垃圾回收器会通过标记-清除算法 / 标记-整理算法回收老年代中的对象

### 介绍一下 GC 中的垃圾回收算法？

#### 复制算法 (Copying)

该算法将可用内存按容量大小划分为了大小相等的两个区域，每次只分配其中一个区域的内存

当这个区域的内存用完时，就将还存活着的对象复制到另一个区域中，并把已分配的内存空间一次性清空

这样使得每次都只是对整个半区进行内存回收，内存分配时也就不用考虑内存碎片等复杂情况，只要按顺序分配内存即可

只是这种算法的代价是将可用内存缩小为了原来的一半

由于此算法实现简单、运行高效，因此常应用于回收新生代对象，且 HotSpot 虚拟机针对复制算法进行了优化

> 专门研究表明，98% 的新生代对象都是“朝生夕死”的，因此并不需要按照 1 : 1 的比例来划分内存空间
>
> HotSpot 虚拟机将堆内存分成了一块较大的 Eden 区域和两块较小的 Survivor 区域，每次只分配 Eden 和其中一块 Survivor 区域的内存
>
> 当发生 GC 时，会将 Eden 和 Survivor 中还存活着的对象复制到另一块 Survivor 区域中，并清空 Eden 和已分配内存的 Survivor 区域
>
> HotSpot 虚拟机默认 Eden 和 Survivor 的大小比例是 8 : 1，也就是说新生代中的可用内存空间是整个新生代内存容量的 90% (80% + 10%)，只有 10% 的内存会被“浪费”，大大提升了内存可用率

#### 标记-清除算法 (Mark-Sweep)

该算法首先会标记出所有需要回收的对象，在标记完成后统一回收所有被标记的对象

该算法的最大不足之处在于：标记-清除后会产生大量不连续的内存碎片

#### 标记-整理算法 (Mark-Compact)

该算法首先会标记出所有需要回收的对象

当标记完成后，将所有存活的对象都朝着某一端移动，并清空边界之外的区域

标记-整理算法虽然不会产生内存碎片，但需要移动对象，因此效率偏低

### 介绍一下可达性分析算法？

该算法的主要思路是将 GC Roots 对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链 (Reference Chain)

当一个对象与 GC Roots 没有任何引用链相连时，则说明此对象已经不可用了，需要被回收

GC Roots 对象包含以下几种：

* 虚拟机栈 (栈帧中的本地变量表) 中引用的对象

* 方法区中类静态属性引用的对象

* 方法区中常量引用的对象

* 本地方法栈中 JNI (即 Native 函数) 引用的对象

* JVM 的内部引用 (例如：Class 对象，异常对象 NullPointException、OutofMemoryError，系统类加载器)

* 所有被同步锁 (synchronized) 持有的对象

* JVM 内部的 JMXBean、JVMTI 中注册的回调、本地代码缓存等

* JVM 实现中的“临时性”对象，跨代引用的对象

### 什么是类加载机制？

当 Class 文件由类装载器装载后，在 JVM 中会形成一个描述 Class 结构的元信息，通过该元信息可以获知 Class 的结构信息：如构造函数，属性和方法等

Java 允许用户通过调用 Class 对象来访问这个 Class 所对应的元信息

JVM 把描述类的数据从 Class 文件加载到内存，并对数据进行校验，转换解析和初始化，最终形成可以被用户直接使用的 Java 类型，这就是类加载机制

### 类加载机制中的双亲委派机制是什么？为什么需要这个机制？

双亲委派机制是：

> 当一个类加载器收到了类加载的请求时，它不会直接去加载指定类，而是把这个请求委托给自己的父加载器去执行，依次向上
>
> 只有当父加载器在它的搜索范围中没有找到目标类时，即无法加载目标类，才会由子加载器来完成类的加载

双亲委派机制有些类似于设计模式中的责任链模式，解决了类加载流程中的先后执行关系

双亲委派机制有以下几个优点：

* 可以避免类的重复加载：当父加载器已经加载过某个类时，子加载器就不用再重新加载这个类了

* 保证了安全性

  例如 java.lang.Object 这个根类，位于 rt.jar 中，无论是哪个类加载器加载此类，最终都是由启动类加载器进行加载，从而保证了安全

  这样即使开发者自己编写了一个 java.lang.Object 类，此类也不会被加载运行，JVM 不会弄混这二者

### 方法调用过程。这个也问的挺多，也看对JVM的学习程度了。



### 线程与进程的内存关系。如一个线程占多少内存、一个进程可以开多少线程、一个进程占用多少内存等。



### 内存分布。JMM、运行时数据区、native内存分布。很看对JVM的理解程度。



# 计算机网络

## Http

### GET 请求和 POST 请求的区别？

* GET 请求一般是用于获取数据 (其实也可以提交，但常见的是获取数据)

  POST 请求一般是用于提交数据

* GET 请求因为会把参数添加到 url 中，所以隐私性、安全性较差，请求的数据长度是有限制的

  不同的浏览器和服务器不同，一般限制在 2~8K 之间，更加常见的是 1k 以内

  而 POST 请求是没有的长度限制，请求数据是放在 body 中

* GET 请求刷新服务器或者回退没有影响，POST 请求回退时会重新提交数据请求

* GET 请求可以被缓存 (这也是 GET 请求用于获取数据的重要原因之一)

  而 POST 请求不会被缓存

* GET 请求会被保存在浏览器历史记录当中，而 POST 请求不会

  GET 请求可以被收藏为书签，而 POST 请求不能

* GET 请求只能进行 url 编码 (appliacation-x-www-form-urlencoded)

  而 POST 请求支持多种编码 (比如 multipart/form-data 等等)

但本质上 GET 和 POST 都是 Http 协议的请求方式，其底层都是 TCP/IP 协议

通常 GET 请求会产生一个 TCP 数据包，POST 请求会产生两个 TCP 数据包

对于 GET 请求，浏览器会把 Http Header 和 Data 一并发送给服务器，服务器响应状态码 200 
表示成功，并返回数据

对于 POST 请求，浏览器会先发送 Http Header，服务器响应状态码 100 表示成功；收到 100 状态码后浏览器再发送 Data，服务器响应状态码 200 表示成功，并返回数据

### Http 的报文结构？

Http 报文分为请求报文和响应报文两种类型，这两种报文都由四部分所组成：起始行、首部行、空行、实体主体

#### 请求报文

* 起始行 (也叫请求行)：由请求方法、请求 URL (不包括域名)、Http 协议版本这三部分构成

  最常见的请求方法是 GET 和 POST

* 首部行 (也叫请求头部)：由关键字 / 值对组成，每行一对

  * User-Agent：产生请求的浏览器类型

  * Accept：客户端希望接受的数据类型

    比如 Accept：application/json 表示希望接受到的是 json 类型

  * Content-Type：客户端发送的实体数据的数据类型
    比如 Content-Type：text/html 表示发送的是 html 类型
  * Host：请求的主机名，允许多个域名同处一个 IP 地址，即虚拟主机
  * Content-Length：客户端发送的实体数据的大小

* 空行：用于标识请求头部结束

* 实体主体 (也叫请求体)：客户端发送给服务端的数据主体

  GET 请求有请求体，POST 请求没有请求体

#### 响应报文

* 起始行 (也叫状态行)：由服务器 Http 协议版本、响应状态码、状态码的文本描述这三部分构成
* 首部行 (也叫响应头部)：由关键字 / 值对组成，每行一对
* 空行：用于标识响应头部结束
* 实体主体 (也叫响应体)：服务端发送给客户端的数据主体

### Http 状态码

状态码是服务端用来告诉客户端，发生了什么事情，状态码位于响应报文的起始行中

大体上状态码的分类如下：

| 整体范围  | 已定义范围 |    分类    |
| :-------: | :--------: | :--------: |
| 100 ~ 199 | 100 ~ 101  |  信息提示  |
| 200 ~ 299 | 200 ~ 206  |    成功    |
| 300 ~ 399 | 300 ~ 305  |   重定向   |
| 400 ~ 499 | 400 ~ 415  | 客户端错误 |
| 500 ~ 599 | 500 ~ 505  | 服务器错误 |

### Http 和 Https 的区别？

- Http 是明文传输，数据都是未加密的，安全性较差

  Https (SSL+HTTP) 数据传输过程是加密的，安全性较好

- 使用 Https 协议需要到 CA (Certificate Authority，数字证书认证机构) 申请证书，一般免费证书较少，因而需要一定费用

- Http 页面响应速度比 Https 快

  主要是因为 Http 使用 TCP 三次握手建立连接，客户端和服务器需要交换 3 个包

  而 Https 除了 TCP 的三个包，还要加上 SSL 握手需要的 9 个包，所以一共是 12 个包

- Http 和 Https 使用的是完全不同的连接方式，用的端口也不一样，前者是 80，后者是 443

- Https 其实就是建构在 SSL / TLS 之上的 Http 协议，因此 Https 比 Http 要更耗费服务器资源

### Https 的详细流程？



### Https 会不会发生中间人攻击？如何预防和解决？



## TCP

### TCP 和 UDP 的区别？

* TCP 面向连接，即发送数据之前需要建立连接

  而 UDP 是无连接的，即发送数据之前不需要建立连接

* TCP 所要求的系统资源比 UDP 所要求的要多

* TCP 向上提供可靠的传输服务

  即通过 TCP 连接传送的数据，无差错、不丢失、不重复，且按序到达

  而 UDP 向上提供不可靠的传输服务，即通过 UDP 连接传送的数据，可能会出现差错、丢失、重复

* TCP 面向字节流传输

  UDP 面向报文传输，并且 UDP 没有拥塞控制，因此网络出现拥塞时源主机的发送速率不会下降

* TCP 只能是一对一点对点的连接通信

  UDP 支持一对一，一对多，多对一和多对多的交互通信

* TCP 的首部开销为 20 字节

  UDP 的首部开销小，只有 8 字节

* TCP 是面向连接的可靠性传输，而 UDP 是不可靠的

|              |                   TCP                    |                     UDP                      |
| :----------: | :--------------------------------------: | :------------------------------------------: |
|   是否连接   |                 面向连接                 |                    无连接                    |
|   是否可靠   |                   可靠                   |                    不可靠                    |
| 连接对象个数 |              只能一对一通信              | 支持一对一，一对多，多对一和多对多的交互通信 |
|   传输方式   |                面向字节流                |                   面向报文                   |
|   首部开销   | 首部开销较大，最小 20 字节，最大 60 字节 |           首部开销较小，仅 8 字节            |
|   应用场景   |    适用于可靠传输的应用，例如传输文件    |         适用于实时应用，例如视频会议         |

### TCP 三次握手的详细过程？为什么不能两次？为什么不能四次？

TCP 是面向连接的协议

所谓的 TCP 三次握手，是指在建立一个 TCP 连接时，需要客户端和服务器总共发送 3 个包

三次握手整体过程如下图所示：

![](https://note.youdao.com/yws/api/personal/file/WEB8f9428e90fc6ed7d71a398cce5cc1e43?method=download&shareKey=731773429e9afd392273bbd8a09b581f)

第一次握手，客户端会给服务器发送一个 SYN 数据包，该包中包含客户端的初始序列号 (seq = x)

第二次握手，服务器会给客户端发送一个 SYN + ACK 数据包，该包中包含服务器的初始序列号 (seq = y)；同时使 ack = x + 1 来表示服务端已确认收到客户端发来的 SYN 数据包

第三次握手，客户端会给服务器发送一个 ACK 数据包，该包中使 ack = y + 1 来表示客户端已确认收到服务器发来的 SYN 数据包

**三次握手是安全建立点对点连接的最小次数**

如果采取四次握手的方式建立连接，那么第四次握手就完全没有必要了，因为经过三次握手就已经可以建立连接了

如果采取两次握手的方式建立连接，就有可能出现下图中的情况：

![](https://note.youdao.com/yws/api/personal/file/WEBf1110c2d370d2e85d4993c16d761db8b?method=download&shareKey=40bc7e9b58877ebf52ec875b26dc19fe)

当客户端发送 TCP 连接请求出现错误，发生了超时或丢包，服务器没有收到此数据包

由于服务器迟迟没有发送回复，客户端自己的超时重传器触发了，客户端重新发送了一个 TCP 连接请求

如果此时服务器接收到了客户端第二次发送的 TCP 连接请求，并给客户端发送了回复，客户端也接收到了服务器发来的回复

由于采用两次握手的方式建立连接，此时客户端与服务器之间就已建立连接，可以通信了

如果当此次连接断开之后，服务器接收到了客户端第一次发送的 TCP 连接请求，就会认为客户端又在请求建立连接，于是就会发送回复给客户端

可此时客户端收到来自服务器的回复后并不会理睬，因为自己并没有发送 TCP 连接请求给服务器

这样就会导致服务器一直处于 TCP 连接已建立状态，并一直等待 TCP 客户端发来数据，这将白白浪费 TCP 服务器的系统资源

因此采用三次握手而不是两次握手建立 TCP 连接，是为了防止已失效的连接请求报文段突然又被传送到了 TCP 服务器，造成 TCP 服务器出现错误

### TCP 四次挥手的详细过程？为什么不能三次？

所谓的 TCP 四次挥手，是指在断开一个 TCP 连接时，需要客户端和服务器总共发送 4 个包

主动进行关闭的一方将执行主动关闭，而另一方则执行被动关闭

四次挥手整体过程如下图所示：

![](https://note.youdao.com/yws/api/personal/file/WEB16878b9ce1db69fcf0397fe39680a7d0?method=download&shareKey=44ad9ae7e90fda15abb2deb6460bc02e)

第一次挥手，客户端会给服务器发送一个 FIN + ACK 数据包，用来关闭客户端到服务器这一方向的数据传输

第二次挥手，服务器会给客户端发送一个 ACK 数据包，同时使 ack = u + 1 来表示服务端已确认收到客户端发来的  FIN 数据包

第三次握手，服务器会给客户端发送一个 FIN + ACK 数据包，用来关闭服务器到客户端这一方向的数据传输

第四次挥手，客户端会给服务器发送一个 ACK 数据包，同时使 ack = w + 1 来表示客户端已确认收到服务端发来的  FIN 数据包

**由于 TCP 连接是全双工的通信，因此每个方向的通信都必须要单独进行关闭**

如果采取三次挥手的方式断开连接，当服务器发送的 FIN 数据包出现丢包时，就会导致客户端一直处于终止等待，等待服务器发送 FIN 数据包 (因为经过三次挥手，服务器已进入关闭状态，不可能再给客户端发送 FIN 数据包)

## DNS

### DNS 解析过程？



# 操作系统

## 进程

### 进程和线程的区别？



### 进程的内存模型 (虚拟内存) ？