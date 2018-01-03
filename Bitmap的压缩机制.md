Bitmap的压缩机制
---
> 按照一定的采样率去将要显示的图片缩小后，再加载入ImageView；

从源码得知；
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
