---
layout: post
title: "Fresco初体验(二)-Fresco的二次封装"
date: 2018-05-21
description: "Fresco的结构与用法"
tag: Android
---   
## 之前看过一篇文章如何二次封装Fresco，就是在load之前把属性都设置好，仿照这个思路和框架，咱们可以在其基础试一下，这次封装的最终目标是把所有加载过程都包含在一行代码中,下面简单的讲一下封装的思路(大致过程，细节太多讲不完)

## Fresco的二次封装步骤
### 1,定义两个接口，FrescoController和BaseFrescoImageView，在其中定义抽象方法，FrescoController主要用来定义Fresco的一些设置或者加载的方法，BaseFrescoImageView主要设置一些用来得到Fresco的参数的方法
### 2，新建一个FrescoImageView继承于FrescoImageView，实现上面两接口，在FrescoController中定义
```
loadView(String lowUrl, String url, int defaultResID, int failResID,int emptyResID)

loadLocalImage(String path, int defaultRes , int failRes,int emptyResID)
```
### 两个方法，并在FrescoImageView中实现，这里主要讲一下loadView（）的实现过程

```
 try {
            mThumbnailPath = null;
            mThumbnailUrl = url;
            mLowThumbnailUrl = lowUrl;
            mDefaultResID = defaultResID;
            mFailResID = failResID;
            mEmptyResID = emptyResID;
            if (!TextUtils.isEmpty(mThumbnailUrl)) {
                if(defaultResID != 0){
                    this.getHierarchy().setPlaceholderImage(defaultResID);
                }
                if(failResID != 0){
                    this.getHierarchy().setFailureImage(failResID);
                }
                this.setSourceController();

                return;
            }
            if(defaultResID != 0){
                this.getHierarchy().setPlaceholderImage(defaultResID);
            }
            if(failResID != 0){
                this.getHierarchy().setFailureImage(failResID);
            }
            mDefaultResID = mEmptyResID;
            this.setResourceController();
        } catch (OutOfMemoryError e) {
            e.printStackTrace();
        }
```
### 成员变量mThumbnailPath-表示本地路径
### mThumbnailUrl表示-线上路径
### mLowThumbnailUrl表示-低分辨率的图片路径
### mFailResID表示-加载失败的图片
### mEmptyResID表示-空链接时候的图片 
### 先保存下来 如果链接不为空，则把链接和占位图设置进去，如果链接为空，则把空图片资源复制给默认图片mDefaultResID = mEmptyResID;

### 3，定义setSourceController()和setResourceController()方法，setSourceController()方法主要把online数据给填充好，setResourceController()主要针对本地加载的时候或者网上链接加载不出来的时候调用

```
 private void setResourceController() {
        mRequest = (isResize()) ? FrescoFactory.buildImageRequestWithResource(this, reSize) :
                FrescoFactory.buildImageRequestWithResource(this);

        mController = FrescoFactory.buildDraweeController(this);

        this.setController(mController);
    }

private void setSourceController() {
        mRequest = (isResize()) ? FrescoFactory.buildImageRequestWithSource(this, reSize) :
                FrescoFactory.buildImageRequestWithSource(this);

        mLowResRequest = FrescoFactory.buildLowImageRequest(this);

        mController = FrescoFactory.buildDraweeController(this);

        this.setController(mController);
    }
```


### 4，定义FrescoFactory类，用来提供ImageRequest和Controller

```
public static DraweeController buildDraweeController(BaseFrescoImageView fresco){
        return Fresco.newDraweeControllerBuilder()
                .setImageRequest(fresco.getImageRequest())
                .setAutoPlayAnimations(fresco.isAnim())
                .setTapToRetryEnabled(fresco.getTapToRetryEnabled())
                .setLowResImageRequest(fresco.getLowImageRequest())
                .setControllerListener(fresco.getControllerListener())
                .setOldController(fresco.getDraweeController())
                .build();
    }

    public static ImageRequest buildImageRequestWithResource(BaseFrescoImageView fresco){
        return  ImageRequestBuilder.newBuilderWithResourceId(fresco.getDefaultResID())
                .setPostprocessor(fresco.getPostProcessor())
                .setAutoRotateEnabled(fresco.getAutoRotateEnabled())
                .setLocalThumbnailPreviewsEnabled(true)
                .build();
    }

    public static ImageRequest buildImageRequestWithResource(BaseFrescoImageView fresco, Point size){
        return  ImageRequestBuilder.newBuilderWithResourceId(fresco.getDefaultResID())
                .setPostprocessor(fresco.getPostProcessor())
                .setAutoRotateEnabled(fresco.getAutoRotateEnabled())
                .setLocalThumbnailPreviewsEnabled(true)
                .setResizeOptions(new ResizeOptions(size.x, size.y))
                .build();
    }
```

### 5,再定义FrescoHelper类，定义一些load的重载方法，来辅助加载，首先定义最基本的方法

```
 /**
     * @param imageView     图片加载控件
     * @param uri           路径或者URL
     * @param defaultImg    默认图片
     * @param cornerRadius  弧形角度
     * @param isCircle      是否为圆
     * @param loadLocalPath 是否本地资源,如果显示R.drawable.xxx,Path可以为null,前提isCircle为true
     * @param isAnima       是否显示GIF动画
     * @param size          是否再编码
     * @param postprocessor 图像显示处理
     */
    public static void loadFrescoImage(FrescoImageView imageView, String uri, int defaultImg, int failImg, int emptyImg,
                                       int cornerRadius, boolean isCircle, boolean loadLocalPath, boolean isAnima,
                                       Point size, Postprocessor postprocessor, ControllerListener controllerListener) {
        init(imageView, cornerRadius, isCircle, isAnima, size, postprocessor, controllerListener);
        if (loadLocalPath) {
            imageView.loadLocalImage(uri, defaultImg, failImg, emptyImg);
        } else {
            imageView.loadView(uri, defaultImg, failImg, emptyImg);
        }
    }
```
### 之后所有的load方法都调用此方法去加载,在此方法中init把所有初始化参数都设置进去,

```
private static void init(FrescoImageView imageView, int cornerRadius, boolean isCircle, boolean isAnima,
                             Point size, Postprocessor postprocessor, ControllerListener controllerListener) {
        imageView.setAnim(isAnima);
        imageView.setCornerRadius(cornerRadius);
        imageView.setFadeTime(500);
        imageView.setControllerListener(controllerListener);
        if (isCircle)
            imageView.asCircle();
        if (postprocessor != null)
            imageView.setPostProcessor(postprocessor);
        if (size != null) {
            imageView.setResize(size);
        }
    }

```


### 6,然后加载方法的重载,比如：

```
    public static void loadFrescoImage(FrescoImageView imageView, String uri, int defaultImg, int failImg, int emptyImg, int cornerRadius) {
        loadFrescoImage(imageView, uri, defaultImg, failImg, emptyImg, cornerRadius, false, false, true, null, null, null);
    }
```

### 7，还有一种加载大图的方法,

```
  /**
     * 超大图片的就接口
     *
     * @param context   上下玩
     * @param imageView 图片加载控件
     * @param imageUri  图片地址
     * @param defaultId 默认失败图片
     */
    public static void loadBigImage(final Context context, final SubsamplingScaleImageView imageView, String imageUri, final int defaultId) {
        final Uri uri = Uri.parse((imageUri.startsWith("http")) ? imageUri : (imageUri.startsWith("file://")) ? imageUri : "file://" + imageUri);
        final Handler handler = new Handler();
        if (imageUri.startsWith("http")) {
            File file = FrescoHelper.getCache(context, uri);
            if (file != null && file.exists()) {
                imageView.setImage(ImageSource.uri(file.getAbsolutePath()));
            } else {
                FrescoHelper.getFrescoImg(context, imageUri, 0, 0, new LoadFrescoListener() {
                    @Override
                    public void onSuccess(Bitmap bitmap) {
                        handler.post(new Runnable() {
                            @Override
                            public void run() {
                                File file = FrescoHelper.getCache(context, uri);
                                if (file != null && file.exists()) {
                                    imageView.setImage(ImageSource.uri(file.getAbsolutePath()));
                                }
                            }
                        });
                    }

                    @Override
                    public void onFail() {
                        imageView.setImage(ImageSource.resource(defaultId));
                    }
                });
            }
        } else {
            imageView.setImage(ImageSource.uri(imageUri.replace("file://", "")));
        }
    }
```
### SubsamplingScaleImageView是一个加载高清图和长途的库，其控件在加载的时候内存管理比较好，高清图一般都不会造成OOM，其中有  
```
image.setMaxScale(7.0f);
image.setDoubleTapZoomScale(7.0f);
```
### 可以改变其缩放大小.

### 8,可以定义各种各样的Postprocessor，而且网上还有关于这类的库，可以使图片达到各种效果

### 9，xml控件中ImageView替换成我们的FrescoImageView 


```
  <com.example.chenzhe.frescodemo.fresco.FrescoImageView
        android:id="@+id/simpleDraweeView"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>
```
```
FrescoHelper.loadDefaultImage(mImage,url);
```

### [下载地址](https://github.com/a272312970/FrescoLibrary.git)，可下载直接放入项目module









