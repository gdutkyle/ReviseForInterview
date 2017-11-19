# Android 布局优化 #
## 一 为什么要进行布局优化 
布局优化，我觉得总结起来就是：减少嵌套，避免过度加载。
  
##二 如果能使用linearlayout的，尽量不用RelativeLayout  
 
很多书上介绍这句话的时候，都是简单的一句话带过：因为Relativelayout的功能比较复杂，他的布局过程会花费更多的cpu时间。那么，RelativeLayout为什么会比LinearLayout多耗费cpu时间呢？我们来看Linearlayout的onMeasure（）源码 

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        if (mOrientation == VERTICAL) {
            measureVertical(widthMeasureSpec, heightMeasureSpec);
        } else {
            measureHorizontal(widthMeasureSpec, heightMeasureSpec);
        }
    }
好，就一句简单的，判断方向，执行对应的绘制方向，我们挑VERTICAL方向的代码进行分析  

    void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
        for (int i = 0; i < count; ++i) {
				...
                measureChildBeforeLayout(child, i, widthMeasureSpec, 0,
                        heightMeasureSpec, usedHeight);
				.....
            }
        if (skippedMeasure || remainingExcess != 0 && totalWeight > 0.0f) {

            float remainingWeightSum = mWeightSum > 0.0f ? mWeightSum : totalWeight;

            mTotalLength = 0;

            for (int i = 0; i < count; ++i) {
                final View child = getVirtualChildAt(i);
                if (child == null || child.getVisibility() == View.GONE) {
                    continue;
                }

                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                final float childWeight = lp.weight;
                if (childWeight > 0) {
                    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);

                }

              
            }

         
        } else {
            alternativeMaxWidth = Math.max(alternativeMaxWidth,                                           weightedMaxWidth);
            if (useLargestChild && heightMode != MeasureSpec.EXACTLY) {
                for (int i = 0; i < count; i++) {

                    float childExtra = lp.weight;
                    if (childExtra > 0) {
                        child.measure(
                                MeasureSpec.makeMeasureSpec(child.getMeasuredWidth(),
                                        MeasureSpec.EXACTLY),
                                MeasureSpec.makeMeasureSpec(largestChildHeight,
                                        MeasureSpec.EXACTLY));
                    }
                }
            }
        }

好了，我已经把对我们这次分析的主题没有关联的代码删除了。从这段代码我们可以看出，如果我们没有设置LinearLayout的weight属性的话，那么我们的Linearlayout代码只会执行onMeasure一次。  
那么，Relativelayout呢？我们接下来来看RelativeLayout的OnMeasure（）方法：

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {

        for (int i = 0; i < count; i++) {
            View child = views[i];
            if (child.getVisibility() != GONE) {
				...
                measureChildHorizontal(child, params, myWidth, myHeight);
				...
            }
        }

        for (int i = 0; i < count; i++) {
            final View child = views[i];
            if (child.getVisibility() != GONE) {
                
                measureChild(child, params, myWidth, 
            }
        }
    }

由此我们可以看出，RelativeLayout的onMeasure()会执行两次child的measure()方法。

所以，如果对于LinearLayout和ReleativeLayout都能够实现的布局，我们还是推荐使用Linearlayout,但是，如果需要LinearLayout结合其他的layout才能实现的布局，我们就建议还是使用RelativeLayout，因为viewGroup的嵌套使用，也会使减慢布局的绘制。  
## 三 使用标签
**1 include标签使用** 

include标签可以使一个我们已经写好的布局加载到当前的布局中，通过include标签，可以使我们的代码变得简洁，不用再多次重写相同的页面，include标签只支持android：layout_XX开头的属性，其他的属性不支持。 
 
    <include layout="@layout/mergelayout"
        android:id="@+id/merge_latyou_root"
        android:layout_height="match_parent"
        android:layout_width="match_parent">
        
    </include>

**2 merge标签使用**
merge标签和include一般一起使用，用来减少布局的嵌套。一般来说，如果外面的布局是一个linearlayout,而被包含的布局也是一个linearlayou,那么我们就可以使用merge标签，去减少多余的那一层LinearLayout嵌套  

    <?xml version="1.0" encoding="utf-8"?>
    <merge
    xmlns:android="http://schemas.android.com/apk/res/android" android:layout_width="match_parent"
    android:layout_height="match_parent">
    <TextView
        android:layout_height="100dp"
        android:layout_width="match_parent"
        android:id="@+id/tv1"
        android:text="text1"/>
    <TextView
        android:layout_height="100dp"
        android:layout_width="match_parent"
        android:id="@+id/tv2"
        android:text="text2"/>
    </merge>
**3 ViewStub标签使用**  
我们来看看ViewStub的注释  

    /**
     * A ViewStub is an invisible, zero-sized View that can be used to lazily inflate
	 * layout resources at runtime.
	 *
 	 * When a ViewStub is made visible, or when {@link #inflate()}  is invoked, the layout resource 
 	 * is inflated. The ViewStub then replaces itself in its parent with the inflated View or Views.
	 * Therefore, the ViewStub exists in the view hierarchy until {@link #setVisibility(int)} or
 	 * {@link #inflate()} is invoked.
 	**/ 
这段话的意思是，ViewStub是一个不可见的，宽度和高度为0的View，能够通过layout的inflate方法被加载当ViewStub被设置成Visible或者inflate方法被调用，那么，被ViewStub所引用的View资源将会被加载，ViewStub在父布局中的位置将会被所加载的资源所替代。因此，这个时候，viewStub就已经不在了，我们来看看ViewStub的使用：  

     <ViewStub
        android:id="@+id/panel_import"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:inflatedId="@+id/viewstub_import"
        android:layout="@layout/mergelayout" />
ViewStub标签暂时不支持merge标签。  
接下来我们来分析ViewStub的源码，首先我们来看ViewStub的构造方法  

    public ViewStub(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context);

        final TypedArray a = context.obtainStyledAttributes(attrs,
                R.styleable.ViewStub, defStyleAttr, defStyleRes);
        mInflatedId = a.getResourceId(R.styleable.ViewStub_inflatedId, NO_ID);
        mLayoutResource = a.getResourceId(R.styleable.ViewStub_layout, 0);
        mID = a.getResourceId(R.styleable.ViewStub_id, NO_ID);
        a.recycle();

        setVisibility(GONE);
        setWillNotDraw(true);
    }
我们可以看到，我们在构造方法中，引用了我们在xml中制定的inflatedId,mLayoutResource,然后我们可以看到，viewStub调用了  

    setVisibility(GONE);
    setWillNotDraw(true);
也就是我们上文所说的，ViewStub默认是不可见的，他的宽高是0dp。
接下来，我们来看使ViewStub加载对应资源的两个方法  

    public View inflate() {
        final ViewParent viewParent = getParent();

        if (viewParent != null && viewParent instanceof ViewGroup) {
            if (mLayoutResource != 0) {
                final ViewGroup parent = (ViewGroup) viewParent;
                final View view = inflateViewNoAdd(parent);
                replaceSelfWithView(view, parent);

                mInflatedViewRef = new WeakReference<>(view);
                if (mInflateListener != null) {
                    mInflateListener.onInflate(this, view);
                }

                return view;
            } else {
                throw new IllegalArgumentException("ViewStub must have a valid layoutResource");
            }
        } else {
            throw new IllegalStateException("ViewStub must have a non-null ViewGroup viewParent");
        }
    }

从这段代码中，我们大致分析流程，ViewStub先去获取父布局，如果父布局是一个ViewGroup,并且引用的mLayoutResource不为0的话，那么就在父布局中，把自己移除，并把当前加载的布局放到父布局对应的位置中  

     private void replaceSelfWithView(View view, ViewGroup parent) {
        final int index = parent.indexOfChild(this);
        parent.removeViewInLayout(this);

        final ViewGroup.LayoutParams layoutParams = getLayoutParams();
        if (layoutParams != null) {
            parent.addView(view, index, layoutParams);
        } else {
            parent.addView(view, index);
        }
    }
接下里，我们来看另一个方法，也即是setVisible（int）方法  
    public void setVisibility(int visibility) {
        if (mInflatedViewRef != null) {
            View view = mInflatedViewRef.get();
            if (view != null) {
                view.setVisibility(visibility);
            } else {
                throw new IllegalStateException("setVisibility called on un-referenced view");
            }
        } else {
            super.setVisibility(visibility);
            if (visibility == VISIBLE || visibility == INVISIBLE) {
                inflate();
            }
        }
    }

所以，当我们第一次设置setVisible（）方法的时候，mInflatedViewRef是空的，所以调用的还是inflate方法。  
所以我们可以看到，调用setViisible方法，和inflate（）方法，都是可以起到一样的效果。。。




      