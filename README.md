

# 前言


<img align="center" width="390" height="844" src="https://github.com/htaiwan/flutter_push_notification_remote/blob/main/assets/1.jpg">

<img align="center" width="360" height="640" src="https://github.com/htaiwan/flutter_push_notification_remote/blob/main/assets/2.jpg">


在整合flutter的推播時，查了很多線上flutter推播教學，發現都講得有點亂，不然就是內容已經有點過時了。最後還是仔細地看過官方文件跟範例才完成，推播的整合不只是單單只有寫程式，還是有很多細碎的設定要完成，所以要寫出一個完整的教學的確不容易，連官方文件其實都寫得有點雜亂。

我覺得在進行這些設定前，應該要先宏觀的了解為什麼要進行這個設定，不然就陷入一片森林中，不知道自己在做什麼，大多的教學也都是一股腦就開始進行設定，連官方文件也是這樣，所以我下面用一張圖來說明，如何將整個推波串連起來。

# 宏觀

![3](https://github.com/htaiwan/flutter_push_notification_remote/blob/main/assets/3.png)

我認為整個設定過程中，最重要的就是要這張圖中所有相關的部分連接起來。

1. Firebase跟Flutter app產生連接

- 在Firebase創建新專案
- 透過Firebase SDK安裝跟整合，讓flutter app跟Firebase專案產生連接

2. Firebase將訊息推到APNs

- 在apple developer的後台中取得一把key，將這把key上傳到Firebase中，讓Firebase跟APNs產生連接

3. APNs如何推到iOS app中

- 透過provisioning profile的設定，開啟app的推播能力，讓APNs可以推送訊息到iOS app中

至於4.5 這兩個步驟，我只能說Firebase不愧是google的產品，這兩個步驟，我完全不用做什麼就已經幫我們打通串好了，感謝google。

了解了宏觀的概念後，接下來步驟，就是利用一張清單列表，來一一檢核我們要完成的細項。

# 步驟

- [ ] Step1: 創建Flutter App project
- [ ] 1.1 [安裝相關package](https://firebase.flutter.dev/docs/messaging/overview#1-add-dependency)(firebase_core, firebase_messaging)
- 此步驟分別會替Flutter中的iOS, android專案安裝對應的Firebase SDK

- [ ] Step2: 創建Firebase project，並整合Firebase SDK到Flutter中iOS跟android專案
- [ ] 2.1 登入[Firebase console](https://console.firebase.google.com/)，新增專案
- [ ] 2.2 在專案中新增iOS Android應用程式，根據指示完成SDK設定
- 利用Xcode開啟Flutter iOS專案(Runner.xcwrokspace)
- 將GoogleService-Info.plist 檔案移到 Xcode 專案的根目錄
- 利用Android Studio開啟Flutter android專案
- 將google-services.json 檔案移到 Android 應用程式的模組根目錄

- [ ] Step4: 設定iOS專案的provisioning profile
- [在iOS專案開啟推播跟背景功能](https://firebase.flutter.dev/docs/messaging/apple-integration/#configuring-your-app)
- [註冊app identifier](https://firebase.flutter.dev/docs/messaging/apple-integration/#2-registering-an-app-identifier)
- [產生provisioning profile](https://firebase.flutter.dev/docs/messaging/apple-integration/#3-generating-a-provisioning-profile)

- [ ] Step5: 整合APNs到Firebase project中

- [取得APNs key](https://firebase.flutter.dev/docs/messaging/apple-integration#1-registering-a-key)

- 將key上傳到Firebase project中iOS app設定頁中

- [ ] Step6: 撰寫Flutter推播相關程式碼

- [permission處理](https://firebase.flutter.dev/docs/messaging/permissions)
- [取得device token](https://pub.dev/documentation/firebase_messaging/latest/firebase_messaging/FirebaseMessaging/getToken.html)
- [處理notification](https://firebase.flutter.dev/docs/messaging/notifications)
- [topic訂閱](https://pub.dev/documentation/firebase_messaging/latest/firebase_messaging/FirebaseMessaging/subscribeToTopic.html)

- [ ] Step7:測試

- 在Firebase中Cloud message裡，可以進行裝置測試，只要取得FCM token就可以測試了

# 細項說明

- permission處理

```dart
Future<void> requestPermissions() async {
setState(() {
_fetching = true;
});
// https://firebase.flutter.dev/docs/messaging/permissions#permission-settings
NotificationSettings settings =
await FirebaseMessaging.instance.requestPermission(
  alert: true,
  badge: true,
  sound: true,
);

setState(() {
  _requested = true;
  _fetching = false;
  _settings = settings;
});
}
```

- 取得device token

```dart
// vapidKey: 網路推播憑證
// https://console.firebase.google.com/project/flutter-push-notificatio-570d7/settings/cloudmessaging/ios:com.htaiwan.flutterPushNotification
FirebaseMessaging.instance
  .getToken(
    vapidKey:
    'BF8gHR4Z1aCTPzme2UylEZNtcv1ySJfCthZrP-eXGmYHRoPemsI6Syyu29wJrA2vQ0iA6JFmH20vMm45PDY6-vc')
  .then(setToken);

_tokenStream = FirebaseMessaging.instance.onTokenRefresh;
_tokenStream.listen(setToken);


void setToken(String token) {
  print('FCM Token: $token');
  setState(() {
  _token = token;
  });
}

```

- 處理notification

```dart
// case1: If the application is opened from a terminated state
RemoteMessage initialMessage =
await FirebaseMessaging.instance.getInitialMessage();

if (initialMessage != null) {
  Navigator.pushNamed(
  context,
  '/message',
  arguments: MessageArguments(initialMessage, true),
  );
}

// case2: the application is opened from a background state
FirebaseMessaging.onMessageOpenedApp.listen((RemoteMessage message) {
// 在背景時，點擊推播後的處理
Navigator.pushNamed(
  context,
  '/message',
  arguments: MessageArguments(message, true),
  );
});

// case3: application in foreground
FirebaseMessaging.onMessage.listen((RemoteMessage message) {
// Andorid在前景時，推播訊息預設是不會出現，但可以利用local notification來顯示
RemoteNotification notification = message.notification;
AndroidNotification android = message.notification?.android;
if (notification != null && android != null) {
  flutterLocalNotificationsPlugin.show(
  notification.hashCode,
  notification.title,
  notification.body,
  NotificationDetails(
  android: AndroidNotificationDetails(
  channel.id,
  channel.name,
  channel.description,
  // TODO add a proper drawable resource to android, for now using
  //      one that already exists in example app.
  icon: 'launch_background',
  ),
));

```



- topic訂閱

```dart

Future<void> onActionSelected(String value) async {
switch (value) {
case 'subscribe':
  {
  print(
 	 'FlutterFire Messaging Example: Subscribing to topic "fcm_test".');
  await FirebaseMessaging.instance.subscribeToTopic('fcm_test');
  print(
  	'FlutterFire Messaging Example: Subscribing to topic "fcm_test" successful.');
  }
break;
case 'unsubscribe':
  {
  print(
  	'FlutterFire Messaging Example: Unsubscribing from topic "fcm_test".');
  await FirebaseMessaging.instance.unsubscribeFromTopic('fcm_test');
  print(
 	 'FlutterFire Messaging Example: Unsubscribing from topic "fcm_test" successful.');
  }
break;
case 'get_apns_token':
  {
  if (defaultTargetPlatform == TargetPlatform.iOS ||
  defaultTargetPlatform == TargetPlatform.macOS) {
  print('FlutterFire Messaging Example: Getting APNs token...');
  String token = await FirebaseMessaging.instance.getAPNSToken();
  print('FlutterFire Messaging Example: Got APNs token: $token');
  } else {
  print(
  'FlutterFire Messaging Example: Getting an APNs token is only supported on iOS and macOS platforms.');
  }
  }
break;
default:
break;
}
}

```



測試畫面

<img src="https://github.com/htaiwan/flutter_push_notification_remote/blob/main/assets/4.png" alt="4" style="zoom:50%;" />



# 參考資料

[官方範例](https://github.com/FirebaseExtended/flutterfire/tree/master/packages/firebase_messaging/firebase_messaging/example)

[官方文件](https://firebase.flutter.dev/docs/messaging/overview)

