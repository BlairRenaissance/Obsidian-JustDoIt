本文我想说明的是为什么要使用 handler.post 方法，它和常用的 handler.sendmessage 方法的区别是什么？

其实写的时候我犹豫了要不要把 handler 和 post 的源码搬出来说，会显得更有说服力，但是我看的那些博客都是附带源码说明问题的，结果长篇大论，却还说了个云里雾里，所以我不做搬运工，讲明道理，让大家知道是怎么一回事，然后感兴趣的去自己去看源码，OK。

**1 先看用法 1 之主线程中使用：**

```Java
new Handler().post(new Runnable() {
	@Override
	public void run() {
		  mTest.setText("post");//更新UI
	}
});
```


可以看到，new 了 Runnable 像是开启了一个子线程，但是不然，大家可以看到这儿调用的是 run 方法，而不是 start 方法，因此并不是子线程，其实还是在主线程中，那为什么又要使用 post 方法呢？其实一般不这样用，也没人这样用，并没有多大意义，这就像是在主线程中给主线程 sendmessage，并没有什么意义（我们知道 sendmessage 是子线程为了通知主线程更新 UI 的），主线程是可以直接更新 UI 的。

**2 再看用法 2 之子线程中使用：**

```Java
Handler handler；
new Thread(new Runnable() {
	@Override
	public void run() {
		try {
			Thread.sleep(1000);
			handler.post(new Runnable() {
				@Override
				public void run() {
					mTest.setText("post");//更新UI
				}
			});
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}).start();
```

由上面总结我们知道这儿的 post 并不是新开启的子线程，存在的子线程只有一个，即为 new 的 Thread，那么为什么我们在其中可以 settext 做更新 UI 的操作呢？ 其实 post 方法 post 过去的是一段代码，相当于将这个 Runable 体放入消息队列中，那么 looper 拿取的即为这段代码去交给 handler 来处理，其实也相当于我们常用的下面这段代码：

```Java
private Handler mHandler = new Handler() {
	@Override
	public void handleMessage(Message msg) {
		super.handleMessage(msg);
		switch (msg.what) {
			case 0:
				mTest.setText("handleMessage");//更新UI
				break;
		}
	}
};
```

看起来熟悉吧，就是用这个 Runnable 体代替了上面这一大段代码，当然，我们的 post 方法就可以执行 UI 操作了。

平常情况下我们一个 activity 有好多个子线程，那么我们都会采用上面这种 handleMessage(msg) 方式，然后 case 0：case 1：等等，但是当我们只有一个子线程时呢，用 post 反而比上面一大串代码轻便了不少, 何乐而不为呢？