---
layout: post
title:  "Android部分"
date:   2015-6-2 16:14:09 +0800
categories: Android
---

1. Activity 系列问题
1.1 绘制Activity生命周期流程图
![生命周期](http://img.blog.csdn.net/20170221091511386?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQva3N3czAzMjA0MDM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


1.2 介绍下不同场景下Activity生命周期的变化过程

启动Activity： onCreate()--->onStart()--->onResume()，Activity进入运行状态。
Activity退居后台： 当前Activity转到新的Activity界面或按Home键回到主屏： onPause()--->onStop()，进入停滞状态。
Activity返回前台： onRestart()--->onStart()--->onResume()，再次回到运行状态。
Activity退居后台，且系统内存不足， 系统会杀死这个后台状态的Activity，若再次回到这个Activity,则会走onCreate()-->onStart()--->onResume()
锁定屏与解锁屏幕 只会调用onPause()，而不会调用onStop方法，开屏后则调用onResume()
1.3 内存不足时系统会杀掉后台的Activity，若需要进行一些临时状态的保存，在哪个方法进行？
Activity的 onSaveInstanceState() 和 onRestoreInstanceState()并不是生命周期方法，它们不同于 onCreate()、onPause()等生命周期方法，它们并不一定会被触发。当应用遇到意外情况（如：内存不足、用户直接按Home键）由系统销毁一个Activity，onSaveInstanceState() 会被调用。但是当用户主动去销毁一个Activity时，例如在应用中按返回键，onSaveInstanceState()就不会被调用。除非该activity是被用户主动销毁的，通常onSaveInstanceState()只适合用于保存一些临时性的状态，而onPause()适合用于数据的持久化保存。
1.4 onSaveInstanceState()被执行的场景有哪些：

系统不知道你按下HOME后要运行多少其他的程序，自然也不知道activity A是否会被销毁，因此系统都会调用onSaveInstanceState()，让用户有机会保存某些非永久性的数据。以下几种情况的分析都遵循该原则

    当用户按下HOME键时
    长按HOME键，选择运行其他的程序时
    锁屏时
    从activity A中启动一个新的activity时
    屏幕方向切换时 
1.5 介绍Activity的几中启动模式，并简单说说自己的理解或者使用场景
standard：这是默认模式，每次激活Activity时都会创建Activity实例，并放入任务栈中。
singleTop： 如果在任务的栈顶正好存在该Activity的实例，就重用该实例( 会调用实例的 onNewIntent() )，否则就会创建新的实例并放入栈顶，即使栈中已经存在该Activity的实例，只要不在栈顶，都会创建新的实例。
singleTask：如果在栈中已经有该Activity的实例，就重用该实例(会调用实例的 onNewIntent() )。重用时，会让该实例回到栈顶，因此在它上面的实例将会被移出栈。如果栈中不存在该实例，将会创建新的实例放入栈中。
singleInstance：在一个新栈中创建该Activity的实例，并让多个应用共享该栈中的该Activity实例。一旦该模式的Activity实例已经存在于某个栈中，任何应用再激活该Activity时都会重用该栈中的实例( 会调用实例的 onNewIntent() )。其效果相当于多个应用共享一个应用，不管谁激活该 Activity 都会进入同一个应用中。
2. Service系列问题
2.1 注册Service需要注意什么

Service还是运行在主线程当中的，所以如果需要执行一些复杂的逻辑操作，最好在服务的内部手动创建子线程进行处理，否则会出现UI线程被阻塞的问题

2.2 Service与Activity怎么实现通信

方法一：
    添加一个继承Binder的内部类，并添加相应的逻辑方法
    重写Service的onBind方法，返回我们刚刚定义的那个内部类实例
    Activity中创建一个ServiceConnection的匿名内部类，并且重写里面的onServiceConnected方法和onServiceDisconnected方法，这两个方法分别会在活动与服务成功绑定以及解除绑定的时候调用，在onServiceConnected方法中，我们可以得到一个刚才那个service的binder对象，通过对这个binder对象进行向下转型，得到我们那个自定义的Binder实例，有了这个实例，做可以调用这个实例里面的具体方法进行需要的操作了 

方法二 通过BroadCast(广播)的形式 当我们的进度发生变化的时候我们发送一条广播，然后在Activity的注册广播接收器，接收到广播之后更新视图

2.3介绍源码中binder机制
我后面的博客会有后续
2.4 IntentService与Service的区别
IntentService是Service的子类，是一个异步的，会自动停止的服务，很好解决了传统的Service中处理完耗时操作忘记停止并销毁Service的问题

会创建独立的worker线程来处理所有的Intent请求；
会创建独立的worker线程来处理onHandleIntent()方法实现的代码，无需处理多线程问题；
所有请求处理完成后，IntentService会自动停止，无需调用stopSelf()方法停止Service；
为Service的onBind()提供默认实现，返回null；
为Service的onStartCommand提供默认实现，将请求Intent添加到队列中；
IntentService不会阻塞UI线程，而普通Serveice会导致ANR异常
Intentservice若未执行完成上一次的任务，将不会新开一个线程，是等待之前的任务完成后，再执行新的任务，等任务完成后再次调用stopSelf() 

3. Handle系列问题
作者：闭关写代码
链接：https://www.nowcoder.com/discuss/3244
来源：牛客网

3.1 介绍Handle的机制

    Handler通过调用sendmessage方法把消息放在消息队列MessageQueue中，Looper负责把消息从消息队列中取出来，重新再交给Handler进行处理，三者形成一个循环
    通过构建一个消息队列，把所有的Message进行统一的管理，当Message不用了，并不作为垃圾回收，而是放入消息队列中，供下次handler创建消息时候使用，提高了消息对象的复用，减少系统垃圾回收的次数
    每一个线程，都会单独对应的一个looper，这个looper通过ThreadLocal来创建，保证每个线程只创建一个looper，looper初始化后就会调用looper.loop创建一个MessageQueue，这个方法在UI线程初始化的时候就会完成，我们不需要手动创建 
3.2 谈谈对HandlerThread的理解
HandlerThread本质上就是一个普通Thread,只不过内部建立了Looper.
特点：
    HandlerThread将loop转到子线程中处理，说白了就是将分担MainLooper的工作量，降低了主线程的压力，使主界面更流畅。
    开启一个线程起到多个线程的作用。处理任务是串行执行，按消息发送顺序进行处理。
    相比多次使用new Thread(){…}.start()这样的方式节省系统资源。
    但是由于每一个任务都将以队列的方式逐个被执行到，一旦队列中有某个任务执行时间过长，那么就会导致后续的任务都会被延迟处理。
    HandlerThread拥有自己的消息队列，它不会干扰或阻塞UI线程。
    通过设置优先级就可以同步工作顺序的执行，而又不影响UI的初始化；
总结：HandlerThread比较适用于单线程+异步队列的场景，比如IO读写操作，耗时不多而且也不会产生较大的阻塞。对于网络IO操作，HandlerThread并不适合，因为它只有一个线程，还得排队一个一个等着。
4. ListView系列问题
4.1 ListView卡顿的原因与性能优化，越多越好
1.重用converView： 通过复用converview来减少不必要的view的创建，另外Infalte操作会把xml文件实例化成相应的View实例，属于IO操作，是耗时操作。

2.减少findViewById()操作： 将xml文件中的元素封装成viewholder静态类，通过converview的setTag和getTag方法将view与相应的holder对象绑定在一起，避免不必要的findviewbyid操作

3.避免在 getView 方法中做耗时的操作: 例如加载本地 Image 需要载入内存以及解析 Bitmap ，都是比较耗时的操作，如果用户快速滑动listview，会因为getview逻辑过于复杂耗时而造成滑动卡顿现象。用户滑动时候不要加载图片，待滑动完成再加载，可以使用这个第三方库glide

4.Item的布局层次结构尽量简单，避免布局太深或者不必要的重绘

5.尽量能保证 Adapter 的 hasStableIds() 返回 true 这样在 notifyDataSetChanged() 的时候，如果item内容并没有变化，ListView 将不会重新绘制这个 View，达到优化的目的

6.在一些场景中，ScollView内会包含多个ListView，可以把listview的高度写死固定下来
ScollView在快速滑动过程中需要大量计算每一个listview的高度，阻塞了UI线程导致卡顿现象出现，如果我们每一个item的高度都是均匀的，可以通过计算把listview的高度确定下来，避免卡顿现象出现

7.使用 RecycleView 代替listview
 每个item内容的变动，listview都需要去调用notifyDataSetChanged来更新全部的item，太浪费性能了。RecycleView可以实现当个item的局部刷新，并且引入了增加和删除的动态效果，在性能上和定制上都有很大的改善
8.ListView 中元素避免半透明
半透明绘制需要大量乘法计算，在滑动时不停重绘会造成大量的计算，在比较差的机子上会比较卡。 在设计上能不半透明就不不半透明。实在要弄就把在滑动的时候把半透明设置成不透明，滑动完再重新设置成半透明。
9.尽量开启硬件加速
 硬件加速提升巨大，避免使用一些不支持的函数导致含泪关闭某个地方的硬件加速。当然这一条不只是对 ListView
 4.2 怎么实现一个部分更新的 ListView？
 4.3 怎么实现ListView多种布局？
 4.4 ListView与数据库绑定的实现