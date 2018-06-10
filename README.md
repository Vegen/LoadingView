### **前言**
> 很多时候，我们的页面需要网络请求完毕再根据数据加载 UI，而在此过程中，用户等待时，一般会有一个加载中的提示。本文结合自定义 View/ViewGroup 以及属性动画相关知识，讲述较为炫酷的动画效果的 LoadingView，并在细节上优化了代码。

---
### **1.LoadingView 效果分析**
#### 1.1 最终实现的效果图如下
![LoadingView.gif](https://upload-images.jianshu.io/upload_images/4193211-858e661baaad5dc6.gif?imageMogr2/auto-orient/strip)

#### 1.2 效果分析
LoadingView 有三个不同形状的 View 进行下落回弹并且不断切换，与此同时，底部有阴影在不断的切换大小，最底下是文字说明。

---
### **2.实现分析**
 - *先自定义 ShapeView 打造三个形状变换的 View。*
```
public class ShapeView extends View {
    private Shape mCurrentShape = Shape.Circle;
    Paint mPaint;
    private Path mPath;

    public ShapeView(Context context) {
        this(context, null);
    }

    public ShapeView(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public ShapeView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        mPaint = new Paint();
        mPaint.setAntiAlias(true);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        // 只保证是正方形
        int width = MeasureSpec.getSize(widthMeasureSpec);
        int height = MeasureSpec.getSize(heightMeasureSpec);
        setMeasuredDimension(Math.min(width, height), Math.min(width, height));
    }

    @Override
    protected void onDraw(Canvas canvas) {

        switch (mCurrentShape) {
            case Circle:
                // 画圆形
                int center = getWidth() / 2;
                mPaint.setColor(ContextCompat.getColor(getContext(), R.color.circle));
                canvas.drawCircle(center, center, center, mPaint);
                break;
            case Square:
                // 画正方形
                mPaint.setColor(ContextCompat.getColor(getContext(), R.color.rect));
                canvas.drawRect(0, 0, getWidth(), getHeight(), mPaint);
                break;
            case Triangle:
                // 画三角  Path 画路线
                mPaint.setColor(ContextCompat.getColor(getContext(), R.color.triangle));
                if (mPath == null) {
                    // 画路径
                    mPath = new Path();
                    mPath.moveTo(getWidth() / 2, 0);
                    mPath.lineTo(0, (float) ((getWidth() / 2) * Math.sqrt(3)));
                    mPath.lineTo(getWidth(), (float) ((getWidth() / 2) * Math.sqrt(3)));
                    // path.lineTo(getWidth()/2,0);
                    mPath.close();// 把路径闭合
                }
                canvas.drawPath(mPath, mPaint);
                break;
        }
    }

    /**
     * 改变形状
     */
    public void exchange() {
        switch (mCurrentShape) {
            case Circle:
                mCurrentShape = Shape.Square;
                break;
            case Square:
                mCurrentShape = Shape.Triangle;
                break;
            case Triangle:
                // 画三角  Path 画路线
                mCurrentShape = Shape.Circle;
                break;
        }
        // 不断重新绘制形状
        invalidate();
    }

    public enum Shape {
        Circle, Square, Triangle
    }

    public Shape getCurrentShape() {
        return mCurrentShape;
    }
```

 - 自定义 ViewGroup 打造整体 View。
```
public class LoadingView extends LinearLayout {
    private ShapeView mShapeView;// 上面的形状
    private View mShadowView;// 中间的阴影
    private int mTranslationDistance = 0;
    // 动画执行的时间
    private final long ANIMATOR_DURATION = 500;
    // 是否停止动画
    private boolean mIsStopAnimator = false;

    public LoadingView(Context context) {
        this(context, null);
    }

    public LoadingView(Context context, @Nullable AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public LoadingView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        mTranslationDistance = dip2px(80);
        initLayout();
    }

    private int dip2px(int dip) {
        return (int) TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, dip, getResources().getDisplayMetrics());
    }

    /**
     * 初始化加载布局
     */
    private void initLayout() {
        // 1. 加载写好的 ui_loading_view
        // 1.1 实例化View
        // this 代表把 ui_loading_view 加载到 LoadingView 中
        inflate(getContext(), R.layout.layout_loading, this);

        mShapeView = (ShapeView) findViewById(R.id.shape_view);
        mShadowView = findViewById(R.id.shadow_view);

        post(new Runnable() {
            @Override
            public void run() {
                // onResume 之后View绘制流程执行完毕之后（View的绘制流程源码分析那一章）
                startFallAnimator();
            }
        });
        // onCreate() 方法中执行 ，布局文件解析 反射创建实例
    }

    // 开始下落动画
    private void startFallAnimator() {
        if (mIsStopAnimator) {
            return;
        }
        // 动画作用在谁的身上
        // 下落位移动画
        ObjectAnimator translationAnimator = ObjectAnimator.ofFloat(mShapeView, "translationY", 0, mTranslationDistance);
        // 配合中间阴影缩小
        ObjectAnimator scaleAnimator = ObjectAnimator.ofFloat(mShadowView, "scaleX", 1f, 0.3f);
        AnimatorSet animatorSet = new AnimatorSet();
        animatorSet.setDuration(ANIMATOR_DURATION);
        // 下落的速度应该是越来越快
        animatorSet.setInterpolator(new AccelerateInterpolator());
        animatorSet.playTogether(translationAnimator, scaleAnimator);
        animatorSet.start();
        // 下落完之后就上抛了，监听动画执行完毕
        // 是一种思想，在 Adapter 中的 BannerView 写过
        animatorSet.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationEnd(Animator animation) {
                // 改变形状
                mShapeView.exchange();
                // 下落完之后就上抛了
                startUpAnimator();
                // 开始旋转
            }
        });
    }

    /**
     * 开始执行上抛动画
     */
    private void startUpAnimator() {
        if (mIsStopAnimator) {
            return;
        }
        Log.e("TAG", "startUpAnimator" + this);
        // 动画作用在谁的身上
        // 下落位移动画
        ObjectAnimator translationAnimator = ObjectAnimator.ofFloat(mShapeView, "translationY", mTranslationDistance, 0);
        // 配合中间阴影缩小
        ObjectAnimator scaleAnimator = ObjectAnimator.ofFloat(mShadowView, "scaleX", 0.3f, 1f);

        AnimatorSet animatorSet = new AnimatorSet();
        animatorSet.setDuration(ANIMATOR_DURATION);
        animatorSet.setInterpolator(new DecelerateInterpolator());
        animatorSet.playTogether(translationAnimator, scaleAnimator);
        // 先执行 translationAnimator 接着执行 scaleAnimator
        // 上抛完之后就下落了，监听动画执行完毕
        animatorSet.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationEnd(Animator animation) {
                // 上抛完之后就下落了
                startFallAnimator();
            }

            @Override
            public void onAnimationStart(Animator animation) {
                // 开始旋转
                startRotationAnimator();
            }
        });
        // 执行 -> 监听的 onAnimationStart 方法
        animatorSet.start();
    }

    /**
     * 上抛的时候需要旋转
     */
    private void startRotationAnimator() {
        ObjectAnimator rotationAnimator = null;
        switch (mShapeView.getCurrentShape()) {
            case Circle:
            case Square:
                // 180
                rotationAnimator = ObjectAnimator.ofFloat(mShapeView, "rotation", 0, 180);
                break;
            case Triangle:
                // 120
                rotationAnimator = ObjectAnimator.ofFloat(mShapeView, "rotation", 0, -120);
                break;
        }
        rotationAnimator.setDuration(ANIMATOR_DURATION);
        rotationAnimator.setInterpolator(new DecelerateInterpolator());
        rotationAnimator.start();
    }

    @Override
    public void setVisibility(int visibility) {
        super.setVisibility(View.INVISIBLE);// 不要再去排放和计算，少走一些系统的源码（View的绘制流程）
        // 清理动画
        mShapeView.clearAnimation();
        mShadowView.clearAnimation();
        // 把LoadingView从父布局移除
        ViewGroup parent = (ViewGroup) getParent();
        if (parent != null) {
            parent.removeView(this);// 从父布局移除
            removeAllViews();// 移除自己所有的View
        }
        mIsStopAnimator = true;
    }
}
```

 - *layout_loading.xml*
```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:orientation="vertical"
    android:background="#FFFFFF"
    android:gravity="center"
    android:layout_height="match_parent">

    <com.study.vegen.loadingview.ShapeView
        android:id="@+id/shape_view"
        android:layout_width="25dp"
        android:layout_marginBottom="82dp"
        android:layout_height="25dp" />

    <View
        android:id="@+id/shadow_view"
        android:background="@drawable/bg_loading_shadow"
        android:layout_width="25dp"
        android:layout_height="3dp"/>

    <TextView
        android:layout_marginTop="5dp"
        android:layout_width="wrap_content"
        android:text="玩命加载中..."
        android:layout_height="wrap_content" />
</LinearLayout>
```

 - *在 Activity 中使用，添加到布局文件，此处不贴代码*
 
[**博客地址**][1]


  [1]: https://blog.csdn.net/androidtalent/
  
---
### **3.总结**  
> 在 LoadingView.java 中，我们重写了 setVisibility(int visibility)，就是当加载完毕需要 GONE/INVISIBLE 掉 LoadingView 时，统一调 setVisibility(View.INVISIBLE)，目的是不要再去摆放布局和计算，少走一些系统的源码，并做一些清理工作。
```
@Override
    public void setVisibility(int visibility) {
        super.setVisibility(View.INVISIBLE);// 不要再去摆放布局和计算，少走一些系统的源码（View的绘制流程）
        // 清理动画
        mShapeView.clearAnimation();
        mShadowView.clearAnimation();
        // 把LoadingView从父布局移除
        ViewGroup parent = (ViewGroup) getParent();
        if (parent != null) {
            parent.removeView(this);// 从父布局移除
            removeAllViews();// 移除自己所有的View
        }
        mIsStopAnimator = true;
    }
```
