---
layout:     post
title:      Android 之 IntentService
subtitle:   读Android源码太难？从简单的 IntentService 开始。
date:       2018-08-09
author:     钟瞻忠
catalog: true
tags:
    - Android
---

>
  不管在日常开发还是面试中，IntentService 都是一个非常重要的知识点。虽然在 RxJava 出现后 IntentService 并没有什么特别的优势了，但是它内部的设计思想还是值得花时间好学习一下的。

<h2>简介</h2>
  众所周知，IntentService 直接继承自 Service，所以使用的时候也需要在 AndroidManifest 中进行注册，因此它也可以认为是四大组件之一。 Android Developers 中对它的介绍如下：
>
  IntentService is a base class for Services that handle asynchronous requests (expressed as Intents) on demand. Clients send requests through Context.startService(Intent) calls; the service is started as needed, handles each Intent in turn using a worker thread, and stops itself when it runs out of work.

  官网就是官网，几句话就将 IntentService 的用处精辟的概括了出来。这段内容我不知如何信达雅的翻译出来，但是敏锐的我把握到了其中的几个关键词，“异步”、“轮流”、“工作线程”、“自动停止”。在后面我们读完源码后，相信大家会对这几个关键词有更深刻的理解。而这几个关键词也给我们提供了使用 IntentService 的最佳指导 ，不管什么场景下，只要符合这几个关键词，都应该想到可以使用 IntentService 来实现功能。  
  基础的东西不愿过多赘述，我们直接来撸源码。
  

<h2>源码</h2>
  在看源码时，可以从总览的角度观察一下整个类的结构，这样后面才不会顾此失彼。一点开 IntentService 的源码，可以直接看到他是直接继承自 Service 的。为了避免被混淆，我们直接将属于 Service 的内容剥离出来，先将这一块给消化了。


```java
public abstract class IntentService extends Service { //直接继承自 Service
    //一个继承自 Handler 的内部类
    private volatile ServiceHandler mServiceHandler;
    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }
        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }
    @Override
    public void onCreate() {
        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();
				mServiceLooper = thread.getLooper();
		    mServiceHandler = new ServiceHandler(mServiceLooper);
	}

	@Override
	public void onStart(@Nullable Intent intent, int startId) {
  	  Message msg = mServiceHandler.obtainMessage();
    	msg.arg1 = startId;
	    msg.obj = intent;
  	  mServiceHandler.sendMessage(msg);
	}

	@Override
	public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
  	  onStart(intent, startId);
    	return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
	}

	@Override
	public void onDestroy() {
  	  mServiceLooper.quit();
	}

	@Override
	@Nullable
	public IBinder onBind(Intent intent) {
  	  return null;
	}

	//继承 IntentService 的时候，需要实现这个方法
	@WorkerThread
	protected abstract void onHandleIntent(@Nullable Intent intent); 
```

  别看这一大段代码好像很多，其实关键点只有两处：

>
  IntentService 直接继承自 Service；  
  IntentService 是一个抽象类，子类需要实现 onHandleIntent 方法；

  所以对我们来说，正常使用 IntentService 只需要实现它的 onHandleIntent 方法即可，参数 Intent 是我们在启动它的时候传递过来的。启动后，就可以让 IntentService 帮我们完成后台的工作。
  
<h2>生命周期</h2>
  对于一个 Service ，我们可以有两种方式去启动它，其一是 startService，还有一种是 bindService。对于 IntentService ，一般都是直接使用 startService。在使用 startService 启动一个全新的 Service 时，它会走 onCreate()--onStartCommand()--onStart() 方法。现在让我们按顺序来分析一下，启动一个 IntentService 时这些方法都干了些什么。
  

<h3>onCreate</h3>
  在 onCreate 中，首先会创建一个 HandlerThread 。而这个 HandlerThread 是个啥呢？不知道各位是否还记得官方简介中的那个关键词“工作线程”，其实这里的 HandlerThread 就是所谓的工作线程。我们可以先看看他的源码。


```java
public class HandlerThread extends Thread {
    Looper mLooper;
    private @Nullable Handler mHandler;
    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }

	public Looper getLooper() {
  	  if (!isAlive()) {
    	    return null;
	    }
  	  synchronized (this) {
    	    while (isAlive() && mLooper == null) {
      	      try {
        	        wait();
          	  } catch (InterruptedException e) {
            	}
	        }
  	  }
    	return mLooper;
	}
}
```

  这段代码也比较简单，只有以下几个看点：

>
  1.直接继承自 Thread；  
  2.在 run 方法中，准备好了一个 Looper，并开始轮询消息；  
  3.提供了返回 Looper 的方法。

  总结一下，这是一个带有 Looper 的 Thread，在线程启动后，会直接准备好这个 Looper 并开始轮询消息。因此，可以得出在 IntentService 的 onCreate 方法中就干了两件事：

>
  1.开启一个子线程用于完成任务;  
  2.准备好用于线程间通信的 Handler ，也就是内部类 ServiceHandler；




<h3>onStartCommand 以及 onStart</h3>
  这两个方法可以一起看，因为 onStartCommand 只干了一件事，就是将 Intent 传递给 onStart。（至于 onStartCommand 方法中的返回值则不用去关心，因为 mRedelivery 默认是 false 的，所以它的返回值可以默认为是 START_REDELIVER_INTENT，这个值的意义也可以直接在源码中看到，简而言之就是在 Service 被系统杀掉时，系统会重启这个服务，在此我们不关心这点。）  
  而在 onStartCommand 中，则是直接构造一个消息，传递给内部类 ServiceHandler 去处理。这里的 Intent 会被放到 obj 中， 后续 ServiceHandler 会将这个取出来传递给 onHandleIntent 去执行任务；而 startId 这个参数则会直接调用 stopSelf 方法去尝试停止这个服务。
    

```java
@Override
public void onStart(@Nullable Intent intent, int startId) {
		Message msg = mServiceHandler.obtainMessage();
		msg.arg1 = startId;
		msg.obj = intent;
		mServiceHandler.sendMessage(msg);
}
```


  在上面说到过，在 IntentService 执行完任务后，会直接调用 stopSelf(int arg) 方法去尝试停止这个服务。为何说是尝试去停止这个服务呢？因为 stopSelf() 跟 stopSelf(int arg) 是有区别的：

>
  stopSelf()：在调用该方法后，系统会立即杀死这个服务；  
  stopSelf(int arg)：在开始一个服务时，会生成一个 startId，如果 stopSelf(int arg) 中的 arg 参数跟最新任务的 startId 一致，那么会立即杀死这个服务；而如果在连续开启两个任务后，使用第一个任务的 startId 去调用 stopSelf(int arg)，这时因为最新的 startId 跟第一个任务的 startId 不一致，所以调用方法后并不会立即杀死服务。

<h3>onDestroy</h3>
  在服务被停止后，会调用 onDestroy 方法，这个方法也很简单，单纯的将在 onCreate 方法中被开启的 Looper 给停下来。
  

<h2>常见问题</h2>
  在日常开发中，对应 IntentService 我们只要会用就够了，但是在面试中，通常会被问到一些平时并不会注意到的知识点。下面我将总结几个面试中常见的问题。
    

<h3>IntentService 与 Service 有何区别？</h3>
  这个问题可以说只要问到了 IntentService 就一定会被问到，因为这确实是两个需要区分开的概念：

>
  1，Service 默认会运行在启动它的那个线程，所以耗时操作仍然需要单独的开启线程去执行，否则可能会出现 ANR，而 IntentService 则默认会开启新线程去执行任务；  
  2，IntentService 是一个抽象类，需要实现其中的抽象方法，也即 onHandleIntent(Intent intent) 方法；  
  3，Service 在启动后，不会主动的停止自己，除非被系统杀死，而 IntentService 在完成任务后则会自动停止自己；

<h3>同时执行多个任务时，IntentService 会如何处理它们？</h3>
  从上面的源码解读中可以发现，那就是所有的任务最终都会通过 Handler 发送消息的形式传递到 ServiceHandler 中去处理，而在短时间内同时启动多个 IntentService 时，onCreate 方法并不会多次执行。所以多次启动后，还是只会有一个 Looper 去轮询消息。因此有多个任务需要处理时，会“轮流”的去处理消息。  
  让我们写一段代码来测试一下：
    
```kotlin
class TestIntentService : IntentService("test") {
    private val TAG = "MY_INTENT_SERVICE"
    override fun onHandleIntent(intent: Intent?) {
        Log.i(TAG, "${intent?.getStringExtra("arg")} is running")
        Thread.sleep(2000)
        Log.i(TAG, "${intent?.getStringExtra("arg")} is finished")
    }
	override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
  	  Log.i(TAG, "${intent?.getStringExtra("arg")} onStartCommand")
    	return super.onStartCommand(intent, flags, startId)
	}

	override fun onDestroy() {
  	  super.onDestroy()
    	Log.d(TAG, "onDestroy")
	}

	override fun onCreate() {
  	  Log.i(TAG, "onCreate")
    	super.onCreate()
	}
}
```

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        mBtn1.setOnClickListener {
            start1()
        }    
	    mBtn2.setOnClickListener {
  	      start2()
    	}
    }


	private fun start1() {
    	val intent1 = Intent(this, TestIntentService::class.java)
  	  intent1.putExtra("arg", "test1")
	    startService(intent1)
	}

	private fun start2() {
  	  val intent2 = Intent(this, TestIntentService::class.java)
	    intent2.putExtra("arg", "test2")
  	  startService(intent2)
	}
}
```
  上述代码的逻辑很简单，有两个 Button，每个 Btn 点击后都会启动一次 IntentService。现在让我们来看看连续执行多个任务时，任务会以串行的方式还是并行的方式执行。  
  下面的截图是只点击 Btn1 的运行结果：
<a href="https://i.loli.net/2019/03/05/5c7e4ceb30ba0.png"><img src="https://i.loli.net/2019/03/05/5c7e4ceb30ba0.png" alt="" /></a>
  可以看到，只点击一个 Btn 时，执行完会立即停止这个服务。下面我们看看在点击 Btn1 之后立即点击 Btn2 的结果：
  
<a href="https://i.loli.net/2019/03/05/5c7e4d3f1edf8.png"><img src="https://i.loli.net/2019/03/05/5c7e4d3f1edf8.png" alt="" /></a>
  而在这种情况下任务2会在任务1完成之后才开始执行，但是 startCommand 方法是立即执行的，所以我们的结论得以验证。

>
  本文作者：ZhanZhong  
  版权声明：本博客所有文章均采用 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/">CC BY-NC-SA 3.0</a>  许可协议，如需转载，请注明出处！
