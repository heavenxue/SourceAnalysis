### canvas源码解析
##简介
        Android中使用图形处理引擎，2D部分是android SDK内部自己提供，3D部分是用Open GL ES 1.0。今天我们主要要了解的是2D相关的，
        如果你想看3D的话那么可以跳过这篇文章。    大部分2D使用的api都在android.graphics和android.graphics.drawable包中。
        他们提供了图形处理相关的： Canvas、ColorFilter、Point(点)和RetcF(矩形)等，还有一些动画相关的：AnimationDrawable、
        BitmapDrawable和TransitionDrawable等。以图形处理来说，我们最常用到的就是在一个View上画一些图片、形状或者自定义的
        文本内容，这里我们都是使用Canvas来实现的。你可以获取View中的Canvas对象，绘制一些自定义形状，然后调用View.invalidate
        方法让View重新刷新，然后绘制一个新的形状，这样达到2D动画效果。
        下面我们就主要来了解下Canvas的使用方法。Canvas对象的获取方式有两种：
        一种我们通过重写View.onDraw方法，View中的Canvas对象会被当做参数传递过来，我们操作这个Canvas，效果会直接反应在View中。
        另一种就是当你想创建一个Canvas对象时使用的方法： 
        Canvas c =new Canvas(Bitmap.createBitmap(100,100,Bitmap.Config.ARGB_88880));

### 探索

#### 思路
    我们知道一个View的绘制过程都必须经历三个重要的过程，也就是measure,layout和draw
    既然一个view的绘制主要是这三步，那一定有一个开始的地方啊，就像一个类是从main函数执行一样，对于view的绘制开始，这里先给出结论，后面会分析原因，具体结论如下：
    整个view的绘制流程是在ViewRootImpl类的performTraversals()方法（这个方法巨长）开始的，该函数做的执行过程主要是根据之前设置的状态，判断是否重新计算视图大小（measure）、
    是否重新放置视图的位置（layout）、是否重绘(draw)、其核心就是通过判断来选择顺序执行这三个方法中的哪个，如下：
    
    private void performTraversals() {
         //最外层的根视图的withMeasureSpec和heightMeasureSpec由来
         //lp.width和lp.height在创建ViewGroup实例时等于MATCH_PARENT
         ...
         int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
         int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
         ...
         mView.measure(childWidthMeasureSpec,childHeightMeasureSpec);
         ...
         mView.layout(0,0,mView.getMeasuredWidth(),mView.getMeasuredHeight());
         ...
         mView.draw(canvas);
         ...
    }
    
    /**
     * Figures out the measure spec for the root view in a window based on it's
     * layout params.
     *
     * @param windowSize
     *            The available width or height of the window
     *
     * @param rootDimension
     *            The layout params for one dimension (width or height) of the
     *            window.
     *
     * @return The measure spec to use to measure the root view.
     */
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
    好，既然这样，我们就从这里入手。

#### draw源码解析
    刚才上面我们已经提到了入口，所以，上面的注释说是用来测RootView的，上面传入参数后这个函数走的是MATCH_PARENT,使用MeasureSpec.makeMeasureSpec方法组装一个MeasureSpec,MeasureSpec的SpecMode等于EXACTLY,specSize等于WindowSize,也就是为何根视图总是全屏的原因。
整个流程如下：
![github](http://github.com/unicorn.png "github")
