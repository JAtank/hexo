---
title: Andrid个推（推送）集成
date: 2017-10-31 21:04:43
tags: android
---
项目背景是公司的产品中需要用到推送来从服务器端主动推出，然后app接收，从而完成下一步操作。其实这个功能让app进行循环检测也可以完成，但是这样就造成了以下几个问题：
1. app和服务器开销过大。
2. 并不能进行实时监控。
因为公司以前的产品都在使用个推，所以我也就使用个推了。

----------

个推集成前注意事项：
1. 根据需求进行操作。在我们的产品中不需要个推的推送界面和其他杂七杂八的推送服务，只是简简单单的主动推出，传递数据。所以我们只需要简单的进行必要的集成和封装就行。
2. 在集成过程中。注意区分个推版本：2.9.5.0和之前的版本有很大的不一样，在集成过程中要注意区分。
3. 一定要仔细产看集成文档，必要的时候可以先写个简单的demo。原因就是集成的过程中稍有不注意，就会造成clientId获取不到，进而造成推送获取不到。

----------

具体的集成步骤和方法：
1. 申请个推账号，创建应用（这些在个推官网按照具体步骤即可）
2. 添加maven仓库（没有添加会提示找不到依赖）
	![](http://docs.getui.com/mobile/android/img/asmv_maven.png)
3. 添加依赖
	![](http://docs.getui.com/mobile/android/img/asmv_dep.png)
4. gradle配置（在标红色的区域，填写上你自己应用的信息）
 	![](http://docs.getui.com/mobile/android/img/asmv_param.png)
5. 创建PushServer（用于接收数据）
	<pre><code>
	public class PushService extends GTIntentService {
	    @Override
	    public void onReceiveServicePid(Context context, int i) {

	    }

	    @Override
	    public void onReceiveClientId(Context context, String s) {
			//获取client_id（s 就是client_id的内容）
	        Intent intent_onInited = new Intent(WelcomeActivity.ACTION_ONGETUI_INITED);
	        intent_onInited.putExtra("clientid",s);
	        context.sendBroadcast(intent_onInited);
	    }

	    @Override
	    public void onReceiveMessageData(Context context, GTTransmitMessage gtTransmitMessage) {
			//处理个推信息
	        byte[] payload = gtTransmitMessage.getPayload();
	        if (payload != null) {
	            String json = new String(payload); //个推传递过来的数据
	            Log.d("PushReceiver", "receiver payload= " + json);
	            Intent intent_onReceive = new Intent(WelcomeActivity.ACTION_ONGETUI_MSG);
	            intent_onReceive.putExtra("push_data",json);
	            context.sendBroadcast(intent_onReceive);
	        }
	    }

	    @Override
	    public void onReceiveOnlineState(Context context, boolean b) {

	    }

	    @Override
	    public void onReceiveCommandResult(Context context, GTCmdMessage gtCmdMessage) {

	    	}
	}
	</code></pre>
以上的方法中的内容可以视情况而定。
6. 创建第三方推送服务
	<pre><code>
	public class PushUtilService extends Service {
	    @Override
	    public void onCreate() {
	        super.onCreate();
	        GTServiceManager.getInstance().onCreate(this);
	    }

	    @Override
	    public int onStartCommand(Intent intent, int flags, int startId) {
	        super.onStartCommand(intent, flags, startId);
	        return GTServiceManager.getInstance().onStartCommand(this, intent, flags, startId);
	    }

	    @Override
	    public IBinder onBind(Intent intent) {
	        return GTServiceManager.getInstance().onBind(intent);
	    }

	    @Override
	    public void onDestroy() {
	        super.onDestroy();
	        GTServiceManager.getInstance().onDestroy();
	    }

	    @Override
	    public void onLowMemory() {
	        super.onLowMemory();
	        GTServiceManager.getInstance().onLowMemory();
	    }
	}
	</code></pre>
 这个服务类是很有意思的服务了，其中
![](https://i.imgur.com/cyLLRDf.png)
这个地方当你写完之后是会报错警告的，但是在个推的官方文档上并没提到。原因是在这里的返回值必须是固定的。但是这个报错警告是不影响你整个个推的。
另外，这个类还存在一个问题，这个问题到下边使用的时候再讲。
7.  AndroidManifest.xml文件配置（很关键）
为什么说这里很关键，是因为在你配置了必要的权限管理并且将上边定义的服务也添加了之后，一定不要忘记添加
![](https://i.imgur.com/b3RG705.png)
特别是红框标记的，虽然你在gradle中有配置，但这里一定还要添加上，就像图中添加上即可。
8. 使用（初始化推送和注册PushServer）:
使用的时候在onCreate中注册。但是，当我自己写个demo的时候这种方式是可行的，当我集成进入项目，却发现很难初始化成功，于是又在onResume中重新初始化（重复初始化是不会影响个推的，这点在文档上提到过）。另外，我们上边遗留的问题也在这里揭晓。
	<pre><code>
	PushManager.getInstance().initialize(this.getApplicationContext(), PushUtilService.class);//初始化
    PushManager.getInstance().registerPushIntentService(this, PushService.class);//注册接收数据服务
	</code></pre>
在我们进行初始化的时候，我们的第二个参数就是我们自定义的推送服务类（也就是文档上说的支持的第三方的推送服务），这个参数如果我们传null的时候，也是可行的，大家可以点到初始化的方法中查看![](https://i.imgur.com/i6hJPXI.png)在传null的时候是会调用个推自己的服务的。再进入个推自己的服务类中我们会发现，和我们自己写的很类似。但是，这里可以传null,不代表这个类不需要写，我在不写的时候，确实出现了一些很奇怪的错误。当然也可能是我没有研究透彻，或者哪里写的不对，所以这个问题，大家还是可以自己探索的。当然我们既然写了，在初始化的时候还是添加上。
注意：
![](https://i.imgur.com/FGHtUvp.png)
自定义的服务类请按照文档上的格式在配置文件中进行配置。

----------
好了，以上就是本人在集成过程中的步骤和遇到坑。当然可能很多问题还有其他的途径解决，错误的地方希望大家指正。
