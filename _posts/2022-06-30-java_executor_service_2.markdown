---
layout: post
title:  關於java.util.concurrent.ExecutorService - 下篇
date:   2022-06-30 08:11:11 +0800
last_modified_at: 2022-06-30 08:11:11 +0800
cover_image: 108class.png
author: YP Nieh
categories: Java
tags: ExecutorService
---

今天跟大家介紹`java.util.concurrent.ExecutorService`,  這個主題將分上下兩部分做介紹. 本文章是下半部分, 上半部分請見[連結]({{ site.baseurl }}{% post_url 2022-06-29-java_executor_service %}).

## awaitTermination()方法

上半部分我們提到使用`ExecutorService`的`shutdown()`方法, 它會啓動thread pool shutdown流程, thread pool中正在執行的工作會繼續執行到完畢. 但是無法再接受新的工作. 并且`shutdown()`會立即回傳,不會等待. 有的時候我們希望能讓目前的thread等待, 直到所有送出的工作執行完畢, 再開始繼續執行(例如, 取得執行的結果後再繼續). 此時我們可以呼叫`awaitTermination()`方法, 它會使得目前的thread被阻擋(blocked)然後等待. 所等待的時間不會超過`timeout參數`. 如果我們給的等待時間夠長, 可以讓目前的thread等待, 直到所有送出的工作執行完畢, 然後`ExecutorService`完成shutdown作業之後, 再繼續執行. 例如以下的做法:

```
threadPool.shutdown();
try{
  threadPool.awaitTermination(60, TimeUnit.SECONDS);
  //...
} catch (InterruptedException ex) {
  //...
}
```

## 使用Callable<V>做爲工作的型態

上半部分我們使用了`Runnable`做爲工作的型態. 它的`run()`方法沒有回傳結果. 我們有些工作可能是進行有結果的運算, 因此我們需要使用`Callable<V>`, 它的`call()`可以回傳結果, 并且可以丟出Exception. `call()`的結果是在未來當結果產生之後, 再讓我們取得. 因此是`非同步（asynchronous)`的執行方式. 

我們利用`ExecutorService`的`submit()`方法, 將`Callable`物件送去thread pool中執行,  `submit()`方法會馬上回傳一個`Future<V>`物件. 這個`Future<V>`可以在未來當結果產生之後, 讓我們取得結果. 

## 利用Future<V>取得結果

我們可以利用以下幾種方式從`Future<V>`物件取得工作的結果:
- 直接呼叫`Future`物件的`get()`方法取得結果: 但是這個方法會阻擋(block)目前的thread的執行, 直到結果產生. 
- 或者先呼叫`Future`物件的`isDone()`方法: 若其回傳true, 代表結果已經產生, 然後再呼叫`get()`去取得結果. 例如以下的做法:

```
while(!future.isDone()) {
    System.out.println("Calculating...");
    Thread.sleep(300);
}
result = future.get();
```


## 程式範例

首先我們定義`Callable<V>`實作類別 `ExampleCallable`, 它的`call()`計算數字的總和:

{% highlight java linenos %}
public class ExampleCallable implements Callable<Integer> {
    private final String name;
    private final int len;
    private int sum = 0;

    public ExampleCallable(String name, int len) {
        this.name = name;
        this.len = len;
    }

    @Override
    public Integer call() throws InterruptedException {
        for (int i = 0; i < len; i++) {
            System.out.println(name + ":" + i);
            sum += i;
            Thread.sleep(1);
        }
        return sum;
    }
}
{% endhighlight %}

Note:
- 建構子中傳入`name`: thread 名稱, `len`: 計算總和的數字長度
- `call()`: 呼叫`Thread.sleep(1)`讓正在執行的thread休息, 停止執行1 msec
- `Thread.sleep()`會偵測中斷訊號(interrupt). 若被中斷, 會丟出`InterruptedException`, 於是`call()`將Exception往它的call stack下方丟, 該thread即停止執行.

再看main class `ExecutorExample2`:

{% highlight java linenos %}
public class ExecutorExample2 {

    public static void main(String[] args) {
        int cpuCount = Runtime.getRuntime().availableProcessors();
        ExecutorService es = Executors.newFixedThreadPool(cpuCount);
        Future<Integer> f1 = es.submit(new ExampleCallable("First", 1001));
        Future<Integer> f2 = es.submit(new ExampleCallable("Second", 1001));
        try {
            es.shutdown();
            boolean terminated = es.awaitTermination(10, TimeUnit.SECONDS); //return true if this executor terminated and false if the timeout elapsed before termination
            System.out.println(terminated ? "This executor terminated" : "The timeout elapsed before termination");
            if (terminated) {
                Integer result1 = f1.get();
                System.out.println("Result of First: " + result1);
                Integer result2 = f2.get();
                System.out.println("Result of Second: " + result2);
            } else{
                es.shutdownNow();
            }
        } catch (ExecutionException | InterruptedException ex) {
            System.out.println("Exception: " + ex);
        }

    }
}
{% endhighlight %}

Note:
- Ln 5: 這裏使用`newFixedThreadPool(cpuCount)`, 會根據多核cpu的實際數量來產生等數量的threads. 
- Ln 6-7: 產生兩個`Callable`工作並送到thread pool中執行. `ExecutorService`的`submit()`方法會馬上回傳`Future`物件, 而我們在未來可以透過`Future`物件取得`Callable`的`call()`方法的回傳結果. 因此送出工作與取得結果是非同步（asynchronous)的執行方式. 
- Ln 9: 由於我們不會再使用這個thread pool, 所以呼叫`shutdown()`
- Ln 10: 呼叫`awaitTermination()`方法, `timeout參數`(等待時間)設定為10秒鐘, 因爲我們給的等待時間夠長, 可以讓目前的thread被阻擋(blocked)而等待, 直到thread pool完成兩個`Callable`總和運算的工作然後被shutdown之後, 再繼續執行. 
- Ln 11: 我們列印`awaitTermination()`方法的boolean回傳值的意義:
  - 若爲true: 代表目前的thread在還沒等待到10秒鐘前, thread pool就已經完成工作, 然後被shutdown. 
  - 若爲false: 代表目前的thread在等待10秒鐘之後, thread pool工作尚未完成, 當然也還沒被shutdown. 
- Ln 12: 若`terminated`為true, 我們確定工作已經完成, 可以放心的讀取結果. `Future`的`get()`方法會馬上回傳結果. 
- Ln 18: 若`terminated`為false, 我們呼叫`shutdownNow()`方法來立即中斷正在執行的工作. 這裏我們只是假設工作不應該花這麽多時間來執行, 或者我們已經對運算結果沒有興趣, 於是立即中止工作. 
 
第二部分到此結束.
