---
layout: post
title:  Java物件防禦複製
date:   2022-09-02 23:11:11 +0800
last_modified_at: 2022-09-02 23:11:11 +0800
cover_image: 108class.png
author: YP Nieh
categories: Java
tags: clone
---

我們在[關於複製Java物件]({{ site.baseurl }}{% post_url 2022-09-01-java_clone %})文章中介紹了物件複製的幾種方式. 這一篇文章就來說說在使用mutable物件時幾種會做複製的時機:
- 方法參數輸入mutable物件
- 方法回傳mutable物件屬性時

這些時機基本上是為了不讓外部程式可以在物件建立之後, 利用傳入的物件參考或回傳的物件參考來改變mutable物件屬性的狀態, 因為這些參考是參考到相同的物件. 因此我們在物件的建構子或setter/getter方法裡面必須做物件的複製(deep copy), 稱為防禦複製(defensive copy).

## 方法參數輸入mutable物件

針對mutable的物件屬性, 無論是在建構子或者`setDueDate()` setter方法中, 如果有傳入物件參考, 則必須先複製物件內容(deep copy)再做指定. 例如, 以下範例的`java.util.Date`型態的`dueDate`物件屬性, 利用`new Date(dueDate.getTime())`產生複製物件, 然後再做指定.

{% highlight java linenos %}
import java.util.Date; //mutable type

public class LibraryBook {
    private int id;
    private String title;
    private String borrower;
    Date dueDate;

    public Date getDueDate() {
        return new Date(dueDate.getTime()); //defensive copy
    }

    public void setDueDate(Date dueDate) {
        this.dueDate = new Date(dueDate.getTime()); //defensive copy
    }

    public LibraryBook(int id, String title, String borrower, Date dueDate) {
        this.id = id;
        this.title = title;
        this.borrower = borrower;
        this.dueDate = new Date(dueDate.getTime()); //defensive copy
    }
}
{% endhighlight %}

## 方法回傳mutable物件屬性時

同樣的在getter方法`getDueDate()`, 也要先做物件複製再回傳這個複製物件的參考.

~end~