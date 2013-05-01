---
layout: post
title: "android service startup"
date: 2012-10-05 10:25
comments: true
categories: Android
---

现在要写一个service，在启动之后开启socket监听，等待用户端发消息给它进行处理。而这个service是不会有Activity的，于是乎就要考虑如何让这个service启动起来。

#### System Service Startup

最早想的是将这个service写成system service，然后通过SystemServer启动，这样是可行的，也蛮方便，具体的做法是这样的：

在framework/base/core/java/android/os/app/目录中写一个NewService类extends Service，之后在SystemServer.java中添加

{% codeblock lang:java %}
try {

    Slog.i(TAG, "New Service");

    ServiceManager.addService("newservice", new NewService(context));

} catch (Throwable e) {

    Slog.e(TAG, "Failure starting New Service", e);

}
{% endcodeblock %}

之后NewService就会开机启动，但是这样有一个问题，就是这个NewService的pid也就是SystemServer的pid，它们是处于同一个进程中的。于是由于种种原因我不希望启动一个系统服务，而是一个application level的service，这样就遇到一个问题：如果是一个单纯的没有activity的service应用，在通过*adb install*进系统后并不会自动启动，而是需要其它进程通过*startService()*或者*bindService()*启动，那么有没有什么其它方法让一个service在每次install之后启动呢？

<!-- more -->

#### App Level Service Startup upon Installe

这里有两种方案：

* 1 通过先启动一个Activity，然后*startService()*启动该service，之后在Activity中将自己finish掉；
* 2 在application中注册一个BroadcastReceiver，监听PackageManager的*android.intent.action.PACKAGE_ADDED*的intent-filter，然后在Receiver的onReceive()里面通过

{% codeblock lang:java %}
Intent serviceIntent =new Intent(context, NewService.class);
context.startService(serviceIntent);
{% endcodeblock %}

进行启动。

在我看来第二种方案更符合我的要求，因为这样我就可以不用每次点击一下某个Activity才能启动service，但是这也是我最先否决的方案，因为我在Stack Overflow里面看到好多个讨论这个问题的帖子，其中有一个回复是这么说的：

{% blockquote %}
all applications, upon installation, are placed in a "stopped" state. This is the same state that the application winds up in after the user force-stops the app from the Settings application. While in this "stopped" state, the application will not run for any reason, except by a manual launch of an activity. Notably, noBroadcastReceviers will be invoked, regardless of the event for which they have registered, until the user runs the app manually.
{% endblockquote %}

也就是说任何app level的应用都不可能在没有得到用户同意的情况下自动启动（特别是在android3.1之后，它们在安装后被置于了“stopped”的状态，只有当用户点击了某个Activity的图标才能启动起来，service也只有通过某个启动的Activity或Service通过startService()或bindService()才能启动起来。

所以第二个方案不可行，下面我们来看第一个方案：

在第一个方案里面，当Activity通过startService()启动service之后，就通过finish()将自己退出。这个时候按照android的specification，通过startService()启动的service是不会退出的（通过bindService()启动的service会随着activity的退出而退出），为了确定这一点，可以通过Settings->Apps->Running Service来查看。

##### startForeground()

另外，为了防止你在background运行的service在low memory的时候被系统强制退出，可以通过startForeground()将该service定义在foreground中，而这个所谓的foreground就是我们平时看到的位于android上方的消息栏，里面可以设置该service需要显示的信息，以及时间间隔...具体可以参见[Service API](http://ytliu.github.com "Service API")，这里有一个简单的示例：

{% codeblock lang:java %}
Notification note=new Notification(R.drawable.stat_notify_chat,
	"Can you hear the music?",
	System.currentTimeMillis());
Intent i=new Intent(this, FakePlayer.class);
i.setFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP|
	Intent.FLAG_ACTIVITY_SINGLE_TOP);
PendingIntent pi=PendingIntent.getActivity(this, 0, i, 0);
note.setLatestEventInfo(this, "Fake Player",
	"Now Playing: \"Ummmm, Nothing\"",
	pi);
note.flags|=Notification.FLAG_NO_CLEAR;
startForeground(1337, note);
{% endcodeblock %}

#### App Level Service Startup when Bootup

这是一种理论中的方法，因为我也没尝试过，具体方法参见[this article](http:// "service startup when bootup")，大概的意思是在application的Android.manifest文件中添加一个user-permission和一个receiver的intent-filter：
{% codeblock lang:xml %}
<application ...> 
    ......
    <receiver android:name=".MyReceiver "android:enabled="true" android:exported="false"]]>            
	<intent-filter>
	    <action android:name="android.intent.action.BOOT_COMPLETED"/>            
	</intent-filter>
    </receiver>
</application>

<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>
{% endcodeblock %}

然后在MyReceiver类里面的onReceive()函数中开启相应的service：
{% codeblock lang:java %}
public MyReceiver extends BroadcastReceiver {   
    @Override    
    public void onReceive(Context context,Intent intent) {     
	Intent serviceIntent = new Intent(context, NewService.class);     
	context.startService(serviceIntent);
    }
}
{% endcodeblock %}

这种方法看上去应该是可行的，但是问题是用户每次装你的应用都要让系统重启一遍，那还有谁会那么好心情去安装你的应用啊？
