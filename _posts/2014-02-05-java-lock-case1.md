---
layout: post
title: 一道有趣的java线程题
category: [并发编程]
---


之前在iteye答复过一道问题和最近总结的东西很应景，今天无意中又看到这道题，把自己当时的处理方式记录一下。

实现magic方法，使输出结果只为1,2，不能输出3
{% highlight java %}
public class Blackhole {      
        public static void enter(Object obj) throws InterruptedException {  
        System.out.println("1");  
        magic(obj);  
        System.out.println("2");  
        synchronized (obj) {  
            System.out.println("3");  
        }  
    }  
  
    private static void magic(Object obj){}  
}  
{% endhighlight %} 

思路：把锁抢走，在没有抢到锁之前要一直阻塞主线程，抢到锁之后不给主线程再抢回锁的机会。

方法1：
{% highlight java %}
public static void magic(final Object lock) {  
        Thread t = new Thread() {  
            @Override  
            public void run() {  
                synchronized (lock) {  
                    while (true) {  
                    }  
                }  
            }  
        };  
        t.start();  
        Thread.State s = t.getState();  
        while (!s.equals(s.RUNNABLE)) {  
            s = t.getState();  
        }  
 }  
{% endhighlight %}

方法2：
{% highlight java %}
public static void magic(final Object lock) {  
        final AtomicInteger change = new AtomicInteger(-1);  
        new Thread() {  
            @Override  
            public void run() {  
                synchronized (lock) {  
                    change.incrementAndGet();  
                    while (true) {  
                    }  
                }  
            }  
        }.start();  
        int flag = change.get();  
        while (flag < 0) {  
            flag = change.get();  
        }  
    }   
{% endhighlight %}  

出这道题作者的答案
{% highlight java %}
private static void magic(final Object obj) throws InterruptedException {  
    Thread thread = new Thread() {  
        @Override  
        public void run() {  
            synchronized (obj) {  
                synchronized (this) {  
                    this.setName("LockNow");  
                    this.notifyAll();  
                }  
                while (true) {  
                }  
            }  
        }  
    };  
    synchronized (thread) {  
        thread.setName("");  
        thread.start();  
        while (thread.getName().equals("")) {  
            thread.wait();  
        }  
    }  
}  
{% endhighlight %}


  



 