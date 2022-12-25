---
title: AsyncTask详解
date: 2017-10-17 19:49:18
tags: Android Framework
---


## AsyncTask详解

### AsyncTask中的参数
AsyncTask是一个抽象类,如果我们想使用它就要自定义一个子类去继承它。继成时我们可以为
AsyncTask制定三个泛型参数:

- Params:调用AsyncTask的excute方法时传入的参数类型，如果该方法传入了多个不同类型的参数，那么就定义为Object。
- Progress:执行AsyncTask时如果需要在前台显示进度条,使用该类型作为进度的单位
- Result:当AsyncTask结束后,如果需要对结果进行返回,该类型作为返回值的类型


如果某一个类型不需要传递具体的参数,那么对应的泛型参数用`Void`代替



### AsyncTask中一些重要的方法

- void onPreExecute():在主线程执行,用于进行一些界面上的初始化操作,eg:显示一个进度条对话框等


- Result doInBackground(Params ... params):在子线程中执行,用于执行耗时任务,相当于Thread中的run()方法
- void onProgressUpdate(Progress... values):在主线程中执行,前提是在`doInBackground`方法中执行了publishProgress(Progress... values),由该方法自动调用`onProgressUpdate`,可以界面上实时显示进度
- void onPostExecute(Result res),在子线程中执行,`doInBackground`方法执行结束后,将返回值传给该方法,负责任务结束后的工作
- void publishProgress(Progress... values),一般在`doInBackground`方法中调用(非必须),将当前进度传递出来,一旦执行了该方法,那么将自动回调onProgressUpdate方法,并将当前进度作为参数传递给它.
- void onCancelled(boolean cancel),在主线程中执行,如果被调用,那么`onPostExecute`方法不会被执行,详见源码解析.



### AsyncTask的基本使用

一个简单的下载的例子

```java
public class AsyncTaskTest extends Activity
{
	private TextView show;

	@Override
	public void onCreate(Bundle savedInstanceState)
	{
		super.onCreate(savedInstanceState);
		setContentView(R.layout.main);
		show = (TextView) findViewById(R.id.show);
	}

	// 此方法为界面的按钮提供事件响应方法,
	public void download(View source) throws MalformedURLException
	{
		DownTask task = new DownTask(this);
		task.execute(new URL("http://www.crazyit.org/ethos.php"));
	}

	class DownTask extends AsyncTask<URL, Integer, String>
	{
		// 可变长的输入参数，与AsyncTask.exucute()对应
		ProgressDialog pdialog;
		// 定义记录已经读取行的数量
		int hasRead = 0;
		Context mContext;

		public DownTask(Context ctx)
		{
			mContext = ctx;
		}

		@Override
		protected String doInBackground(URL... params)
		{
			StringBuilder sb = new StringBuilder();
			try
			{
				URLConnection conn = params[0].openConnection();
				// 打开conn连接对应的输入流，并将它包装成BufferedReader
				BufferedReader br = new BufferedReader(
					new InputStreamReader(conn.getInputStream()
					, "utf-8"));
				String line = null;
				while ((line = br.readLine()) != null)
				{
					sb.append(line + "\n");
					hasRead++;
					publishProgress(hasRead);
				}
				return sb.toString();
			}
			catch (Exception e)
			{
				e.printStackTrace();
			}
			return null;
		}

		@Override
		protected void onPostExecute(String result)
		{
			// 返回HTML页面的内容
			show.setText(result);
			pdialog.dismiss();
		}

		@Override
		protected void onPreExecute()
		{
			pdialog = new ProgressDialog(mContext);
			// 设置对话框的标题
			pdialog.setTitle("任务正在执行中");
			// 设置对话框 显示的内容
			pdialog.setMessage("任务正在执行中，敬请等待...");
			// 设置对话框不能用“取消”按钮关闭
			pdialog.setCancelable(false);
			// 设置该进度条的最大进度值
			pdialog.setMax(202);
			// 设置对话框的进度条风格
			pdialog.setProgressStyle(ProgressDialog.STYLE_HORIZONTAL);
			// 设置对话框的进度条是否显示进度
			pdialog.setIndeterminate(false);
			pdialog.show();
		}

		@Override
		protected void onProgressUpdate(Integer... values)
		{
			// 更新进度
			show.setText("已经读取了【" + values[0] + "】行！");
			pdialog.setProgress(values[0]);
		}
	}
}
```

AsyncTask的启动只需要执行:task.execute(Param ... params);可以想其execute()方法中传入多个Param,代表多个任务,(eg:这个例子中可以传入多个url,那么就可以进行多个的下载),可以同时进行,其内部使用了线程池原理;

### AsyncTask中的注意事项


1. AsyncTask的类必须在主线程进行加载,也就是说第一次访问AsyncTask必须在主线程,在4.1之后的版本中,被系统自动完成,在5.0的代码中,在ActivityThread.main()中,调用了AsyncTask.init()方法来实现在主线程中被加载
2. AsyncTask对象必须在主线程中创建
3. execute()方法必须在UI线程中调用
4. 不要在持续中自己调用onPreExecute(),doInBackground()等方法
5. 一个AsyncTask对象执行一次execute()方法,多次执行会报错;
6. AsyncTask在不同版本上的表现是不一样的,eg:1.6版本之前,串行执行任务;1.6时,采用线程池并行处理任务;3.0开始又采用一个线程来串行执行任务;

### AsyncTask源码解析

这里分析的版本是Andoid4.0/4.2,不同的版本可能稍有不同;

首先从AsyncTask的execute()方法分析

```java
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}
// 额,只有一句话,具体实现在下面;

public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,Params... params) {  
    if (mStatus != Status.PENDING) {  
        switch (mStatus) {  
            case RUNNING:  
                throw new IllegalStateException("Cannot execute task:"  
                        + " the task is already running.");  
            case FINISHED:  
                throw new IllegalStateException("Cannot execute task:"  
                        + " the task has already been executed "  
                        + "(a task can be executed only once)");  
        }  
    }  
    mStatus = Status.RUNNING;  
    onPreExecute();  
    mWorker.mParams = params;  
    exec.execute(mFuture);  
    return this;  
}
```

这里关注的点是:

1. 最后将`mStatus`设置为`RUNNING`,由此也可以得出AsyncTask只能执行一次,否则其execute()方法一进去判断状态,如果已经是`RUNNING`,直接报错
2. 执行了`onPreExecute()`方法,因此证明了onPreExecute()方法会第一个得到执行,当前依然在UI线程，所以我们可以在其中做一些准备工作。
3. 调用了Executor的execute()方法，并将前面初始化的mFuture对象传了进去
4. mWorker.mParams = params,将我们传入的参数赋值给了mWorker.mParams
5. exec.execute(mFuture)


相信大家对与`mWorker`和`mFuture`感到困惑,我们找到这两个类

其在`AsyncTask`中的定义`private final WorkerRunnable<Params, Result> mWorker;`

```java
private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {  
        Params[] mParams;  
} 
```

可以看到是Callable的子类，且包含一个`mParams`用于保存我们传入的参数，下面看初始化mWorker的代码

```java
public AsyncTask() {  
        mWorker = new WorkerRunnable<Params, Result>() {  
          	//重写了call方法,只要mWorker一启动,就执行call方法,在call方法中先执行doInBackground方法
          	//然后将其结果作为postResult的参数传入postResult()中,作为callable接口的返回值,此时依然在子线程中
            public Result call() throws Exception {  
                mTaskInvoked.set(true);  
  				//设置线程优先级:后台线程;
                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);  
                //noinspection unchecked 
                return postResult(doInBackground(mParams));  
            }  
        };  
//….  
          
} 
```

可以看到mWorker在构造方法中完成了初始化，并且因为是一个抽象类，在这里new了一个实现类，实现了call方法，call方法中设置mTaskInvoked=true，且最终调用doInBackground(mParams)方法，并返回Result值作为参数给postResult方法.这基本上就将 中几个重要函数的执行流程描述清楚了.



这里也出现了最重要的方法`doInBackground()`,我们知道`doInBackground()`的返回值是是作为参数传给`onPostExecute(Result res)`,为什么是作为参数传给了`postResult`呢?

其实可以想到:`doInBackground()`方法的执行是在子线程的,而`onPostExecute(Result res)`方法是在主线程的,这里直接传递是不可能的,需要一个中间的桥梁来实现线程的切换,这个桥梁就是`postResult();`

接着往下看,`postResult`中具体做了什么

```java
private Result postResult(Result result) {  
        @SuppressWarnings("unchecked")  
        Message message = sHandler.obtainMessage(MESSAGE_POST_RESULT,  
                new AsyncTaskResult<Result>(this, result));  //第一个参数是当前的task,这个方法是task中的方法
        message.sendToTarget();  
        return result;  
}
```

果然,这里用到了`handler`的消息传递机制,是在子线程中将结果传递出去,结合之前所说的AsyncTask类的加载必须在主线程,那么我们还可以料想到这个主线程中的AsyncTask在初始化时也创建了一个handler,并且重写了handleMessage()方法,在该方法中根据Message.What,来将返回的消息结果传给onPostExecute(Result res),这样整个消息就从子线程传递到主线程了.



是不是上面预想的那样,接着看代码,先看一下`AsyncTaskResult`这个类里有什么,这是一个静态内部类,封装了当前的AsyncTask对象和要返回的结果集,其实这个结果集并没有什么用,因为最终取的还是第一个值 data[0].

```java
private static class AsyncTaskResult<Data> {  
       final AsyncTask mTask;  
       final Data[] mData;  
  
       AsyncTaskResult(AsyncTask task, Data... data) {  
           mTask = task;  
           mData = data;  
       }  
   }
```

接下来看一下AsyncTask中的handler对象在哪里?

```java
private static final InternalHandler sHandler = new InternalHandler(); //AsyncTask中的成员变量

//接下来看一下这个handler是专门处理详细的, 在handleMessage(msg)方法中已经说得很清楚了.
private static class InternalHandler extends Handler {  
        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})  
        @Override  
        public void handleMessage(Message msg) {  
            AsyncTaskResult result = (AsyncTaskResult) msg.obj;  
            switch (msg.what) {  
                case MESSAGE_POST_RESULT:  
                    // There is only one result  
                    result.mTask.finish(result.mData[0]);  
                    break;  
                case MESSAGE_POST_PROGRESS:  
                    result.mTask.onProgressUpdate(result.mData);  
                    break;  
            }  
        }  
}
```

我们先看正常发出的标签 msg.what是`MESSAGE_POST_RESULT`,这里调用了`result.mTask.finish(result.mData[0]);`

```java
private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }
```

看到这里进行了判断,如果该任务被取消了,那么走` onCancelled(result)`的分支,否则,执行`onPostExecute(result)`,最后将状态设置为`最后将状态置为FINISHED`,整个任务就结束了.



mWoker看完了，应该到我们的mFuture了，我们在使用Future的时候,是将其作为参数传入Thread中的,依然实在构造方法中完成mFuture的初始化，将mWorker作为参数，复写了其done方法。done()方法是在其成员变量Callable中的call()方法执行结束之后才执行的回调,此时调用其get()方法不会阻塞主线程,

```java
public AsyncTask() {  
    ...  
        mFuture = new FutureTask<Result>(mWorker) {  
            @Override  
            protected void done() {  
                try {  
                    postResultIfNotInvoked(get());  
                } catch (InterruptedException e) {  
                    android.util.Log.w(LOG_TAG, e);  
                } catch (ExecutionException e) {  
                    throw new RuntimeException("An error occured while executing doInBackground()",  
                            e.getCause());  
                } catch (CancellationException e) {  
                    postResultIfNotInvoked(null);  
                }  
            }  
        };  
}  
```

这里在任务执行结束后调用了`postResultIfNotInvoked(get())`,看一下这个方法是如何实现的

```java
private void postResultIfNotInvoked(Result result) {  
    final boolean wasTaskInvoked = mTaskInvoked.get();  
    if (!wasTaskInvoked) {  
        postResult(result);  
   }  
}
```

如果`mTaskInvoked`不为`true`，则执行`postResult`；但是在`mWorker`初始化时就已经将`mTaskInvoked`为`true`，所以一般这个`postResult()`执行不到。

小总结一下:这里是介绍了两个变量的初始化:分别是`mWorker` 以及`mFuture`,这里`mWorker` 其实是一个`Callable`,`mFuture`其实是一个`FutureTask`

具体使用还请参考《疯狂Java讲义》中的关于Callable和Future的使用吧





好了，到了这里，已经介绍完了execute方法中出现了mWorker和mFurture，不过这里一直是初始化这两个对象的代码，并没有真正的执行。下面我们看真正调用执行的地方。

excute()方法

还记得上面的`execute`中：exec.execute(mFuture),其中`exec`为`executeOnExecutor(sDefaultExecutor, params)`中的`sDefaultExecutor`
下面看这个`sDefaultExecutor`

```java
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;  

public static final Executor SERIAL_EXECUTOR = new SerialExecutor();  


private static class SerialExecutor implements Executor {  
  //维持一个队列,队列里面盛放的是 Runnable,而FutureTask 正好实现了 Runnable 接口;
   final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();  
  
   Runnable mActive;  
  
   public synchronized void execute(final Runnable r) {  
     //差点又被误导了,这里是指创建了一个Runnable,其中的run方法是在线程池被调用的时候执行的,而不是现在;
     //这里只是往上面的队列mTask中提交一个runnable对象,这个runnable对象时什么呢,就是我们所提供的mFuture对象;
            mTasks.offer(new Runnable() {  
              	@override
                public void run() {  
                    try {  
                        r.run();  //这个步骤本身就是一个阻塞的,会经过一段时间,执行过程中可能有新的任务添加进来
                    } finally {  
                        scheduleNext();  
                    }  
                }  
            });  
            if (mActive == null) {  //当前没有要执行的任务,那么就取出队列的第一个任务开始执行;
                scheduleNext();  
            }  
        }  
   
  protected synchronized void scheduleNext() {  
            if ((mActive = mTasks.poll()) != null) {  
                THREAD_POOL_EXECUTOR.execute(mActive);  //此时才会执行其r.run()方法,真正执行的线程池是这个,而上一个SERIAL_EXECUTOR的线程池只负责线程串行的调度;
            }  
        }  
}  
```



可以看到sDefaultExecutor其实为SerialExecutor的一个实例,(SerialExecutor实现了Executor接口,该接口中只有一个void execute(Runnable r)方法)其内部维持一个任务队列;直接看其execute（Runnable runnable）方法，将runnable放入mTasks队尾,但是这里提供的Runnable并不是AsyncTask在构造方法中创建的FutureTask(虽然FutureTask也是一个Runnable),而是自己new了一个Runnable对象,在其run的内部手动调用了FutureTask.run(),最后执行scheduleNext()方法.



这个时候就要说一下其实AsyncTask其实是串行执行任务的,我们向`ArrayDeque<Runnable> mTasks`中添加一个任务(注意这里只是添加并不是执行...),添加完后都会判断mActivie是不是为null,如果此时没有任务在执行,那么就会调用scheduleNext()方法,但是此时不为null的话,就什么也不做,仅仅是添加任务,而当scheduleNext()中执行完一个任务后,其finally中会再次调用scheduleNext()方法,执行下一个任务,这就做到了串行...



## 问题:既然AsyncTask只能execute一次,要线程池干什么?

两个原因:

(一)同一个AsyncTask只能execute()一次,如果代码是

```java
new AsyncTask().execute();
new AsyncTask().execute();
new AsyncTask().execute();
new AsyncTask().execute();
new AsyncTask().execute();
```

那么就会有多个任务了,注意:AsyncTask中的线程池是是`static final`的,这样就会产生多任务了,那么线程池自然也就派上用场了



(二)在3.0之前的版本,如果提交了多个任务,那么其线程池不是串行执行的,而是并行执行的.



## 问题:在3.0之后可以依旧使用任务并发执行吗?

可以,只不过不要调用`new AsyncTask().execute();`而是调用`new AsyncTask().executeOnExecutor(Executor exec,Params... params)`,这时候,我们就可以不使用默认的sDefaultExecutor了,而是我们自己提供一个线程池,实现并发.

我们知道在execute()方法中其实是调用了executeOnExecutor(),而这个方法在3.0之前是没有的,3.0之前直接把execute(),然后线程池就并发执行了,当然了,线程池你也知道,如果你提交的任务数过多,就直接报错的,这也是3.0之前的一个缺点,不可配置.
