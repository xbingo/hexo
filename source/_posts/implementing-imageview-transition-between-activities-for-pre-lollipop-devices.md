---
title: 【译】实现通用的共享图片过场动画
date: 2016-10-04
tags: [transition]
categories: UI
---
> 最近想要对项目里的一些过场动画作体验优化，但是迫于API level的限制，无法直接使用Android原生的特性，刚好可以参考此文实现。
> 
> 原文链接：[Implementing ImageView transition between activities for pre-Lollipop devices.](https://medium.com/@v.danylo/implementing-imageview-transition-between-activities-for-pre-lollipop-devices-8b24bc387a2a#.jagzoke8q)


从Android 5.0开始，谷歌引入了一个很好玩的特性，“共享元素过场(shared elements transition)”。使用它我们可以让我们的UI元素在activity切换时就像用一个很酷的动画从一个activity移动到另一个一样。我们将在Android 5.0以下实现它。文中使用的代码可以在[GitHub](https://github.com/danylovolokh/ImageTransition)上找到。

本文的动画基于Chet Haase的视频：[DevBytes: Custom Activity Animations](https://www.youtube.com/watch?v=CPxkoe2MraA)[](http://savefrom.net/?url=https%3A%2F%2Fwww.youtube.com%2Fwatch%3Fv%3DCPxkoe2MraA&utm_source=chameleon&utm_medium=extensions&utm_campaign=link_modifier "Get a direct link")

本文和Chet的例子不同之处在于，我还会处理`ImageView`的缩放。这会使动画看起来更自然。`ImageView`在前一个界面的`ImageView.ScaleType`跟第二个界面放大的`ImageView`是不一样的。以下是一个示例：
![](http://o6l8fqr7s.bkt.clouddn.com/set_1-1.gif "Shared element transition from one activity to another.")

![](http://o6l8fqr7s.bkt.clouddn.com/set_2-1.gif "Canceling of shared element transition until it’s finished.")

这是它用原生shared element transition实现的样子：
![](http://o6l8fqr7s.bkt.clouddn.com/set_3-1.gif "Shared element transition between activities on Android Lollipop.")


### 使Activity透明

工程里有两个Activity：

- `ImagesListActivity`
- `ImageDetailsActivity`

首先，我们要让`ImagesListActivity`的内容在图片过度的过程中可见。这要改变activity样式中的两个属性：

``` xml
<item name="android:windowBackground">
    @android:color/transparent
</item>
<item name="android:windowIsTranslucent">
    true
</item>
```

现在 `ImageListActivity` 在 `ImageDetailsActivity` 出现在屏幕上的时候也可见了。但是这样做有个主要的缺点：我们最终会通过设置背景颜色让 `ImageDetailsActivity` 完全不透明。这样 `ImageListActivity` 仍然会被 `WindowManger` 绘制，尽管它不可见！

我们的例子中有两个 activity。在 Android 5.0以后，我们可以在动画运行过程中让“第二个activity”透明，然后再把它变得不透明。在这之后"第一个activity"不会再被绘制，并且可以引发垃圾回收。然而，在Lollipop之前的版本不是这样的。

让 `ImageDetailsActivity` 透明也会改变 `ImagesListActivity` 的行为。当 `ImageDetailsActivity` 启动的时候，`ImageListActivity` 只会执行 `onPause()` 回调。

### 进入ImageDetailsActivity的动画

当 `ImagesListActivity` 的图片被点击的时候，会启动 `ImageDetailsActivity`：

``` java
// ImagesListActivity

mImageDetailsImageModel = imageModel;

int[] screenLocation = new int[2];
image.getLocationInWindow(screenLocation);

Intent startIntent = ImageDetailsActivity.getStartIntent(this, 
        imageFile,
        screenLocation[0],
        screenLocation[1],
        image.getWidth(),
        image.getHeight(),
        image.getScaleType());

startActivity(startIntent);
```

被点击图片的一些信息会被传递到 `ImageDetailsActivity`：

* `ImageView` 的左上角位置
* `ImageView` 的宽高
* 图片的 `ScaleType`
* 图片本身，用来在 `ImageDetailsActivity` 中加载图片

另外，`ImageListActivity`中的`imageModel`要保存起来以备后面使用。

在`ImageDetailActivity`中，创建一个跟被点击的图片一样的`ImageView`，添加到跟节点`FrameLayout`（android.R.id.content ——传递到`setContentView()`中的layout的父view）：

``` java
// ImageDetailsActivity

FrameLayout androidContent = (FrameLayout)
        getWindow()
        .getDecorView()
        .findViewById(android.R.id.content);

mTransitionImage = new ImageView(this);
androidContent.addView(mTransitionImage);

Bundle bundle = getIntent().getExtras();

int thumbnailTop = bundle.getInt(KEY_THUMBNAIL_INIT_TOP_POSITION)
        - getStatusBarHeight();
int thumbnailLeft = bundle.getInt(KEY_THUMBNAIL_INIT_LEFT_POSITION);
int thumbnailWidth = bundle.getInt(KEY_THUMBNAIL_INIT_WIDTH);

int thumbnailHeight = bundle.getInt(KEY_THUMBNAIL_INIT_HEIGHT);

ImageView.ScaleType scaleType = (ImageView.ScaleType) 
        bundle.getSerializable(KEY_SCALE_TYPE);

FrameLayout.LayoutParams layoutParams = (FrameLayout.LayoutParams) 
        mTransitionImage.getLayoutParams();

layoutParams.height = thumbnailHeight;
layoutParams.width = thumbnailWidth;
layoutParams.setMargins(thumbnailLeft, thumbnailTop, 0, 0);

File imageFile = (File)      
        getIntent()
        .getSerializableExtra(IMAGE_FILE_KEY);

mTransitionImage.setScaleType(scaleType);

mImageDownloader.load(imageFile).noFade().into(mTransitionImage);
```

我们设置 view 的 margin 来达到新的 view 更之前界面中的位置完全一致。你也许注意到了`mImageDownloader`，我在项目里使用了 Picasso 加载图片。

然后，放大的ImageView要先加载到layout中，然后我们才能开始运行动画：

``` java
// ImageDetailsActivity

mImageDownloader.load(imageFile).into(mEnlargedImage, new Callback() {

       /**
         * 图片加载成功
         */
        @Override
        public void onSuccess() {
            Log.v(TAG, "onSuccess, mEnlargedImage");
            // 在这个回调中，我们已经把图片加载到ImageView中，但是
            // 我们还要等到measure完成以后才能运行动画。我们利用
            // OnPreDrawListener来确保所有的view已经被measure
            if (savedInstanceState == null) {
                // 如果saveInstanceState是null，
                // 证明activity是第一次启动，运行动画
                runEnteringAnimation();
            } else {
                // activity是从后台切换回前台的，只加载图片
            }
        }

        @Override
        public void onError() {
             // 注意：onError没有处理，如果图片加载过程中
             // 出现了OutOfMemory，我们必须在这里处理
            Log.v(TAG, "onError, mEnlargedImage");
        }
    });
}
```

重要的是，只有`savedInstanceState`是null的时候我们才运行动画。否则，就意味着只是从后台切换到前台，或者从其它activity返回。

### 用OnPreDrawListener在正确的时候开始动画

接下来是`runEnteringAnimation()`的代码。这个方法是一个小窍门：设置
[android.view.ViewTreeObserver.OnPreDrawListener](https://developer.android.com/reference/android/view/ViewTreeObserver.OnPreDrawListener.html)。当`onPreDraw()`方法被调用的时候，布局已经测量过了。这意味着我们这是可以使用图片在屏幕上的位置了。以下是我们在`OnPreDrawListener`做的事情：

1. 当第一帧被绘制时我们开始动画。
2. 我们只是单纯地让第二帧绘制。
3. 让之前界面被点击的`ImageView`不可见，并且移除`onPreDrawListener`。我们通过让它不可见使得`ImageDetailsActivity`中复制的`ImageView`就像是直接从它被点击的位置移动过来的一样。并且它移动的时候还要在原来的位置留下一个空洞。

``` java
// ImageDetailsActivity

private void runEnteringAnimation() {
    Log.v(TAG, "runEnteringAnimation, addOnPreDrawListener");

    mEnlargedImage
        .getViewTreeObserver()
        .addOnPreDrawListener(
        new ViewTreeObserver.OnPreDrawListener() {

        int mFrames = 0;

        @Override
        public boolean onPreDraw() {
            // When this method is called we already have everything 
            // laid out and measured so we can start our animation
            Log.v(TAG, "onPreDraw, mFrames " + mFrames);

            switch (mFrames++) {
                case 0:
                    /**
                     * 1\. start animation on first frame
                     */ 
                    final int[] finalLocationOnTheScreen = new int[2];
                 mEnlargedImage.getLocationOnScreen(finalLocationOnTheScreen);

                    mEnterScreenAnimations.playEnteringAnimation(
                            finalLocationOnTheScreen[0], // left
                            finalLocationOnTheScreen[1], // top
                            mEnlargedImage.getWidth(),
                            mEnlargedImage.getHeight());

                    return true;
                case 1:
                    /**
                     * 2\. Do nothing. We just draw this frame
                     */

                    return true;
            }
            /**
             * 3.
             * Make view on previous screen invisible on after this 
             * drawing frame
             * Here we ensure that animated view will be visible 
             * when we make the view behind invisible
             */
            Log.v(TAG, "run, onAnimationStart");
            mBus.post(new ChangeImageThumbnailVisibility(false));
            mEnlargedImage.getViewTreeObserver()
                .removeOnPreDrawListener(this);

            Log.v(TAG, "onPreDraw, << mFrames " + mFrames);

            return true;
        }
    });
}
```

为什么我们要这样处理帧呢？这是因为Android的渲染系统是双缓冲的，我们让系统画两帧来确保变换中的`ImageView`已经可见。相似的技术在SDK中也被使用。如果我们不这样做`ImageView`在动画过程中可能会闪。

参见`android.app.EnterTransitionCoordinator#startSharedElementTransition`。我们也可以通过[这个链接](https://source.android.com/devices/graphics/architecture.html)读到更多相关知识。

上面的代码有一行代码是Otto event bus的调用：

``` java
mBus.post(new ChangeImageThumbnailVisibility(false));
```

它发送了一条消息给`ImageListActivity`，让点击的图片从列表中隐藏。它是这样实现的：

``` java
// ImagesListActivity
@Subscribe
public void hideImageThumbnail(ChangeImageThumbnailVisibility message){
    Log.v(TAG, ">> hideImageThumbnail");

    mImageDetailsImageModel.setVisibility(message.isVisible());

    updateModel(mImageDetailsImageModel);

    Log.v(TAG, "<< hideImageThumbnail");
}
/**
 * This method basically changes visibility of concrete item
 */
private void updateModel(Image imageToUpdate) {
    Log.v(TAG, "updateModel, imageToUpdate " + imageToUpdate);
    for (Image image : mImagesList) {

        if(image.equals(imageToUpdate)){
            image.setVisibility(imageToUpdate.isVisible());
            break;
        }
    }
    int index = mImagesList.indexOf(imageToUpdate);
    Log.v(TAG, "updateModel, index " + index);

    mAdapter.notifyItemChanged(index);

    /**
     * For some reason recycler view is not always redrawn when 
     * adapter is updated.
     * onBindViewHolder is called but image doesn't disappear from 
     * screen. That's why we have to do this invalidation
     */
    Rect dirty = new Rect();
    View viewAtPosition = mLayoutManager.findViewByPosition(index);
    viewAtPosition.getDrawingRect(dirty);
    mRecyclerView.invalidate(dirty);
}
```

我们可以看到，这里我们用到了我们在启动`ImageDetailActivity`时保存的`mImageDetailsImageModel`。

### 真正创建动画

让我们仔细分析一下`playEnteringAnimation`方法：

``` java
public void playEnteringAnimation(
    int left, 
    int top, 
    int width,  
    int height) {
    Log.v(TAG, ">> playEnteringAnimation");

    mToLeft = left;
    mToTop = top;
    mToWidth = width;
    mToHeight =  height;

    AnimatorSet imageAnimatorSet = createEnteringImageAnimation();

    Animator mainContainerFadeAnimator = 
        createEnteringFadeAnimator();

    mEnteringAnimation = new AnimatorSet();
    mEnteringAnimation.setDuration(IMAGE_TRANSLATION_DURATION);
    mEnteringAnimation.setInterpolator(
        new AccelerateInterpolator());
    mEnteringAnimation.addListener(new SimpleAnimationListener() {

        @Override
        public void onAnimationCancel(Animator animation) {
            mEnteringAnimation = null;
        }

        @Override
        public void onAnimationEnd(Animator animation) {

            if (mEnteringAnimation != null) {
                mEnteringAnimation = null;

                mImageTo.setVisibility(View.VISIBLE);
                mAnimatedImage.setVisibility(View.INVISIBLE);
            } else {
                // Animation was cancelled. Do nothing
            }
        }
    });

    mEnteringAnimation.playTogether(
            imageAnimatorSet,
            mainContainerFadeAnimator
    );

    mEnteringAnimation.start();
    Log.v(TAG, "<< playEnteringAnimation");
}
```

这个方法做了两件事：
1. 它结合了两个动画：图片进入动画和根节点淡入动画
2. 如果动画没有被取消，当动画结束时，设置“动画的图片”为不可见，“放大的图片”为可见

`createEnteringFadeAnimator()`方法非常简单，它就是一个普通的`ObjectAnimator`，用于背景动画：

``` java
private ObjectAnimator createEnteringFadeAnimator() {
    return ObjectAnimator.ofFloat(mMainContainer, "alpha", 0.0f, 1.0f);
}
```

### 图片位置和ImageView变换

有趣的地方在我们前面提到的`createEnteringImageAnimation()`。

``` java
private AnimatorSet createEnteringImageAnimation() {
    Log.v(TAG, ">> createEnteringImageAnimation");
    
    ObjectAnimator positionAnimator = 
        createEnteringImagePositionAnimator();
    
    ObjectAnimator matrixAnimator = 
        createEnteringImageMatrixAnimator();

    AnimatorSet enteringImageAnimation = new AnimatorSet();
    
    enteringImageAnimation
        .playTogether(positionAnimator, matrixAnimator);

    Log.v(TAG, "<< createEnteringImageAnimation");
    return enteringImageAnimation;
}
```

这个方法结合了两个动画并返回了一个`AnimatorSet`。

`createEnteringImagePositionAnimator()`方法创建了两个animator，将图片从前一个界面中的位置移动到它在`ImageDetailsActivity`中应该在的位置。

``` java
private ObjectAnimator createEnteringImagePositionAnimator() {

    Log.v(TAG, "createEnteringImagePositionAnimator");

    PropertyValuesHolder propertyLeft = PropertyValuesHolder.ofInt(
            "left", 
            mAnimatedImage.getLeft(), 
            mToLeft);
    PropertyValuesHolder propertyTop = PropertyValuesHolder.ofInt(
            "top", 
            mAnimatedImage.getTop(),
            mToTop - getStatusBarHeight());

    PropertyValuesHolder propertyRight = PropertyValuesHolder.ofInt(
            "right", 
            mAnimatedImage.getRight(), 
            mToLeft + mToWidth);
   PropertyValuesHolder propertyBottom = PropertyValuesHolder.ofInt(
            "bottom", 
            mAnimatedImage.getBottom(), 
            mToTop + mToHeight - getStatusBarHeight());

   ObjectAnimator animator = ObjectAnimator.ofPropertyValuesHolder(
            mAnimatedImage, 
            propertyLeft, 
            propertyTop, 
            propertyRight, 
            propertyBottom);
   animator.addListener(new SimpleAnimationListener() {
        @Override
        public void onAnimationEnd(Animator animation) {
            // set new parameters of animated ImageView. This will 
            // prevent blinking of view when set visibility to 
            // visible in Exit animation
            
            FrameLayout.LayoutParams layoutParams = 
                  (FrameLayout.LayoutParams) mAnimatedImage
                                         .getLayoutParams();
            layoutParams.height = mImageTo.getHeight();
            layoutParams.width = mImageTo.getWidth();
            
            layoutParams.setMargins(
                mToLeft, 
                mToTop - getStatusBarHeight(), 
                0, 
                0);
        }
    });
    return animator;
}
```

这个方法创建了一个`ObjectAnimator`，改变View的属性：
1. 在X轴方向改变`ImageView`的left属性
2. 在Y轴方向改变`ImageView`的top属性。注意最终的top位置要减去状态栏的高度。这是因为`mToTop`变量的值没有把状态栏的高度考虑进去，它是其在屏幕里的top位置。
3. bottom和top一起，可以改变`ImageView`的高度。同样的，状态栏的高度也要从`mToHeight`中减去。
4. right和left一起改变了`ImageView`的宽度.

当动画结束，图片的位置通过设置`ImageView`的layoutParams的margin固定下来。

### ImageView拉伸的动画

为了设置`ImageView`拉伸的动画，我们需要调用`ImageView.animateTransform()`方法，但是这个方法对开发者不可见。所以我们要创建一个自定义属性：

``` java
/**
 * This property is passed to ObjectAnimator when we are animating
 * image matrix of ImageView
 */
Property<ImageView, Matrix> ANIMATED_TRANSFORM_PROPERTY 
= new Property<ImageView, Matrix>(Matrix.class, "animatedTransform"){
    /**
     * This is copy-paste form ImageView#animateTransform - method
     * is invisible in sdk
     */
    @Override
    public void set(ImageView imageView, Matrix matrix) {
        Drawable drawable = imageView.getDrawable();
        if (drawable == null) {
            return;
        }
        if (matrix == null) {
            drawable.setBounds(
            0, 
            0, 
            imageView.getWidth(), 
            imageView.getHeight());
        } else {
            drawable.setBounds(
            0, 
            0, 
            drawable.getIntrinsicWidth(),
            drawable.getIntrinsicHeight());
           
            Matrix drawMatrix = imageView.getImageMatrix();
            
            if (drawMatrix == null) {
                drawMatrix = new Matrix();
                imageView.setImageMatrix(drawMatrix);
            }
            imageView.setImageMatrix(matrix);
        }
        imageView.invalidate();
    }
    @Override
    public Matrix get(ImageView object) {
        return null;
    }
};
```

这个方法基本上是从`ImageView#animateTransform`复制出来的。当`Animator`尝试改变`animatedTransform`属性时，`set(ImageView, Matrix)`方法就会被调用。这个方法给`ImageView`设置了一个新的`Matrix`，然后调用`invalidate()`来让`ImageView`重绘。

另外，我们需要调用`MatrixEvaluator`来确定在动画过程中`ImageView`的拉伸变化。

``` java
/**
 * This class is passed to ObjectAnimator in order to animate
 * changes in ImageView image matrix
 */
public class MatrixEvaluator implements TypeEvaluator<Matrix> {
    float[] mTempStartValues = new float[9];
    float[] mTempEndValues = new float[9];
    Matrix mTempMatrix = new Matrix();
    @Override
    public Matrix evaluate(float fraction, 
                           Matrix startValue, 
                           Matrix endValue) {
        startValue.getValues(mTempStartValues);
        endValue.getValues(mTempEndValues);
        
        for (int i = 0; i < 9; i++) {
            float diff = mTempEndValues[i] - mTempStartValues[i];
            mTempEndValues[i] = 
                   mTempStartValues[i] 
                   + (fraction * diff);
        }
        mTempMatrix.setValues(mTempEndValues);
        return mTempMatrix;
    }
}
```

这个类（`MatrixEvaluator`）和 `ANIMATED_TRANSFORM_PROPERTY`一起传递给ObjectAnimator。

然后调用方法`createEnteringImageMatrixAnimator()`，它的职责
是创建改变图片拉伸的动画。

``` java
private ObjectAnimator createEnteringImageMatrixAnimator() {

    Matrix initMatrix = MatrixUtils.getImageMatrix(mAnimatedImage);
    // store the data about original matrix into array.
    // this array will be used later for exit animation
    initMatrix.getValues(mInitThumbnailMatrixValues);

    final Matrix endMatrix = MatrixUtils.getImageMatrix(mImageTo);

    mAnimatedImage.setScaleType(ImageView.ScaleType.MATRIX);

    return ObjectAnimator.ofObject(
        mAnimatedImage, 
        MatrixEvaluator.ANIMATED_TRANSFORM_PROPERTY,
        new MatrixEvaluator(), initMatrix, endMatrix);
}
```

这个方法还保存了初始状态的`Matrix`，给创建退出动画时使用。另外，一件很重要的事情是，动画中的`ImageView`的`scaleType`必须设置为`ScaleType.Matrix`。

### ImageDetailsActivity的退出动画

退出动画没有什么特别的，基本上就是把同样的动画反着运行。我们真正需要处理的情况是，进入动画进行到中间，我们就要退出的情况。我们必须停止进入动画，然后把`ImageView`移动回原来的位置。

我们还需要处理返回按钮点击事件：

``` java
@Override
public void onBackPressed() {
    //We don't call super to keep this activity on the screen when 
    //back is pressed
    Log.v(TAG, "onBackPressed");

    mEnterScreenAnimations.cancelRunningAnimations();

    Bundle initialBundle = getIntent().getExtras();
    int toTop = 
        initialBundle.getInt(KEY_THUMBNAIL_INIT_TOP_POSITION);
    int toLeft = 
        initialBundle.getInt(KEY_THUMBNAIL_INIT_LEFT_POSITION);
    int toWidth = 
        initialBundle.getInt(KEY_THUMBNAIL_INIT_WIDTH);
    int toHeight = 
        initialBundle.getInt(KEY_THUMBNAIL_INIT_HEIGHT);

    mExitScreenAnimations.playExitAnimations(
        toTop,
        toLeft,
        toWidth,
        toHeight,
        mEnterScreenAnimations.getInitialThumbnailMatrixValues());
    }
```

流程是这样的：
1. 如果进入动画还在运行，我们把它取消了。进入动画被设计成了动画取消的时候，`ImageView`会停在取消时的当前位置。
2. 我们利用从`ImageListActivity`传递过来的bundle来取出`ImageView`的初始属性。这些值被传到退出动画`ExitScreenAnimations`里，作为图片的最终位置。
3. `ImageView Matrix`的初始值在进入动画创建的时候保存下来了。


进入动画和退出动画之间几乎没有区别，请查看[Github](https://github.com/danylovolokh/ImageTransition) 上的源码获取更多细节。



