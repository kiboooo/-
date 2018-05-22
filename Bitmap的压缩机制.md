Bitmap的压缩机制

> 参考：
>
> [Android压缩图片保持不失帧的方法](https://blog.csdn.net/jiaruihua_blog/article/details/12113521)
>
> [Bitmap的6种压缩方式](https://blog.csdn.net/harryweasley/article/details/51955467)

### Bitmap的数据大小

图片的长（width）x 图片的宽（height）x一个像素点所占用对字节数；

下列为常用的图片压缩格式：

![ ](https://img-blog.csdn.net/20160720113139490)

> A：透明度， R：红色，G：绿色，B：蓝色；

#### 格式解析

+ ALPHA_8 : 表示为 8位的 Alpha 使徒，即 A=8， 一个像素点占用一个字节，只有透明度的特性，没有颜色显示；
+ ARGB_4444: 表示16位的 ARGB 的位图；即 A = 4，B = 4，G = 4，R = 4每个颜色的像素占4位，所以一个像素点的大小为16位，占2个字节；
  + **该字段已在 API 13 中废弃，图片质量较差**
+ ARGB_8888：表示32位的ARGB位图；即  A = 8，B = 8，G = 8，R = 8，一个像素点的大小为32位，显色度更高一点，占4个字节；
+ RGB_565 ：表示16位的RGB位图，R=5，G=6，B=5，没有透明度的设置，一个像素点的大小为16位，占2个字节；

### 压缩方式

#### 质量压缩

> 这是一种，不会减少图片的像素，通过改变图片的位深度（字节的长度）和 透明度等，来达到压缩图片的目的，但是由于图片的像素和长宽都不会改变，所以导致所占内存大小就是一样的；
>
> ##### 适用：（不适用于 png 等无损照片的操作）
>
> 传递二进制流的渠道，比如优化上传速度，微信分享图片的拉取，这类接受二进制流数据，但是又有一定的上限；
>
> 缺点： 
>
> 压缩前后的图片，在内存中的大小没有变化；

实现代码:

```java
ByteArrayOutputStream baos = new ByteArrayOutputStream();
//目前 quality 是从 edittext 中获取的；
// 作用就是，通过这个值设定压缩的位深，位深随设定的值的减小而减小
int quality =Integer.valueOf(editText.getText().toString());
//调用 bitmap 的 compress （图片，设定压缩程度，字节输出）
bit.compress(CompressFormat.JPEG, quality, baos);
byte[] bytes = baos.toByteArray();
bm = BitmapFactory.decodeByteArray(bytes, 0, bytes.length);
 //log测试确认
Log.i("wechat", "压缩后图片的大小" + (bm.getByteCount() /1024 / 1024)+ "M宽度为" + bm.getWidth() + "高度为" + bm.getHeight()  + "bytes.length=  " + (bytes.length / 1024) + "KB"+ "quality=" + quality);
```

quality 变化后 位深的相应变化为：

![ ](https://img-blog.csdn.net/20160720132221751)

#### 采样率压缩

> 通过设定的缩放值：options.inSampleSize = n ，说明缩放倍数是 1/n^2，获取的宽高的 1/n 的像素点，从像素点的个数上，减少图片的大小；
>
> 读取策略：
>
> 先读取原图片的宽高，根据宽高和缩放值，再从原图片中获取相应的像素点，最后才把获取完毕的图片加入内存，在获取原图片的宽高的时候，是不会把图片加加入内存的，大大减少了 OOM 的机率；
>
> ##### 缺点：
>
> 压缩过后的图片由于像素点减少，容易失真；

实现代码：

```java
	BitmapFactory.Options newOpts = new 
        	BitmapFactory.Options();
	//设定inJustDecodeBounds 为 ture，保证第一次decodeFile的时候获取的 bitmap 是一个空对象，但是又获取到图片的宽高； 
        newOpts.inJustDecodeBounds = true;  
        Bitmap bitmap = 
           	BitmapFactory.decodeFile(path,newOpts);
	// false ，保证通过压缩方式，把图片按照取样率加载到内存中
        newOpts.inJustDecodeBounds = false;  
        int w = newOpts.outWidth;  
        int h = newOpts.outHeight;  
        //计算出取样率
        newOpts.inSampleSize = be;
        bitmap = BitmapFactory.decodeFile(srcPath, newOpts);  
```

当设定的采样率为 2 时，压缩图片的结果：

![ ](https://img-blog.csdn.net/20160720134004311)

#### 缩放法压缩

> 在 Android中提供了 一个 3*3 矩阵 Matrix对图像进行缩放、旋转、平移、斜切等变换；
>
> 那么在不要求图片的保真程度的情况下，通过 对Matrix中属性值的设定，就就可以达到压缩图片的目的；
>
> matrix.setScale(0.5f, 0.5f);  
>
> bitmap 的长度和宽度分别缩小了一半，图片缩小了 四分之一；

实现代码：

```java
		Matrix matrix = new Matrix();
            matrix.setScale(0.5f, 0.5f);
//bitmap 的长度和宽度分别缩小了一半，图片缩小了 四分之一；
            bm = Bitmap.
                createBitmap(bit, 0, 0, bit.getWidth(),
    	bit.getHeight(), matrix, true);
```

效果：

![ ](https://img-blog.csdn.net/20160720145255625)

#### RGB_565法压缩

> 在舍弃了透明度的同时，由于是16位的像素点设定，比 32位的默认 ARBG_8888 占用内存更少；在不考虑透明度的情况下，可以使用 RGB_565 对 bitmap 图片读取到内存；可以很好保证图片的质量；

实现代码：

```java
	BitmapFactory.Options options2 = 
    	new BitmapFactory.Options();
	//设定优先加载到内存的数据格式
     options2.inPreferredConfig = 
     	Bitmap.Config.RGB_565;
      bm = BitmapFactory.decodeFile(
          Environment.getExternalStorageDirectory()
          .getAbsolutePath()
           + "/DCIM/Camera/test.jpg", options2);
```

压缩效果：

![ ](https://img-blog.csdn.net/20160720145726252)

#### Bitmap.createScaledBitmap自带API

> 通过自带的API调用，直接设定用户希望加载的大小限定（长宽）；
>
> 缺点：
>
> 会导致图片很不清晰；

实现代码：

```java
bm = Bitmap.createScaledBitmap(bit, 150, 150, true);
```

实现效果:

![ ](https://img-blog.csdn.net/20160720150219297)



### 总结：

Bitmap所占用的内存 = 图片长度 x 图片宽度 x 一个像素点占用的字节数。

3个参数，任意减少一个的值，就达到了压缩的效果 ；无论是自定义实现压缩，也都是想方设法的改变这三个属性值；


#### 从源码得知；
Bitmap的四种加载方式：
+ decodeFile ： 从文件系统
+ decodeResource： 资源
	+ 以上两者间接调用了  decodeStream
+ decodeStream： 输入流
+ decodeByteArray： 字节数组

#### 以上四种加载方式都支持 BitmapFactory.Options的参数
#####  inSampleSize （采样率）
+ inSampleSize = 1，，即采样后的图片大小为图片的原始大小。小于1，也按照1来算；
+ 	当inSampleSize>1，即采样后的图片将会缩小，缩放比例为1/(inSampleSize 的二次方）
	+ 1 / ( inSampleSize ^ 2)	

> 官方文档指出：
> `inSampleSize` 的取值应该总是2的指数，如1，2，4，8等。 如果外界传入的 `inSampleSize` 的值不为2的指数，那么系统会向下取整并选择一个 最接近2的指数来代替。比如3，系统会选择2来代替

##### 注意
缩放比的实际值，应该根据原图的宽高信息计算；
计算方式：
图片的宽高的实际大小 / 需要的宽高大小；
分别计算出宽和高的缩放比。
应该取用其中最小的缩放比，避免图片缩放过小，加载时导致发生拉伸现象；

例子：
> ImageView的大小是100×100像素，而图片的原始大小为200×300，那么宽 的缩放比是2，高的缩放比是3。如果最终inSampleSize=2，那么缩放后的图片大小 100×150，仍然合适ImageView。如果inSampleSize=3，那么缩放后的图片大小小 于ImageView所期望的大小，这样图片就会被拉伸而导致模糊。 

#####   inJustDecodeBounds （加载模式控制）
通过设置 `inJustDecodeBounds  = true `；
就可以在加载图片的时候，只解析图片的宽高信息，并不会真正加载图片；
当计算出缩放比后，再设置 `inJustDecodeBounds  = false`,再次重新加载图片即可；

##### 注意:
BitmapFactory获取的图片宽高信息和图片的位置以及程序运行的设备有 关，比如同一张图片放在不同的drawable目录下或者程序运行在不同屏幕密度的设 备上，都可能导致BitmapFactory获取到不同的结果，和Android的资源加载机制 有关。

#### 加载流程

+ ①将`BitmapFactory.Options`的`inJustDecodeBounds`参数设为true并加载图片。
+ ②从`BitmapFactory.Options`中取出图片的原始宽高信息，它们对应于`outWidth`和 `outHeight`参数。
+ ③根据采样率的规则并结合目标View的所需大小计算出采样率`inSampleSize`。
+  ④将`BitmapFactory.Options`的`inJustDecodeBounds`参数设为false，然后重新加载 图片。

### 实例代码如下
```java
public	static	Bitmap	decodeSampledBitmapFromResource(Resources	res,	int	resId,	int	reqWidth,	int	reqHeight){						

	BitmapFactory.Options options =	new	BitmapFactory.Option s();								
	options.inJustDecodeBounds	=	true;				
	//加载图片								
	BitmapFactory.decodeResource(res,resId,options);					
	//计算缩放比	
	options.inSampleSize = calculateInSampleSize(options,req Height,reqWidth);							
	//重新加载图片	
	options.inJustDecodeBounds	=false;	
	return	BitmapFactory.decodeResource(res,resId,options);	
			}
			
private	static int calculateInSampleSize(BitmapFactory.Optio ns	options,	int	reqHeight,	int	reqWidth)	{	
	int	height	=	options.outHeight;	
	int	width	=	options.outWidth;
	int	inSampleSize	=	1;							
	if(height>reqHeight||width>reqWidth){											
	int	halfHeight	=	height/2;											
    int	halfWidth	=	width/2;											
    //计算缩放比，是2的指数
	while((halfHeight/inSampleSize)>=reqHeight&&(halfWidth/inSampleSize)>=reqWidth){
		inSampleSize*=2;	
		}	}
	return	inSampleSize;	}
```
