Android 布局层级优化策略
---
>  常用布局优化xml中的标签类型：
>  < includ > 
>  < merge > 
>  < ViewStub >

#### < inlcude >
该标签开源允许在一个布局当中引入另一个布局。
比如每个页面的顶部的ActionBar，被抽象成一个布局，被每个布局通过 include 导入，实现了代码的一次编写，多次复用，  这样与原生的 ActionBar 相比，功能性更强，灵活度更高；

自定义标题栏：
```java
<?xml version="1.0" encoding="utf-8"?>  
<RelativeLayout 
	xmlns:android="http://schemas.android.com/apk/res/android"  
    android:layout_width="match_parent"  
    android:layout_height="match_parent" >  
  
    <Button  
        android:id="@+id/back"  
        android:layout_width="wrap_content"  
        android:layout_height="wrap_content"  
        android:layout_alignParentLeft="true"  
        android:layout_centerVertical="true"  
        android:text="Back" />  
  
    <TextView  
        android:id="@+id/title"  
        android:layout_width="wrap_content"  
        android:layout_height="wrap_content"  
        android:layout_centerInParent="true"  
        android:text="Title"  
        android:textSize="20sp" />  
  
    <Button  
        android:id="@+id/done"  
        android:layout_width="wrap_content"  
        android:layout_height="wrap_content"  
        android:layout_alignParentRight="true"  
        android:layout_centerVertical="true"  
        android:text="Done" />  
  
</RelativeLayout>
```

![enter image description here](https://img-blog.csdn.net/20150315145044818?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ3VvbGluX2Jsb2c=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

当某个页面需要加入 titlebar 这个功能的时候，只需要调用 include 标签加载该布局文件即可；通过 layout_width 和  layout_height 设定该布局的大小，设定后，便会覆盖原始布局的属性；还可以设定 id ，区分导入的布局；

```java
<?xml version="1.0" encoding="utf-8"?>  
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"  
    android:layout_width="match_parent"  
    android:layout_height="match_parent"  
    android:orientation="vertical" >  
  
    <include  
        android:layout_width="match_parent"  
        android:layout_height="wrap_content"  
        layout="@layout/titlebar" />  
  
    ......  
  
</LinearLayout>
```
![enter image description here](https://img-blog.csdn.net/20150315154256427?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ3VvbGluX2Jsb2c=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### < merge >
作为 < include > 标签的一种辅助扩展来使用。
主要作用：
防止在引用布局文件时产生多余的布局嵌套,即是减少 View 树的层次来优化布局。
因为Android 在解析和展示一个布局是需要消耗时间的，布局嵌套的越多，就会导致解析起来就越来越耗时，性能降低，因此我们在编写布局文件时应该让布局的嵌套越少越好；

该标签的出现，是为了解决某些 include 包含的布局文件，存在一些多余的父布局的嵌套；

当使用该标签去包含要显示的内容，该部分内容在被 include 的时候，会把内容直接填充到 include 位置，也就是说 加入了 include 位置内中的 布局容器中，称为子控件般的存在。

例子：
![enter image description here](https://img-blog.csdn.net/20150315163951320?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ3VvbGluX2Jsb2c=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
把该布局用  merge 标签优化：
```java
优化之前：
<?xml version="1.0" encoding="utf-8"?>  
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"  
    android:layout_width="match_parent"  
    android:layout_height="wrap_content"  
    android:orientation="vertical" >  
  
    <Button  
        android:id="@+id/ok"  
        android:layout_width="match_parent"  
        android:layout_height="wrap_content"  
        android:layout_marginLeft="20dp"  
        android:layout_marginRight="20dp"  
        android:text="OK" />  
  
    <Button  
        android:id="@+id/cancel"  
        android:layout_width="match_parent"  
        android:layout_height="wrap_content"  
        android:layout_marginLeft="20dp"  
        android:layout_marginRight="20dp"  
        android:layout_marginTop="10dp"  
        android:text="Cancel" />  
  
</LinearLayout>  

优化之后：
<?xml version="1.0" encoding="utf-8"?>  
<merge xmlns:android="http://schemas.android.com/apk/res/android">  
  
    <Button  
        android:id="@+id/ok"  
        android:layout_width="match_parent"  
        android:layout_height="wrap_content"  
        android:layout_marginLeft="20dp"  
        android:layout_marginRight="20dp"  
        android:text="OK" />  
  
    <Button  
        android:id="@+id/cancel"  
        android:layout_width="match_parent"  
        android:layout_height="wrap_content"  
        android:layout_marginLeft="20dp"  
        android:layout_marginRight="20dp"  
        android:layout_marginTop="10dp"  
        android:text="Cancel" />  
  
</merge>  
```
布局层级的比较：
优化之前：
![enter image description here](https://img-blog.csdn.net/20150315170730011?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ3VvbGluX2Jsb2c=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

优化之后：
![enter image description here](https://img-blog.csdn.net/20150315173001332?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ3VvbGluX2Jsb2c=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### ViewStub ：仅在需要时才加载的布局
  适用场景：
填写一些用户的表单信息，或者 设置页面的重点设置和全面信息的区分；通常把核心信息优先显示，再设置一个更多信息的按钮，通过该按钮的点击事件设置，再显示全面的信息版面；

第一种实现方法：
通过控制布局的 INVISIBLE 或者 GONE 属性，显示或者隐藏；
该方法在功能实现方面，是没有问题；但是对于布局的加载有一定的压力，因为在布局进行加载解析的时候，尽管你设置了隐藏的属性，但是还需要进行相应的解析过程，无形中浪费了时间，面对复杂布局的时候，还增加了解析的难度；

第二种实现方法：（暂不支持 merge 标签）
适用 轻量级的 View控件 ：  ViewStub ；
它没有大小，无绘制功能，并且不参加布局，所以资源消耗非常低；

例子：
![enter image description here](https://img-blog.csdn.net/20150315222821454?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ3VvbGluX2Jsb2c=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
实现点击 more 按钮后 显示三个输入框：
![enter image description here](https://img-blog.csdn.net/20150315223018752?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ3VvbGluX2Jsb2c=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

```java
<?xml version="1.0" encoding="utf-8"?>  
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"  
    android:layout_width="match_parent"  
    android:layout_height="match_parent"  
    android:orientation="vertical" >  
  
    <EditText  
        android:id="@+id/edit"  
        android:layout_width="match_parent"  
        android:layout_height="wrap_content"  
        android:layout_marginBottom="10dp"  
        android:layout_marginLeft="20dp"  
        android:layout_marginRight="20dp"  
        android:layout_marginTop="10dp"  
        android:hint="@string/edit_something_here" />  
  
    <Button  
        android:id="@+id/more"  
        android:layout_width="wrap_content"  
        android:layout_height="wrap_content"  
        android:layout_gravity="right"  
        android:layout_marginRight="20dp"  
        android:layout_marginBottom="10dp"  
        android:text="More" />  
      
    <ViewStub   
        android:id="@+id/view_stub"  
        android:layout="@layout/profile_extra"  
        android:layout_width="match_parent"  
        android:layout_height="wrap_content"  
        />  
  
    <include layout="@layout/ok_cancel_layout" />  
  
</LinearLayout>  

实现点击事件：
private EditText editExtra1;  
private EditText editExtra2;  
private EditText editExtra3;  
  
public void onMoreClick() {  
    ViewStub viewStub = (ViewStub) findViewById(R.id.view_stub);  
    if (viewStub != null) {  
        View inflatedView = viewStub.inflate();  
        editExtra1 = (EditText) inflatedView.findViewById(R.id.edit_extra1);  
        editExtra2 = (EditText) inflatedView.findViewById(R.id.edit_extra2);  
        editExtra3 = (EditText) inflatedView.findViewById(R.id.edit_extra3);  
    }  
}  
```