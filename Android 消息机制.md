Android 消息机制
---
 Android中用Handler进行线程之间的消息传递，Handler是 Android消息机制的上层接口；

消息机制的主要模型：
+ Message：需要传递的消息，可以传递数据

+ MessageQueue：消息队列；通过单链表实现，优势：链表在插入和删除上比较有优势；
	+ 向消息池投递消息(MessageQueue.enqueueMessage)
	+ 取走消息池的消息 (MessageQueue.next)

+ Handler：消息辅助类，主要功能向消息池发送各种消息事件（Handler.sendMessage） 和 处理相应的消息事件（Handler.handleMessage）

+ Looper：不断循环执行(Looper.loop)，从MessageQueue中读取消息，按分发机制 将消息分发给目标处理

##### 消息机制的运行流程
1. 当Handler发送消息时，将会调 用	MessageQueue.enqueueMessage	，向消息队列中添加消息。
2. 当通 过	Looper.loop	开启循环后，会不断地从线程池中读取消息，即调 用	MessageQueue.next
3. 调用目标Handler（即发送该消息的Handler） 的	dispatchMessage	方法传递消息
4. 返回到Handler所在线程，目标Handler 收到消息，调用	handleMessage	方法，接收消息，处理消息。

[Handler ——线程间通信，图解：](http://chuantu.biz/t6/193/1514818188x-1404795856.png)
![enter image description here](https://github.com/kiboooo/-/blob/master/Android%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6.png?raw=true)

### Handler内存泄漏解决办法
> 内存泄漏的原因：
> 在java中非静态内部类和匿名内部类都会隐式持有当前类的外部引用；
>  由于Handler是非静态内部类所以其持有当前Activity的隐式引用（Message持有Handler的引用）；
>  如果Handler没有被释放，其所持有的外部引用也就是Activity也不可能被释放; 导致GC的时候无法回收；

由于知道了导致这种现象的原因，我们可以从两个方面解决：
##### 1. 使用静态内部类+弱引用的形式，解决外部强引用持有的问题；
因为弱引用只要发生GC就可以保证被回收；
示例：
```java
 /** 
     * 创建静态内部类 
     */  
    private static class MyHandler extends Handler{  
        //持有弱引用HandlerActivity,GC回收时会被回收掉.  
        private final WeakReference<MainActivity> mActivty;  
        public MyHandler(MainActivity){  
            mActivty =new WeakReference<MainActivity>(activity);  
        }  
        @Override  
        public void handleMessage(Message msg) {  
            MainActivity activity=mActivty.get();  
            super.handleMessage(msg);  
            if(activity!=null){  
                //执行业务逻辑  
            }  
        }  
    }  
```
##### 2. 重写onDestroy，在外部类的生命周期结束时候清除以该Handler为target的所有Message（包括Callback）
这样可以清空消息队列中，持有的对外部类中Handler的引用；
```java
 @Override  
    protected void onDestroy() {  
        super.onDestroy();  
        myHandler.removeCallbacksAndMessages(null);  
    }  
```
