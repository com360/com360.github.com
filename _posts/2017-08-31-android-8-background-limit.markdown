# Android8.0 后台执行限制

Android 8.0 奥利奥对后台app的执行有一定的限制，这些限制都是为了更好的用户体验，和iOS的限制有些接近。

####

应用在两个方面受到限制：

后台Service的限制：

应用处于空闲状态时，应用被限制使用后台服务。 

广播限制：

除了一些例外情况，应用无法使用manifest注册隐式广播。 它们仍然可以在运行时注册隐式广播，并且可以使用清单注册专门针对它们的显式广播。


## 后台服务限制

系统可以区分 前台 和 后台 应用。

前后台的定义基本上遵循下面的这些情况的例外，如果没有则处于后台。

如果满足以下任意条件，应用将被视为处于前台：

* 具有可见 Activity，不管Activity处于started或者paused状态。
* 具有前台服务。
* 另一个前台应用（一般是系统服务）已关联到该应用（不管是通过绑定一个服务，还是通过使用一个content provider）。 例如，如果另一个应用绑定到该应用的如下情况服务，那么该应	用处于前台：
	* 输入法（IME） 
	* 壁纸服务（Wallpaper service）
	* 通知侦听（Notification listener
）
	* 语音或文本服务（Voice or text service）

如果以上条件均不满足，应用将被视为处于后台。

应用在前台的时候，创建、运行前台服务和后台服务都没有限制。

当应用进入后台后，将会有几分钟的窗口期允许应用创建和运行前后台服务，当窗口期结束后，应用将会被进入空闲状态，此时系统会把后台服务停止运行，类似于应用自己调用```Service.stopSelf()``` 方法。

在特定情况下，后台app会被加入到临时白名单中一段时间，当在白名单时，应用可以无限制的启动service，后台service被允许运行。

用于处理对用户可见的任务时，应用将会被加入到白名单中，比如下面的：
* 处理高优先级的推送消息（FCM，系统提供的推送服务）
* 接收短信彩信的广播
* 通知栏中执行PendingIntent
* VPN应用让自己唤起到前台之前启动VpnServce。

在8.0之前，创建一个后台服务是为了把这个服务拉起到前台。8.0已经不允许后台app创建后台服务了，至此，8.0提供了```Context.startForgroundService()```方法启动一个新的前台服务，系统创建了这个服务后，app将有5秒钟的时间调用```Service.startForeground()```方法显示新服务用户可见的通知。如果5秒内没有调用这个方法，那么这儿服务将会被停止，并且报ANR。

### 风险
  * 如果不改，并且在后台跑了服务将会带来什么后果？
  
  需要些demo进行验证

### 适配

* 创建一个后台服务拉到前台，改为使用```NotificationManager.startServiceInForeground()```方法
* ```Context.startService()```都建议改为```NotificationManager.startServiceInForeground()```
* 有规律的或者计划的服务，可以考虑改为```JobScheduler ```
* 发生网络事件时，请使用 FCM 选择性地唤醒您的应用，而不是在后台轮询。
* 推迟后台工作到前台时执行。

以上可能需要分版本进行处理。

## 广播限制

不允许在menifest中注册隐式的广播。隐式广播一般是这对所有应用的。

### 风险
  * 如果不改，menifest中注册隐式的广播（不包括例外）将会带来什么后果？
  
  需要些demo进行验证

### 适配

* menifest里隐式广播改为``` Context.registerReceiver()```方式注册，此种方式不受限制。
* 有规律的任务，可以考虑改为```JobScheduler ```
* 例外
	* 需要签名权限的广播不受此限制所限，因为这些广播只会发送到使用相同证书签名的应用，而不是发送到设备上的所有应用。
	* ACTION_LOCKED_BOOT_COMPLETED, ACTION_BOOT_COMPLETED等，参见[Implicit Broadcast Exceptions](https://developer.android.com/guide/components/broadcast-exceptions.html).


#### 参考
[Background Execution Limits](https://developer.android.com/about/versions/oreo/background.html#migration)



