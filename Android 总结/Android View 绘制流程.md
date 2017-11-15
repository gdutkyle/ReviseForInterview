#Android View 绘制流程#
----
# 一 开始的开始 #
相信很多小伙伴，在去面试的时候，都会遇到面试官一脸正经的问你，简单的说下View的绘制流程。其实一遇到这个问题，我是奔溃的，因为完全无法讲出第一句话，很多人都是一句，哦，view的绘制流程就是onMeasure()->onLayout()->onDraw()。onMeasure()方法就是计算View的大小，onLayout()就是计算view所在的位置，onDraw()就是开始绘制。balabala，这个时候，面试官就会面带微笑，说了句：哦，然后呢？  
# 二 View绘制的开始 #
view的绘制，是从viewRootImp的performTraversals()开始的，performTraversal()的源码很长，其实，我们只要知道以下的三个流程即可：    

    private void performTraversals() {
    
      performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
      performLayout(lp, mWidth, mHeight);
      performDraw();
    }
好了，我们接下来来看看performMeasur(childWidthMeasureSpec,childHeightMeasureSpec)的代码  

    private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        if (mView == null) {
            return;
        }
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
        try {
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }

从这里我们可以看出，ViewRootImp中的performMeasure方法，最终是回调到View中的measure（）方法中。  
我们接下来看performLayout方法  

    private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
            int desiredWindowHeight) {
        mLayoutRequested = false;
        mScrollMayChange = true;
        mInLayout = true;

        final View host = mView;
        if (host == null) {
            return;
        }
        ....
        host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
        ....
        mInLayout = false;
    }
在这段代码中，我们省去了绝大部分顶层view的layout操作，只保留了绘制流程相关的代码。从这段代码我们可以看出performLayout最终会调用到view的layout方法，并把父view的left top right bottom传给view中。至于view的layout方法是怎样的，我们后面再讲。  
接下来，我们来看performDraw方法  

    private void performDraw() {
        if (mAttachInfo.mDisplayState == Display.STATE_OFF && !mReportNextDraw) {
            return;
        } else if (mView == null) {
            return;
        }

        final boolean fullRedrawNeeded = mFullRedrawNeeded;
        mFullRedrawNeeded = false;

        mIsDrawing = true;
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "draw");
        try {
            draw(fullRedrawNeeded);
        } finally {
            mIsDrawing = false;
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
我们主要看draw（）方法，这个方法最后其实调用的就是  

    public final void dispatchOnDraw() {
        if (mOnDrawListeners != null) {
            mInDispatchOnDraw = true;
            final ArrayList<OnDrawListener> listeners = mOnDrawListeners;
            int numListeners = listeners.size();
            for (int i = 0; i < numListeners; ++i) {
                listeners.get(i).onDraw();
            }
            mInDispatchOnDraw = false;
        }
    }
这个方法是用来干嘛的？其实就是通知已经注册了OnDrawListener的监听者，view已经要开始绘制了。其实，也就是到了onDraw（）方法中了。  
通过源码，一步一步追踪，我们可以发现，view的这个绘制流程其实是这样的  
ViewRootImp->PerformTrasvel()->PerformMeasure()->measure()->onMeasure()  
->performLayout()->onLayout()  
->performDraw()->draw()->dispatchDraw()->onDraw()  
#三 开始发大招 #
###1 View的measure(int widthMeasureSpec, int heightMeasureSpec)  
我们通过看注释，知道了measure的作用是，计算这个view应该是多大，因为父布局提供了传入制定的宽高参数。具体的测量工作其实是放在onMeasure（int，int）方法中去执行的，onMeasure（int，int）方法能够也必须被继承（当我们重写view）的时候。  
好了，那么问题来了，什么是widthMeasureSpec 和heighMeasureSpec？
我们回到ViewRootImp的performTrasvel（）方法中  

    childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);
                childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
也即是，width和heigh的MeasureSpec是通过getRootMeasureSpec(int,int)方法中获取的，
  
    private static int getRootMeasureSpec(int windowSize, int rootDimension) {
        int measureSpec;
        switch (rootDimension) {

        case ViewGroup.LayoutParams.MATCH_PARENT:
            // Window can't resize. Force root view to be windowSize.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT:
            // Window can resize. Set max size for root view.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
            break;
        default:
            // Window wants to be an exact size. Force root view to be that size.
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
            break;
        }
        return measureSpec;
    }
核心的计算方法就是MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);这个方法。
其实这个方法的作用就是，MeasureSpec通过SpecMode和SpecSize来判断一个view应该是多大。那么，什么是specMode？
SepcMode一共有三类，除了我们上面看到的MeasureSpec.EXACTLY外，还有MeasureSpec.UNSPECIFIED和MeasureSpec.AT_MOST  
**MeasureSpec.UNSPECIFIED：** 父布局没有指定任何的约束大小，子布局想多大就多大；  
**MeasureSpec.EXACTLY：** 父布局已经指定了一个确切的大小，子布局的大小就是SpecMode的大小；  
**MeasureSpec.AT_MOST：** 子布局可以想多大就多大，但是不能超过SpecSize的值
