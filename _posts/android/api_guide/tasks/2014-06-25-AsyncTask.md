---
layout: post
title: "Android之AsyncTask介绍"
description: "android training"
category: android
tags: [android]
date: 2014-06-25 09:11
---


> 本文介绍AsyncTask，它通常用于执行异步任务。

> **目录**  
> **1**. [AsyncTask介绍](#anchor1)  
> **2**. [AsyncTask使用和示例](#anchor2)  
> **3**. [AsyncTask原理](#anchor3)  



<a name="anchor1"></a>
# AsyncTask介绍

AsyncTask比Handler更轻量级一些，适用于简单的异步处理。

使用AsyncTask时，注意重写以下几个方法：

## 1. doInBackground()
**作用**：执行后台任务。  
**要求**：必须重写！  
**注意**：在doInBackground()中不能进行UI操作！  

## 2. onPreExecute()
**作用**：通常用于执行后台任务开始前的准备动作。在用户调用execute()后，并且在AsyncTask主动执行doInBackground()之前被调用。  
**要求**：选择性重写。如果不重写该函数，默认不执行任何动作  

## 3. onPostExecute()
**作用**：通常用于处理后台动作的返回结果。在AsyncTask主动执行doInBackground()之前被调用。  
**要求**：选择性重写。如果不重写该函数，默认不执行任何动作  

## 4. onProgressUpdate()
**作用**：通常用于执行后台任务执行期间的进度更新。因为doInBackground()中不能操作UI，假如我们想在后台任务处理时显示进度，可以在doInBackground()中调用publishProgress()，而publicProgress()会调用onProgressUpdate()；在onProgressUpdate()中进行UI操作即可。  
**要求**：选择性重写。如果不重写该函数，默认不执行任何动作  

## 5. onCancelled()
**作用**：通常用于执行取消AsyncTask任务时的相关动作。如果客户主动调用cancel()，则会执行onCancelled()；否则(AsyncTask执行完之后正常停止)，则不会调用onCancelled()。  
**要求**：选择性重写。如果不重写该函数，默认不执行任何动作  



<a name="anchor2"></a>
# AsyncTask使用和示例

下面是一个自定义的AsyncTask。

    private class MyTask extends AsyncTask<String, Integer, String> {
        //onPreExecute方法用于在执行后台任务前做一些UI操作
        @Override
        protected void onPreExecute() {
            Log.i(TAG, "onPreExecute");
            textView.setText("loading...");
        }
        
        //doInBackground方法内部执行后台任务,不可在此方法内修改UI
        @Override
        protected String doInBackground(String... params) {
            Log.i(TAG, "doInBackground");
            try {
                for (int i=0; i<6; i++) {
                    Log.d(TAG, "doInBackground: publishProgress="+i);
                    publishProgress(20*i);
                    Thread.sleep(500);
                }

                Log.d(TAG, "doInBackground: return OK!");
                return "OK";
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            Log.d(TAG, "doInBackground: return FAIL!");
            return "FAIL";
        }
        
        //onProgressUpdate方法用于更新进度信息
        @Override
        protected void onProgressUpdate(Integer... progresses) {
            Log.i(TAG, "onProgressUpdate");
            progressBar.setProgress(progresses[0]);
            textView.setText("loading..." + progresses[0] + "%");
        }
        
        //onPostExecute方法用于在执行完后台任务后更新UI,显示结果
        @Override
        protected void onPostExecute(String result) {
            Log.i(TAG, "onPostExecute");
            textView.setText(result);
            
            execute.setEnabled(true);
            cancel.setEnabled(false);
        }
        
        //onCancelled方法用于在取消执行中的任务时更改UI
        @Override
        protected void onCancelled() {
            Log.i(TAG, "onCancelled");
            textView.setText("cancelled");
            progressBar.setProgress(0);
            
            execute.setEnabled(true);
            cancel.setEnabled(false);
        }
    }


说明：  
(01) textView是一个TextView对象，cancel和execute分别是两个Button按钮，而progressBar则是进度条。  
(02) 在任务开始时会通过onPreExecute()更新TextView的显示内容。  
(03) 在任务结束时通过onPreExecute()更新TextView的显示内容。  
(04) 任务执行过程中会通过publishProgress()更新进度条。  
(05) 如果任务被强制取消的话，会将进度条重置为0。 


创建并执行AsyncTask的接口如下：

    mTask = new MyTask();
    mTask.execute("http://www.baidu.com");


取消AsyncTask的接口如下：

    mTask.cancel();

点击查看：[AsyncTask示例完整原理](TODO)




<a name="anchor3"></a>
# AsyncTask原理

下面通过Android4.4.2的AsyncTask源码来对AsyncTask原理进行介绍。

## 1. AsyncTask构造函数

    public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                return postResult(doInBackground(mParams));
            }
        };

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


说明：

(01) mWorker是个WorkerRunnable对象，而WorkerRunnable是Callable的实现类。而在"[线程池中关于Callable的介绍](http://www.cnblogs.com/skywang12345/p/3544116.html)"时，我们说过，Callable类似于Runnable接口，不同之处主要在于Callable能和Future配合使用获取任务的结果，而Runnable不能获取结果！  
(02) mFuture是FutureTask对象，而FutureTask洽洽是Future的实现类。 mWorker和mFuture配合使用，能获取后台任务的结果！  


## 2. execute

execute()的源码如下：

    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }

说明：sDefaultExecutor是线程池对象，而executeOnExecutor()是真正执行后台动作的地方。


下面是线程池相关的代码：

    private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
    private static final int KEEP_ALIVE = 1;

    public static final Executor THREAD_POOL_EXECUTOR
            = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                    TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);

    ...

    public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;

    private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }

说明：该线程池是"通过双向队列实现的串行线程池"。scheduleNext()每次只会执行一个任务。


下面看看executeOnExecutor()的代码：

    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
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

说明：  
(01) executeOnExecutor()首先会对任务的状态进行处理。任务共三种姿态：  
> PENDING: 挂起状态。当AsyncTask被创建时，就进入了PENDING状态。  
> RUNNING: 运行状态。当AsyncTask被执行时，就进入了RUNNING状态。  
> FINISHED: 完成状态。当AsyncTask完成(被客户cancel()或正常运行完毕)时，就进入了FINISHED状态。  

当任务是RUNNING或PENDING状态时，会抛出异常。这就决定了，一个AsyncTask只能被执行一次，即只能对一个AsyncTask调用一次execute()；如果要重新执行任务，则需要新建AsyncTask后再调用execute()。  
(02) 接着，调用onPreExecute()。这也就是任务执行前的准备动作！  
(03) 然后，调用exec.execute(mFuture)。作用是将任务提交到线程池中进行执行。线程池的代码前面已经给出，SerialExecutor中的execute()会执行r.run()任务。r.run()实际上是调用FutureTask中run()方法，而FutureTask的run()方法，则会执行Callable的call()函数，即会执行到mWorker的call()方法。而观察前面mWorker的run()方法，我们会发现它会调用doInBackground()接口，并通过postResult()返回任务执行结果。而postResult()的内容如下：   

    private static final InternalHandler sHandler = new InternalHandler();

    private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = sHandler.obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }

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

说明：postResult()会发送MESSAGE_POST_RESULT给sHandler，而sHandler中会将任务执行结果传递给finish()。以下是finish()的代码：


    private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }

说明：如果是正常执行结束，则调用onPostExecute()方法；否则(异常结束)，则调用onCancelled()方法。

至此，AsyncTask的原理结果完毕！总的来说，就是通过线程池来实现的，AsyncTask的任务会提交到线程池中，执行完后，线程池再返回结果。




