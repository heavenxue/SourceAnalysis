### canvas源码解析
## 简介

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
![github](https://github.com/heavenxue/SourceAnalysis/raw/master/pic/1.png "github")

所以draw过程也是在ViewRootImpl的performTraversals()方法内部调用的，其调用顺序在measure()和layout()之后，这里的mView对于Activity来说就是PhoneWindow.DectorView,ViewRootImpl中的代码会创建一个Canvas对象，然后调用View.draw()来执行具体的绘制工作。
view递归draw流程图如下:
![github](https://github.com/heavenxue/SourceAnalysis/raw/master/pic/2.png "github")
由于ViewGroup没有重写View的draw方法，所以下面直接从View的draw方法开始分析

    public void draw(Canvas canvas) {
                ......
                /*
                 * Draw traversal performs several drawing steps which must be executed
                 * in the appropriate order:
                 *
                 *      1. Draw the background
                 *      2. If necessary, save the canvas' layers to prepare for fading
                 *      3. Draw view's content
                 *      4. Draw children
                 *      5. If necessary, draw the fading edges and restore layers
                 *      6. Draw decorations (scrollbars for instance)
                 */
                // Step 1, draw the background, if needed
                int saveCount;if (!dirtyOpaque) {
                  drawBackground(canvas);
                }
                // skip step 2 & 5 if possible (common case)
                ......
                // Step 2, save the canvas' layers
                if (drawTop) {
                   canvas.saveLayer(left, top, right, top + length, null, flags);
                }
                ......
                // Step 3, draw the content
                if (!dirtyOpaque) onDraw(canvas);
                // Step 4, draw the children
                dispatchDraw(canvas);
                ......
                if (drawTop) {
                    matrix.setScale(1, fadeHeight * topFadeStrength);
                    matrix.postTranslate(left, top);
                    fade.setLocalMatrix(matrix);
                    p.setShader(fade);
                    canvas.drawRect(left, top, right, top + length, p);
                }
                ......
                // Step 6, draw decorations (foreground, scrollbars)
                onDrawForeground(canvas);
            }

看整个view的draw方法很复杂，但是注释很详细，从注释可以看出整个draw过程分6步。源码注释说（skip step 2 & 5 if possible (common case) ）第2步和第5步可以跳过，所以我们重点来看剩余4步，如下：
##### 第一步，对view的背景进行绘制
可以看见，draw方法通过调用drawBackground(canvas)实现了背景绘制，看下源码：

    private void drawBackground(Canvas canvas) {
            //获取xml中通过android:background属性或代码中setBackgroundColor(),setBackgroundResources()等方法进行赋值的背景drawable
            final Drawable background = mBackground;
            ......
            //根据layout过程确定的View位置来设置背景的绘制区域
            if (mBackgroundSizeChanged && mBackground != null) {
               mBackground.setBounds(0, 0,  mRight - mLeft, mBottom - mTop);
               mBackgroundSizeChanged = false;
               rebuildOutline();
            }
            ......
            //调用Drawable的draw()方法来完成背景的绘制
            background.draw(canvas);
        }

##### 第三步，对view的内容绘制
可以看到，这里去调用了View的onDraw()方法，所以我们看下view的onDraw()方法（ViewGroup没有重写这个方法），如下：

    /**
     * Implement this to do your drawing.
     *
     * @param canvas the canvas on which the background will be drawn
     */
    protected void onDraw(Canvas canvas) {
    }

可以看到，是一个空方法，因为每个view的内容部分是各不相同的，所以要由子类去实现具体的逻辑
##### 第四步，对当前的view的所有子view进行绘制，如果当前view没有子view就不需要绘制
我们来看下view的draw方法中的dispatchDraw(canvas)方法源码，可以看到：

    /**
     * Called by draw to draw the child views. This may be overridden
     * by derived classes to gain control just before its children are drawn
     * (but after its own view has been drawn).
     * @param canvas the canvas on which to draw the view
     */
    protected void dispatchDraw(Canvas canvas) {
    
    }

view的dispatchDraw方法也是一个空方法，而且注释说明了如果view包含子类需要重写它，所以我们有必要看下ViewGroup的dispatchDraw()方法源码（这也就是说刚刚说的当前View的所有子view进行绘制，如果当前的View没有子view就不需要进行绘制的原因，因为如果是View调用该方法是空的，而viewGroup才实现），如下：

    @Override
    protected void dispatchDraw(Canvas canvas) {
        ......
        final int childrenCount = mChildrenCount;
        final View[] children = mChildren;
        ......
        for(int i = 0;i < childrenCount; i ++){
             ......
             if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                 more |= drawChild(canvas, child, drawingTime);
             }
            }
        ......
        //Draw any disappearing views that have animations
        if(DisappearingChildren != null){
        ......
        for (int i = disappearingCount; i >= 0; i--) {
           final View child = disappearingChildren.get(i);
           more |= drawChild(canvas, child, drawingTime);
        }
        ......
    }

可以看出，ViewGroup确实重写了View的dispatchDraw()方法，该方法内部会遍历每个子View,然后调用drawChild()方法，我们可以看下ViewGroup的drawChild方法，如下：

    protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        return child.draw(canvas, this, drawingTime);
    }

可以看出，drawChild方法调用了子view的draw方法，所以说viewGroup类已经为我们重写了dispatchDraw()功能实现，我们一般不需要重写这个方法，但可以重载父类函数实现具体功能。
##### 第六步，对view的滚动条进行绘制
 可以看到，这里去调用了一下view的onDrawScrollBars()方法，所以看下它的源码如下：
 
    /**
     * <p>Request the drawing of the horizontal and the vertical scrollbar. The
     * scrollbars are painted only if they have been awakened first.</p>
     *
     * @param canvas the canvas on which to draw the scrollbars
     *
     * @see #awakenScrollBars(int)
     */
    protected final void onDrawScrollBars(Canvas canvas) {
        //绘制ScrollBars分析不是我们这篇的重点，暂时不做分析
        ......
    }

可以看见其实任何一个View都是有（水平垂直）滚动条的，只是一般情况下都不显示而已，到此，View的draw绘制部分已经分析完毕。

### draw原理总结
可以看见，绘制过程就是把view对象绘制在屏幕上，整个draw()过程需要注意如下细节:

* 如果该view是一个ViewGroup，则需要递归绘制其包含的所有子view
* view默认不会绘制任何内容，真正的绘制都需要自己在子类中实现。
* view的绘制是借助onDraw()传入canvas类进行的
* 区分view动画和ViewGroup布局动画，前者指的是View自身的动画，可以通过setAnimation添加，后者专门针对ViewGroup显示内部子视图时设置动画，可以在xml布局文件中对ViewGroup设置layoutAnimation属性（譬如对LinearLayout设置子view在显示时出现逐行，随机等不同动画效果）
* 在获取画布剪切区（每个view的draw中传入的Canvas）时会自动处理掉padding,子view获取Canvas不用关注这些逻辑，只用关心如何绘制即可
* 默认情况下，子view的viewGroup.drawChild绘制顺序和子view被添加的顺序一致，但是你也可以重载ViewGroup.getChildDrawingOrder()方法提供不同的顺序

#### Canvas源码解析
### 从代码入手来看
好了，开始canvas之旅了，因此我们首先从ViewGroup的dispatchDraw开始入手，这里要传入一个Canvas，这个Canvas是由ViewRootImpl.java传入，此时的Canvas是一个画布
而dispatchDraw方法里面会调用了drawChild(canvas, transientChild, drawingTime);这个方法里可以找到child.draw(canvas, this, drawingTime);
继续看，指向了view的draw方法，这个函数不同于draw(Canvas canvas)函数，后者是view绘制开始的地方，下面是源码：

    boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {
        final boolean hardwareAcceleratedCanvas = canvas.isHardwareAccelerated();
        /* If an attached view draws to a HW canvas, it may use its RenderNode + DisplayList.
         *
         * If a view is dettached, its DisplayList shouldn't exist. If the canvas isn't
         * HW accelerated, it can't handle drawing RenderNodes.
         */
        boolean drawingWithRenderNode = mAttachInfo != null
                && mAttachInfo.mHardwareAccelerated
                && hardwareAcceleratedCanvas;
    
        boolean more = false;
        final boolean childHasIdentityMatrix = hasIdentityMatrix();
        final int parentFlags = parent.mGroupFlags;
    
        if ((parentFlags & ViewGroup.FLAG_CLEAR_TRANSFORMATION) != 0) {
            parent.getChildTransformation().clear();
            parent.mGroupFlags &= ~ViewGroup.FLAG_CLEAR_TRANSFORMATION;
        }
    
        Transformation transformToApply = null;
        boolean concatMatrix = false;
        final boolean scalingRequired = mAttachInfo != null && mAttachInfo.mScalingRequired;
        final Animation a = getAnimation();
        if (a != null) {
            more = applyLegacyAnimation(parent, drawingTime, a, scalingRequired);
            concatMatrix = a.willChangeTransformationMatrix();
            if (concatMatrix) {
                mPrivateFlags3 |= PFLAG3_VIEW_IS_ANIMATING_TRANSFORM;
            }
            transformToApply = parent.getChildTransformation();
        } else {
            if ((mPrivateFlags3 & PFLAG3_VIEW_IS_ANIMATING_TRANSFORM) != 0) {
                // No longer animating: clear out old animation matrix
                mRenderNode.setAnimationMatrix(null);
                mPrivateFlags3 &= ~PFLAG3_VIEW_IS_ANIMATING_TRANSFORM;
            }
            if (!drawingWithRenderNode
                    && (parentFlags & ViewGroup.FLAG_SUPPORT_STATIC_TRANSFORMATIONS) != 0) {
                final Transformation t = parent.getChildTransformation();
                final boolean hasTransform = parent.getChildStaticTransformation(this, t);
                if (hasTransform) {
                    final int transformType = t.getTransformationType();
                    transformToApply = transformType != Transformation.TYPE_IDENTITY ? t : null;
                    concatMatrix = (transformType & Transformation.TYPE_MATRIX) != 0;
                }
            }
        }
    
        concatMatrix |= !childHasIdentityMatrix;
    
        // Sets the flag as early as possible to allow draw() implementations
        // to call invalidate() successfully when doing animations
        mPrivateFlags |= PFLAG_DRAWN;
    
        if (!concatMatrix &&
                (parentFlags & (ViewGroup.FLAG_SUPPORT_STATIC_TRANSFORMATIONS |
                        ViewGroup.FLAG_CLIP_CHILDREN)) == ViewGroup.FLAG_CLIP_CHILDREN &&
                canvas.quickReject(mLeft, mTop, mRight, mBottom, Canvas.EdgeType.BW) &&
                (mPrivateFlags & PFLAG_DRAW_ANIMATION) == 0) {
            mPrivateFlags2 |= PFLAG2_VIEW_QUICK_REJECTED;
            return more;
        }
        mPrivateFlags2 &= ~PFLAG2_VIEW_QUICK_REJECTED;
    
        if (hardwareAcceleratedCanvas) {
            // Clear INVALIDATED flag to allow invalidation to occur during rendering, but
            // retain the flag's value temporarily in the mRecreateDisplayList flag
            mRecreateDisplayList = (mPrivateFlags & PFLAG_INVALIDATED) != 0;
            mPrivateFlags &= ~PFLAG_INVALIDATED;
        }
    
        RenderNode renderNode = null;
        Bitmap cache = null;
        int layerType = getLayerType(); // TODO: signify cache state with just 'cache' local
        if (layerType == LAYER_TYPE_SOFTWARE
                || (!drawingWithRenderNode && layerType != LAYER_TYPE_NONE)) {
            // If not drawing with RenderNode, treat HW layers as SW
            layerType = LAYER_TYPE_SOFTWARE;
            buildDrawingCache(true);
            cache = getDrawingCache(true);
        }
    
        if (drawingWithRenderNode) {
            // Delay getting the display list until animation-driven alpha values are
            // set up and possibly passed on to the view
            renderNode = updateDisplayListIfDirty();
            if (!renderNode.isValid()) {
                // Uncommon, but possible. If a view is removed from the hierarchy during the call
                // to getDisplayList(), the display list will be marked invalid and we should not
                // try to use it again.
                renderNode = null;
                drawingWithRenderNode = false;
            }
        }
    
        int sx = 0;
        int sy = 0;
        if (!drawingWithRenderNode) {
            computeScroll();
            sx = mScrollX;
            sy = mScrollY;
        }
    
        final boolean drawingWithDrawingCache = cache != null && !drawingWithRenderNode;
        final boolean offsetForScroll = cache == null && !drawingWithRenderNode;
    
        int restoreTo = -1;
        if (!drawingWithRenderNode || transformToApply != null) {
            restoreTo = canvas.save();
        }
        if (offsetForScroll) {
            canvas.translate(mLeft - sx, mTop - sy);
        } else {
            if (!drawingWithRenderNode) {
                canvas.translate(mLeft, mTop);
            }
            if (scalingRequired) {
                if (drawingWithRenderNode) {
                    // TODO: Might not need this if we put everything inside the DL
                    restoreTo = canvas.save();
                }
                // mAttachInfo cannot be null, otherwise scalingRequired == false
                final float scale = 1.0f / mAttachInfo.mApplicationScale;
                canvas.scale(scale, scale);
            }
        }
    
        float alpha = drawingWithRenderNode ? 1 : (getAlpha() * getTransitionAlpha());
        if (transformToApply != null
                || alpha < 1
                || !hasIdentityMatrix()
                || (mPrivateFlags3 & PFLAG3_VIEW_IS_ANIMATING_ALPHA) != 0) {
            if (transformToApply != null || !childHasIdentityMatrix) {
                int transX = 0;
                int transY = 0;
    
                if (offsetForScroll) {
                    transX = -sx;
                    transY = -sy;
                }
    
                if (transformToApply != null) {
                    if (concatMatrix) {
                        if (drawingWithRenderNode) {
                            renderNode.setAnimationMatrix(transformToApply.getMatrix());
                        } else {
                            // Undo the scroll translation, apply the transformation matrix,
                            // then redo the scroll translate to get the correct result.
                            canvas.translate(-transX, -transY);
                            canvas.concat(transformToApply.getMatrix());
                            canvas.translate(transX, transY);
                        }
                        parent.mGroupFlags |= ViewGroup.FLAG_CLEAR_TRANSFORMATION;
                    }
    
                    float transformAlpha = transformToApply.getAlpha();
                    if (transformAlpha < 1) {
                        alpha *= transformAlpha;
                        parent.mGroupFlags |= ViewGroup.FLAG_CLEAR_TRANSFORMATION;
                    }
                }
    
                if (!childHasIdentityMatrix && !drawingWithRenderNode) {
                    canvas.translate(-transX, -transY);
                    canvas.concat(getMatrix());
                    canvas.translate(transX, transY);
                }
            }
    
            // Deal with alpha if it is or used to be <1
            if (alpha < 1 || (mPrivateFlags3 & PFLAG3_VIEW_IS_ANIMATING_ALPHA) != 0) {
                if (alpha < 1) {
                    mPrivateFlags3 |= PFLAG3_VIEW_IS_ANIMATING_ALPHA;
                } else {
                    mPrivateFlags3 &= ~PFLAG3_VIEW_IS_ANIMATING_ALPHA;
                }
                parent.mGroupFlags |= ViewGroup.FLAG_CLEAR_TRANSFORMATION;
                if (!drawingWithDrawingCache) {
                    final int multipliedAlpha = (int) (255 * alpha);
                    if (!onSetAlpha(multipliedAlpha)) {
                        if (drawingWithRenderNode) {
                            renderNode.setAlpha(alpha * getAlpha() * getTransitionAlpha());
                        } else if (layerType == LAYER_TYPE_NONE) {
                            canvas.saveLayerAlpha(sx, sy, sx + getWidth(), sy + getHeight(),
                                    multipliedAlpha);
                        }
                    } else {
                        // Alpha is handled by the child directly, clobber the layer's alpha
                        mPrivateFlags |= PFLAG_ALPHA_SET;
                    }
                }
            }
        } else if ((mPrivateFlags & PFLAG_ALPHA_SET) == PFLAG_ALPHA_SET) {
            onSetAlpha(255);
            mPrivateFlags &= ~PFLAG_ALPHA_SET;
        }
    
        if (!drawingWithRenderNode) {
            // apply clips directly, since RenderNode won't do it for this draw
            if ((parentFlags & ViewGroup.FLAG_CLIP_CHILDREN) != 0 && cache == null) {
                if (offsetForScroll) {
                    canvas.clipRect(sx, sy, sx + getWidth(), sy + getHeight());
                } else {
                    if (!scalingRequired || cache == null) {
                        canvas.clipRect(0, 0, getWidth(), getHeight());
                    } else {
                        canvas.clipRect(0, 0, cache.getWidth(), cache.getHeight());
                    }
                }
            }
    
            if (mClipBounds != null) {
                // clip bounds ignore scroll
                canvas.clipRect(mClipBounds);
            }
        }
    
        if (!drawingWithDrawingCache) {
            if (drawingWithRenderNode) {
                mPrivateFlags &= ~PFLAG_DIRTY_MASK;
                ((DisplayListCanvas) canvas).drawRenderNode(renderNode);
            } else {
                // Fast path for layouts with no backgrounds
                if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                    mPrivateFlags &= ~PFLAG_DIRTY_MASK;
                    dispatchDraw(canvas);
                } else {
                    draw(canvas);
                }
            }
        } else if (cache != null) {
            mPrivateFlags &= ~PFLAG_DIRTY_MASK;
            if (layerType == LAYER_TYPE_NONE) {
                // no layer paint, use temporary paint to draw bitmap
                Paint cachePaint = parent.mCachePaint;
                if (cachePaint == null) {
                    cachePaint = new Paint();
                    cachePaint.setDither(false);
                    parent.mCachePaint = cachePaint;
                }
                cachePaint.setAlpha((int) (alpha * 255));
                canvas.drawBitmap(cache, 0.0f, 0.0f, cachePaint);
            } else {
                // use layer paint to draw the bitmap, merging the two alphas, but also restore
                int layerPaintAlpha = mLayerPaint.getAlpha();
                mLayerPaint.setAlpha((int) (alpha * layerPaintAlpha));
                canvas.drawBitmap(cache, 0.0f, 0.0f, mLayerPaint);
                mLayerPaint.setAlpha(layerPaintAlpha);
            }
        }
    
        if (restoreTo >= 0) {
            canvas.restoreToCount(restoreTo);
        }
    
        if (a != null && !more) {
            if (!hardwareAcceleratedCanvas && !a.getFillAfter()) {
                onSetAlpha(255);
            }
            parent.finishAnimatingView(this, a);
        }
    
        if (more && hardwareAcceleratedCanvas) {
            if (a.hasAlpha() && (mPrivateFlags & PFLAG_ALPHA_SET) == PFLAG_ALPHA_SET) {
                // alpha animations should cause the child to recreate its display list
                invalidate(true);
            }
        }
    
        mRecreateDisplayList = false;
    
        return more;
    }

上面这段代码，有两个需要指出的地方，一个就是canvas.translate()函数，这里表明了ViewRootImpl传进来的Canvas需要根据子视图本身的布局大小进行裁减，也就是说屏幕上所有子视图的canvas都只是一块裁减后的大小的canvas，当然这也就是为什么子视图的canvas的坐标原点不是从屏幕左上角开始，而是它自身大小的左上角开始的原因。

第二个需要指出的是如果ViewGroup的子视图仍然是ViewGroup，那么它会回调dispatchDraw(canvas)进行分法，如果子视图是View，那么就调用View的draw(canvas)函数进行绘制。

### canvas介绍
Android官方关于canvas的介绍告诉开发者： 
在绘图时需要明确四个核心的东西(basic components)：

1. 用什么工具画？ 
这个小问题很简单，我们需要用一支画笔(Paint)来绘图。 
当然，我们可以选择不同颜色的笔，不同大小的笔。
2. 把图画在哪里呢？ 
我们把图画在了Bitmap上，它保存了所绘图像的各个像素(pixel)。 
也就是说Bitmap承载和呈现了画的各种图形。
3. 画的内容？ 
根据自己的需求画圆，画直线，画路径。
4. 怎么画？ 
调用canvas执行绘图操作。 
比如，canvas.drawCircle()，canvas.drawLine()，canvas.drawPath()将我们需要的图像画出来。

知道了绘图过程中必不可少的四样东西，我们就要看看该怎么样构建一个canvas了。 
在此依次分析canvas的两个构造方法Canvas( )和Canvas(Bitmap bitmap)

    /** * Construct an empty raster canvas. Use setBitmap() to specify a bitmap to 
    * draw into. The initial target density is {@link Bitmap#DENSITY_NONE}; 
    * this will typically be replaced when a target bitmap is set for the 
    * canvas.
    */
    public Canvas() { 
        if (!isHardwareAccelerated()) { 
            mNativeCanvasWrapper = initRaster(null); 
            mFinalizer = new CanvasFinalizer(mNativeCanvasWrapper); 
        } else { 
            mFinalizer = null; 
        } 
    }

请注意该构造的第一句注释。官方不推荐通过该无参的构造方法生成一个canvas。如果要这么做那就需要调用setBitmap( )为其设置一个Bitmap。
为什么Canvas非要一个Bitmap对象呢？原因很简单：Canvas需要一个Bitmap对象来保存像素，如果画的东西没有地方可以保存，又还有什么意义呢？
既然不推荐这么做，那就接着有参的构造方法。

    /** * Construct a canvas with the specified bitmap to draw into. The bitmap 
    * must be mutable. 
    * 
    * The initial target density of the canvas is the same as the given 
    * bitmap's density. 
    * 
    * @param bitmap Specifies a mutable bitmap for the canvas to draw into. 
    */
     public Canvas(Bitmap bitmap) {
          if (!bitmap.isMutable()) { 
            throw new IllegalStateException("Immutable bitmap passed to Canvas constructor"); 
          } 
          throwIfCannotDraw(bitmap); 
          mNativeCanvasWrapper = initRaster(bitmap); 
          mFinalizer = new CanvasFinalizer(mNativeCanvasWrapper); 
          mBitmap = bitmap; mDensity = bitmap.mDensity; 
      }

通过该构造方法为Canvas设置了一个Bitmap来保存所绘图像的像素信息。
好了，知道了怎么构建一个canvas就来看看怎么利用它进行绘图。 
下面是一个很简单的例子：

    private void drawOnBitmap(){
        Bitmap bitmap=Bitmap.createBitmap(800, 400, Bitmap.Config.ARGB_8888);
        Canvas canvas=new Canvas(bitmap);
        canvas.drawColor(Color.GREEN);
        Paint paint=new Paint();
        paint.setColor(Color.RED);
        paint.setTextSize(60);
        canvas.drawText("hello , everyone", 150, 200, paint);
        mImageView.setImageBitmap(bitmap);
    }
    
在此处为canvas设置一个Bitmap，然后利用canvas画了一小段文字，最后使用ImageView显示了Bitmap。 
好了，看到这有人就有疑问了： 
我们平常用得最多的View的onDraw()方法，为什么没有Bitmap也可以画出各种图形呢？ 
请注意onDraw( )的输入参数是一个canvas，它与我们自己创建的canvas不同。这个系统传递给我们的canvas来自于ViewRootImpl的Surface，
在绘图时系统将会SkBitmap设置到SkCanvas中并返回与之对应Canvas。所以，在onDraw()中也是有一个Bitmap的，只是这个Bitmap是由系统创建的罢了。

### canvas方法

我们可以调用canvas画各种图形，我们有时候还有对canvas做一些操作，比如旋转，剪裁，平移等等；有时候为了达到理想的效果，我们可能还需要一些特效。在此，对相关内容做一些介绍。

* canvas.translate
* canvas.rotate
* canvas.clipRect
* canvas.save和canvas.restore
* PorterDuffXfermode
* Bitmap和Matrix
* Shader
* PathEffect

## canvas.translate
从字面意思也可以知道它的作用是位移，那么这个位移到底是怎么实现的的呢？我们看段代码：
 
     protected void onDraw(Canvas canvas) {
         super.onDraw(canvas);
         canvas.drawColor(Color.GREEN);
         Paint paint=new Paint();
         paint.setTextSize(70);
         paint.setColor(Color.BLUE);
         canvas.drawText("蓝色字体为Translate前所画", 20, 80, paint);
         canvas.translate(100,300);
         paint.setColor(Color.BLACK);
         canvas.drawText("黑色字体为Translate后所画", 20, 80, paint);
    }

这段代码的主要操作： 
1 画一句话，请参见代码第7行 
2 使用translate在X方向平移了100个单位在Y方向平移了300个单位，请参见代码第8行 
3 再画一句话，请参见代码第10行

在执行了平移之后所画的文字的位置=平移前坐标+平移的单位。 
比如，平移后所画文字的实际位置为：120(20+100)和380(80+300)。 
这就是说，canvas.translate相当于移动了坐标的原点，移动了坐标系。 
这么说可能还是不够直观，那就上图： 