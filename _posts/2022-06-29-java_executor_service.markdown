---
layout: post
title:  關於java.util.concurrent.ExecutorService - 上篇
date:   2022-06-29 23:11:11 +0800
last_modified_at: 2022-06-29 23:11:11 +0800
cover_image: 108class.png
author: YP Nieh
categories: Java
tags: ExecutorService
---

今天跟大家介紹`java.util.concurrent.ExecutorService`,  這個主題將分別在上下兩篇文章中做介紹. 上篇即是本片文章, 下篇請見[連結](???).

`ExecutorService`是Java 5版本之後推出的執行緒（`Thread`, 亦稱為`綫程`, 本文就直接用英文`thread`稱之）執行機制. 在Java 5之前, 要使用多執行緒（multi-threading）, 必須直接使用Thread相關的APIs, 要注意的細節較多, 屬於較爲低階的APIs.
而Java 5推出的`ExecutorService`則提供較爲高階的APIs, 主要是將要執行的工作(task)與Thread生命周期的管理工作分開, 適合用於大型的專案開發.

## ExecutorService特性

`ExecutorService`有兩個主要的特點: 
- `ExecutorService`幫忙管理Thread物件的建立并且可以重複使用之(亦即利用[thread pool](https://zh.wikipedia.org/wiki/%E7%BA%BF%E7%A8%8B%E6%B1%A0)的方式, pool是儲存池的意思, 將thread物件儲存在池中, 方便重複使用)
- 所執行的工作可以回傳結果, 并且在未來的某個時間, 當結果產生之後, 再取得結果. 是非同步（asynchronous)的執行方式.

## 如何產生ExecutorService的實作物件

我們先看看如何產生`ExecutorService`的實作物件. 最方便的方式是利用`Executors`這個工廠類別(factory class, 注意是複數形式, 加上s). 它提供了許多方便的工廠方法（factory methods)讓我們取得`ExecutorService`的實作物件. 這些物件都使用了Thread pool. 例如: 
- `public static ExecutorService newCachedThreadPool()`
  - Thread pool會隨著需要, 動態增減thread的數量, 適合用在有著許多短暫工作必須同步處理的形況下.
  - 當pool中的thread數量不夠時, 會自動建立新的thread物件, 加入到pool中
  - 當pool中的thread物件空閑時間超過60秒, 就會被消滅
- `public static ExecutorService newFixedThreadPool​(int nThreads)`
  - Thread pool的thread數量采用固定數量模式, 若要執行的工作多于thread的數量時, 工作會在queue中等待
  - 通常使用在有著固定運算資源的環境下, thread數量設成跟硬體環境中的多核CPUs的數量相同. 充分發揮CPUs的運算效能, 但也不過分透支CPUs的運算能力.
- `public static ExecutorService newSingleThreadExecutor()`
  - 僅使用單一一個thread來執行工作. 確保工作是采用循序(sequential)的方式執行, 亦即一次只會有一個工作在進行.
- `public static ScheduledExecutorService newScheduledThreadPool​(int corePoolSize)`
  - Thread pool可以在未來時間排程工作或執行周期性的工作

## 提交給ExecutorService執行的工作的形式

再來看看可以執行的工作包括兩種形式: 
- `java.lang.Runnable`
  - 這是Java 5之前唯一的工作形式. 實作類比必須實作`void run()`方法. 沒有回傳值.
- `java.util.concurrent.Callable<V>`
  - Java 5後提出, 類似`Runnable`, 但是工作可以回傳運算結果, 所以`Callable<V>`是一種Generic type, `V`是結果的型態
  - 須實作`V call() throws Exception`方法, 可以丟出Exception

## 如何清除ExecutorService 

最後在看實際範例前, 我們看看如何在使用完`ExecutorService`後, 清除pool中的threads. 這裏需要注意的是***`ExecutorService`中的threads是屬於`non-daemon thread`.*** 

我們先看什麽是`Daemon thread`. 有人翻譯成`後臺綫程`或`守護綫程`, 通常是用來執行背景工作, 提供服務給一般的threads(稱爲`user threads`, 也就是`non-daemon threads`). 它們的優先順序比較低. Java平臺裏面像是垃圾回收的工作就在`daemon thread`中進行. 如果JVM中僅剩下`daemon threads`, 沒有任何的`non-daemon threads`的話, JVM就會停止執行. 所以在使用完`ExecutorService`後必須將其停止(shutdown), 結束pool中的`non-daemon thread`, 否則application在執行完畢後, 因爲還有`non-daemon thread`在執行, JVM也因此無法中止.

停止`ExecutorService`的動作可以兩種方式進行: 
- `void shutdown()`
  - 啓動shutdown流程, 之前已經送出的工作會繼續執行到完畢. 但是無法再接受新的工作. 
  - 呼叫`shutdown()`會立即回傳,不會等待.
- `List<Runnable> shutdownNow()`
  - 啓動shutdown流程, 正在執行的工作會嘗試中斷之. 在等待執行的工作會停止之, 回傳一list包含所有等待執行的工作
  - 呼叫`shutdownNow()`會立即回傳,不會等待.
  - `shutdownNow()`不保證一定可以中斷正在執行的工作. 它透過Thread.interrupt()去進行中斷, 而工作本身必須偵測中斷訊號(interrupt), 否則無法被順利中斷. 

## 程式範例

好! 我們將基本的知識交代完畢. 讓我們來看程式範例.

首先我們定義`Runnable`實作類別 `ExampleRunnable`

{% highlight java linenos %}
public class ExampleRunnable implements Runnable {

    private final String name;
    private final int len;

    public ExampleRunnable(String name, int len) {
        this.name = name;
        this.len = len;
    }

    @Override
    public void run() {
        System.out.println(name + " thread started!");
        for (int i = 0; i < len; i++) {
            System.out.println(name + ":" + i);
            try {
                Thread.sleep(5); 
            } catch (InterruptedException ex) {
                System.out.println(name + " was interrupted!");
                return;
            }
        }
        System.out.println(name + " thread finished!");
    }
}
{% endhighlight %}

Note:
- 建構子中傳入`name`: thread 名稱, `len`: 列印的數字長度
- `run()`: 呼叫`Thread.sleep()`讓正在執行的thread休息, 停止執行5 msec
- `Thread.sleep()`會偵測中斷訊號(interrupt). 若被中斷, 會丟出`InterruptedException`,然後清除中斷狀態(`interrupted status`). 所以如果它被中斷, run()就會馬上回傳, 該thread即停止執行.

再看main class `ExecutorExample`, 我們先不去做shutdown.

{% highlight java linenos %}
public class ExecutorExample {

    public static void main(String[] args) throws InterruptedException {
        ExecutorService es = Executors.newCachedThreadPool();
        es.execute(new ExampleRunnable("First", 10001));
        es.execute(new ExampleRunnable("Second", 10001));        
        Thread.sleep(10000);
        //es.shutdown(); //先不做shutdown
        //es.execute(new ExampleRunnable("Third", 10001));    //shutdown後, 送出新的工作會丟出RuntimeException
        System.out.println("The main thread finished!");
    }
}
{% endhighlight %}

Note:
- 這裏使用`newCachedThreadPool()`, 會隨需求在pool中產生兩個thread物件來并行執行First與Second. 注意當First與Second執行完畢後, 若沒有新的工作進來, 這兩個沒事做的thread物件會在60秒後, 被消滅掉.
- 在`NetBeans`中執行, First與Second完成工作後, 因爲沒有將`ExecutorService`做shutdown,  `ExecutorService`中的`non-daemon thread`沒有被消滅, 造成JVM無法即刻中止. 如下圖所示, First與Second完成工作後, Console視窗顯示application仍然繼續執行.

![JVM無法即刻中止](/classroom/assets/images/post220629/post220629_1.PNG)

我們可以看一下JVM的threads實際的執行歷程狀況(利用`NetBeans`的`Profile工具`), 如下圖:

![Profile_threads_1](/classroom/assets/images/post220629/post220629_2.PNG)
[點此開啓新視窗檢視](/classroom/assets/images/post220629/post220629_2.PNG){:target="_blank"}

Note:
- `pool-1-thread-1` 與 `pool-1-thread-2` 即為pool中的兩個threads, 用來執行First與Second. 
- `main`為main thread(執行main method)
- `DestroyJavaVM` thread會在main thread結束後開始運作, 其主要角色是會等待JVM中所有`non-daemon threads`都完成執行工作後, 然後將JVM停止. 我們提及過, 由於我們使用`newCachedThreadPool()`, 在完成First與Second工作之後, 若沒有新的工作進來, 這pool中兩個沒事做的thread物件會在60秒後, 被消滅掉. 於是JVM也就會被 `DestroyJavaVM` thread消滅.

讓我們取消第8行的comment, 執行`shutdown()`. 在完成First與Second工作後, `ExecutorService`被shutdown, `DestroyJavaVM` thread立即消滅JVM. 如下圖所示:

![Profile_threads_1](/classroom/assets/images/post220629/post220629_3.PNG)
[點此開啓新視窗檢視](/classroom/assets/images/post220629/post220629_3.PNG){:target="_blank"}

讓我們將第8行改成執行`shutdownNow()`. 在`Thread.sleep(10000)`後便呼叫`shutdownNow()`,這會提早將First與Second工作中斷. 然後Pool中的threads被消滅, JVM也立即被 `DestroyJavaVM` thread消滅. 如下圖所示:

![Profile_threads_1](/classroom/assets/images/post220629/post220629_4.PNG)
[點此開啓新視窗檢視](/classroom/assets/images/post220629/post220629_4.PNG){:target="_blank"}

另外取消第9行的comment,測試一下在shutdown之後送出新的工作, 會丟出`RuntimeException` - `RejectedExecutionException`:

```
Exception in thread "main" java.util.concurrent.RejectedExecutionException: Task com.example.ExampleRunnable@dd3b207 rejected from java.util.concurrent.ThreadPoolExecutor@551bdc27[Shutting down, pool size = 2, active threads = 2, queued tasks = 0, completed tasks = 0]
	at java.base/java.util.concurrent.ThreadPoolExecutor$AbortPolicy.rejectedExecution(ThreadPoolExecutor.java:2055)
	at java.base/java.util.concurrent.ThreadPoolExecutor.reject(ThreadPoolExecutor.java:825)
	at java.base/java.util.concurrent.ThreadPoolExecutor.execute(ThreadPoolExecutor.java:1355)
	at com.example.ExecutorExample.main(ExecutorExample.java:14)
 ```

上篇到此結束.
