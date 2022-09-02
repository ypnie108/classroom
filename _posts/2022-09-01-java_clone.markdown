---
layout: post
title:  關於複製Java物件
date:   2022-09-01 23:11:11 +0800
last_modified_at: 2022-09-01 23:11:11 +0800
cover_image: 108class.png
author: YP Nieh
categories: Java
tags: clone
---

今天跟大家聊聊如何在java中做物件的複製。常常我們會需要建立物件的複製(copy), 例如對於一些可以改變內容(mutable)的物件屬性, 為了避免外部程式可以直接改變這些屬性, 我們要在建構子(constructor)或getter/setter方法中做物件屬性的複製(如果對於這個主題不熟悉的話, 請參考我的這篇[文章]())。我們可以利用幾種方式來做物件複製: 
- `clone()`方法, 
- 複製建構子(copy constructor), 
- 靜態工廠方法(static factory methods)。

## 利用clone()方法做複製

- `clone()`方法定義在`Object`類別中, 它的宣告如下: 

```protected Object clone() throws CloneNotSupportedException```

- 對於要提供複製功能的類別來說, 必須實作`Cloneable`介面, 這是一個標記介面(Marker interface), 並沒有要實作的抽象方法(abstract method)。該類別會繼承`Object`的`clone()`方法。但是因為它的存取等級是protected, 因此如果不做覆蓋(override), 則在使用上只能在本身類別中呼叫。於是我們通常會覆蓋`clone()`方法
- 在覆蓋方法`clone()`中, 呼叫父類別的`clone()`並轉型(cast)成為本身類別的型態
- `Object`的`clone()`只會進行bit內容的複製, 亦即對於基本型態屬性或物件參考屬性做數值的複製, 而物件的內容是不會複製的(所謂的shallow copy), 因此會造成複製的物件與原始物件共用其中的物件屬性。對於immutable物件屬性, 共用並不會產生問題。但是對於mutable物件屬性就不妥當了。因此對於mutable的物件屬性, 我們必須做物件內容的複製(所謂的deep copy)。例如:

{% highlight java linenos %}
public class MyClass implements Cloneable{
    int x;
    String s;
    List aList; //mutable object

    public MyClass(int x, String s, List aList) {
        this.x = x;
        this.s = s;
        this.aList = aList;
    }
    
    @Override
    public MyClass clone() throws CloneNotSupportedException{
        MyClass c1= (MyClass)super.clone();
        c1.aList = new ArrayList(this.aList); //內容的複製(deep copy)
        return c1;
    }
}
{% endhighlight %}

Note:
- Ln 13: 存取等級改成public, 所以可以在外部程式呼叫。回傳值我們改成本身類別的型態(Java 5之後可以使用covariant return types)
- Ln 14: 呼叫父類別的`clone()`並轉型成本身類別的型態
- Ln 15: 對於mutable的物件屬性`aList`, 我們必須產生deep copy(內容的複製), 否則`Object`的`clone()`只會進行shallow copy(物件參考的複製)

{% highlight java linenos %}
public class Test {

    public static void main(String[] args) throws CloneNotSupportedException {
        MyClass c1 = new MyClass(100, "Hello", new ArrayList(List.of("Apple", "Orange")));
        MyClass c2 = c1.clone();
        c2.aList.add("Pear");
        System.out.println("c1.aList="+c1.aList);  //列印 c1.aList=[Apple, Orange]
    }
}
{% endhighlight %}

以上例子中`MyClass`的`clone()`做了物件屬性`aList`的內容複製, 因此所複製的物件與原本物件內的`aList`物件屬性就是不同的物件, 互相獨立.

### clone()的缺點

`clone()`方法在設計上有一個很大的問題, 就是我們大多不能透過抽象型態(介面或抽象類別)的參考去呼叫`clone()`。例如`List` interface, 它並沒有實作`clone()`方法, 因此我們必須使用實作類別的型態, 例如`ArrayList`, 才有`clone()`方法可以呼叫。這就違反了物件導向語言應該盡量使用抽象型態的準則.

另外, `clone()`對於final mutable 物件屬性無法做deep copy。因為在呼叫父類別的`clone()`時, 這些final物件屬性便已經被初始化了, 無法去進行物件內容的複製, 然後再指定給final參考.

### 如何避免物件被複製

有的時候我們希望mutable物件不能夠被複製。如果物件的任何一個父類別已經有實作`Cloneable`, 則我們必須override `clone()`並且丟出`CloneNotSupportedException`。例如:

{% highlight java linenos %}
public Object clone() throws CloneNotSupportedException {
    throw new CloneNotSupportedException();
}
{% endhighlight %}

### 子類別利用clone()破壞父類別的內部狀態的不變性

如果一個類別可以被繼承, 則可能讓有心人士透過繼承的子類別實作`clone()`, 對目標物件做shallow copy。有可能破壞目標物件的內部狀態的不變性(invariants)。例如：

{% highlight java linenos %}
public class MyVulnerableClass {

    private boolean expired = false;
    private List<String> contents = new ArrayList<>(List.of("abc", "def")); //mutable

    public List<String> getContents() {
        return contents;
    }

    public void addContents(String content) {
        if (!expired) {
            contents.add(content);
        } else {
            System.out.println("Your status is expired, so new content can not be added!");
        }
    }

    public void makeExpired() {
        expired = true;
        System.out.println("The status is expired!");
    }

    public static void main(String[] args) throws CloneNotSupportedException {
        MyVulnerableClass c1 = new MyVulnerableClass();
        c1.makeExpired();
        c1.addContents("ghi");
        System.out.println("c1.getContents()=" + c1.getContents());
//
        MyVulnerableClass c2 = new MaliciousSubclass();
        MyVulnerableClass c3 = (MyVulnerableClass) c2.clone();
        c2.makeExpired();
        System.out.println("c2.getContents()=" + c2.getContents());
//
        c3.addContents("jkl");
        System.out.println("c2.getContents()=" + c2.getContents());
//
    }
}
 
public class MaliciousSubclass extends MyVulnerableClass implements Cloneable{
    public MaliciousSubclass clone() throws CloneNotSupportedException{
        return (MaliciousSubclass) super.clone();
    }
}
{% endhighlight %}

Note:
- `MyVulnerableClass`類別的不變性就是當狀態`expired`=true, 則不能更新`contents`
- Ln 29~32: `c3`是`c2`的複製, 但是它們有各自的`expired`屬性, 但是共用了mutable `contents`物件內容, 因此透過`c3.addContents()`來改變了`c2`的`contents`, 破壞了`MyVulnerableClass`類別的不變性

## 利用複製建構子做複製

- 相對`clone()`來說比較容易實作, 將`target`物件的屬性取出來, 傳給一般建構子來建立複製物件
- 同樣針對mutable物件屬性做deep copy

{% highlight java linenos %} 

public class MyClass {

    int x;
    String s;
    List aList;

    public MyClass(int x, String s, List aList) {
        this.x = x;
        this.s = s;
        this.aList = aList;
    }

    //copy constructor
    public MyClass(MyClass target) {
        this(target.x, target.s, new ArrayList(target.aList));
    }

}

public class Test {

    public static void main(String[] args) {
        MyClass c1 = new MyClass(100, "Hello", new ArrayList(List.of("Apple", "Orange")));
        MyClass c2 = new MyClass(c1);
        c2.aList.add("Pear");
        System.out.println("c1.aList="+c1.aList);  //列印 c1.aList=[Apple, Orange]
    }
}
{% endhighlight %}

## 利用靜態工廠方法做複製

- 靜態工廠方法(static factory methods)通常用來替代建構子來建立物件, 這裡也可以傳入要被複製的物件做複製
- 同樣針對mutable物件屬性做deep copy

{% highlight java linenos %} 
public class MyClass {

    int x;
    String s;
    List aList;

    public MyClass(int x, String s, List aList) {
        this.x = x;
        this.s = s;
        this.aList = aList;
    }

    // static factory method
    public static MyClass makeCopy(MyClass target){
        return new MyClass(target.x, target.s, new ArrayList(target.aList));
    }
}

public class Test {

    public static void main(String[] args) {
        MyClass c1 = new MyClass(100, "Hello", new ArrayList(List.of("Apple", "Orange")));
        MyClass c2 = MyClass.makeCopy(c1);
        c2.aList.add("Pear");
        System.out.println("c1.aList="+c1.aList);  //列印 c1.aList=[Apple, Orange]
    }
}
{% endhighlight %}

~end~