---
layout: post
title: '一个兼容性强的android摄像头调用方案'
date: 2014-08-03 09:04
comments: true
categories: 
---
####兼容性强的定义
1. 摄像头像素无关
2. 分辨率无关
3. 屏幕方向无关

<!--more-->


####摄像头像素无关
像素无关体现在，无论摄像头的像素几何，我都能获取到相对合适的照片。
假设这里的合适是指：需要的照片尺寸和摄像头获取到的数据尺寸是相吻合的。

```
float scale = expectedHeight / expectedWidth;
List<Camera.Size> pictureSize = params.getSupportedPictureSizes();
for (Camera.Size size:pictureSize)
	if (Math.abs(size.height/size.width - scale)<=0.01) {	
   		params.setPictureSize(size.width, size.height);   		
		break;	
	}
```

这样写，就能保证获取到的照片的长宽比是正确的，不至于被拉伸。
当然` for循环 `里边的条件判断怎么写个人喜欢。

####分辨率无关
这里的分辨率无关，是关于 ` previewSize参数 ` 的设置。假设预览窗体的宽和高分别是 ` viewWidth ` 和 ` viewHeight ` ，我们取最接近的值。

```
List<Camera.Size> sizes = params.getSupportedPreviewSizes();
for (Camera.Size size : sizes) {
	if (Math.abs(size.height-viewHeight)<10 && Math.abs(size.width-viewWidth)<10) {
		params.setPreviewSize(size.width, size.height);
		break;
	}

```

这样就能保证在预览时画面不被拉伸变形。

####屏幕方向无关
主要是设置`DisplayOrientation参数`。
如果是横屏应用，那必须在AndroidManifest.xml中的activity标签中声明` android:screenOrientation="landscape"  `。
如果是竖屏应用，AndroidManifest.xml倒不需要设置，但是需要设置两个参数：

```
params.setRotation(90);
camera.setDisplayOrientation(90);
```
第一个参数是设置输出图像时照片的方向，对应的是`PictureCallback`中的byte[]的图像方向。
第二个参数是设置预览界面的方向，这里图方便都写成90度。

但是注意：

无论参数如何设置，PreviewCallback中，`onPreviewFrame()`中的图像方向必然是横屏的方向，这是没法更改的。

如果需要实时获得正确方向的图像，那就这样做：

```
@Override
public void onPictureTaken(byte[] data, Camera camera) {
	Bitmap bitmap = BitmapFactory.decodeByteArray(data, 0, data.length);
	Matrix matrix = new Matrix();
	matrix.preRotate(90);
	bitmap = Bitmap.createBitmap(bitmap ,0,0, bitmap .getWidth(), bitmap .getHeight(),matrix,true);
};
```
但是强烈建议不要这样做！效率是个大问题。
假设摄像头设置的fps是24，1080p的分辨率，那么一秒钟传输的数据量大约是1080 * 1920 * 8 * 24，具体就不算了，反正是一个非常大的数，还来个旋转，这对于手机硬件资源来说简直是找死。


####视频录制
其实跟前面的获取照片几乎是一样的，只不过是用到了一个MediaRecorder而已。也是一些参数的设置。不再赘述。
