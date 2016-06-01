# canvas源码解析

- [简介](#简介)
- [探索](#探索)
    - [思路](#思路)
- [draw源码解析](#draw源码解析)
- [canvas源码解析](#canvas源码解析)
    - [从代码入手来看](#从代码入手来看)
    - [canvas介绍](#canvas介绍)
    - [canvas方法](#canvas方法)
- [参照](#参照)
        
    
  
  
## 简介

>Android中使用图形处理引擎，2D部分是android SDK内部自己提供，3D部分是用Open GL ES 1.0。今天我们主要要了解的是2D相关的，
如果你想看3D的话那么可以跳过这篇文章。<br/>
    大部分2D使用的api都在android.graphics和android.graphics.drawable包中。
他们提供了图形处理相关的： Canvas、ColorFilter、Point(点)和RetcF(矩形)等，还有一些动画相关的：AnimationDrawable、
BitmapDrawable和TransitionDrawable等。</br>
以图形处理来说，我们最常用到的就是在一个View上画一些图片、形状或者自定义的文本内容，这里我们都是使用Canvas来实现的。</br>
你可以获取View中的Canvas对象，绘制一些自定义形状，然后调用View.invalidate方法让View重新刷新，然后绘制一个新的形状，这样达到2D动画效果。</br>
下面我们就主要来了解下Canvas的使用方法。Canvas对象的获取方式有两种：
一种我们通过重写View.onDraw方法，View中的Canvas对象会被当做参数传递过来，我们操作这个Canvas，效果会直接反应在View中。</br>
另一种就是当你想创建一个Canvas对象时使用的方法： </br>
Canvas c =new Canvas(Bitmap.createBitmap(100,100,Bitmap.Config.ARGB_88880));

## 探索

### 思路
  我们知道一个View的绘制过程都必须经历三个重要的过程，也就是measure,layout和draw
既然一个view的绘制主要是这三步，那一定有一个开始的地方啊，就像一个类是从main函数执行一样，对于view的绘制开始，这里先给出结论，后面会分析原因，具体结论如下：
整个view的绘制流程是在ViewRootImpl类的performTraversals()方法（这个方法巨长）开始的，该函数做的执行过程主要是根据之前设置的状态，判断是否重新计算视图大小（measure）、
是否重新放置视图的位置（layout）、是否重绘(draw)、其核心就是通过判断来选择顺序执行这三个方法中的哪个，如下：

``` java    
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
```
 
  好，既然这样，我们就从这里入手。

## draw源码解析
   刚才上面我们已经提到了入口，所以，上面的注释说是用来测RootView的，上面传入参数后这个函数走的是MATCH_PARENT,使用MeasureSpec.makeMeasureSpec方法组装一个MeasureSpec,MeasureSpec的SpecMode等于EXACTLY,specSize等于WindowSize,也就是为何根视图总是全屏的原因。
整个流程如下：
![github](https://github.com/heavenxue/SourceAnalysis/raw/master/pic/1.png "github")

所以draw过程也是在ViewRootImpl的performTraversals()方法内部调用的，其调用顺序在measure()和layout()之后，这里的mView对于Activity来说就是PhoneWindow.DectorView,ViewRootImpl中的代码会创建一个Canvas对象，然后调用View.draw()来执行具体的绘制工作。
view递归draw流程图如下:<br/>
![github](https://github.com/heavenxue/SourceAnalysis/raw/master/pic/2.png "github")
由于ViewGroup没有重写View的draw方法，所以下面直接从View的draw方法开始分析

``` java
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
```
看整个view的draw方法很复杂，但是注释很详细，从注释可以看出整个draw过程分6步。源码注释说（skip step 2 & 5 if possible (common case) ）第2步和第5步可以跳过，所以我们重点来看剩余4步，如下：
### 第一步，对view的背景进行绘制
可以看见，draw方法通过调用drawBackground(canvas)实现了背景绘制，看下源码：
``` java
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
```

### 第三步，对view的内容绘制
可以看到，这里去调用了View的onDraw()方法，所以我们看下view的onDraw()方法（ViewGroup没有重写这个方法），如下：
``` java
/**
 * Implement this to do your drawing.
 *
 * @param canvas the canvas on which the background will be drawn
 */
protected void onDraw(Canvas canvas) {
}
```

可以看到，是一个空方法，因为每个view的内容部分是各不相同的，所以要由子类去实现具体的逻辑
### 第四步，对当前的view的所有子view进行绘制，如果当前view没有子view就不需要绘制
我们来看下view的draw方法中的dispatchDraw(canvas)方法源码，可以看到：
``` java
/**
 * Called by draw to draw the child views. This may be overridden
 * by derived classes to gain control just before its children are drawn
 * (but after its own view has been drawn).
 * @param canvas the canvas on which to draw the view
 */
protected void dispatchDraw(Canvas canvas) {

}
```
view的dispatchDraw方法也是一个空方法，而且注释说明了如果view包含子类需要重写它，所以我们有必要看下ViewGroup的dispatchDraw()方法源码（这也就是说刚刚说的当前View的所有子view进行绘制，如果当前的View没有子view就不需要进行绘制的原因，因为如果是View调用该方法是空的，而viewGroup才实现），如下：

``` java
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
```

可以看出，ViewGroup确实重写了View的dispatchDraw()方法，该方法内部会遍历每个子View,然后调用drawChild()方法，我们可以看下ViewGroup的drawChild方法，如下：

    protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        return child.draw(canvas, this, drawingTime);
    }

可以看出，drawChild方法调用了子view的draw方法，所以说viewGroup类已经为我们重写了dispatchDraw()功能实现，我们一般不需要重写这个方法，但可以重载父类函数实现具体功能。
### 第六步，对view的滚动条进行绘制
 可以看到，这里去调用了一下view的onDrawScrollBars()方法，所以看下它的源码如下：
``` java
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
``` 
可以看见其实任何一个View都是有（水平垂直）滚动条的，只是一般情况下都不显示而已，到此，View的draw绘制部分已经分析完毕。

#### draw原理总结
可以看见，绘制过程就是把view对象绘制在屏幕上，整个draw()过程需要注意如下细节:

* 如果该view是一个ViewGroup，则需要递归绘制其包含的所有子view
* view默认不会绘制任何内容，真正的绘制都需要自己在子类中实现。
* view的绘制是借助onDraw()传入canvas类进行的
* 区分view动画和ViewGroup布局动画，前者指的是View自身的动画，可以通过setAnimation添加，后者专门针对ViewGroup显示内部子视图时设置动画，可以在xml布局文件中对ViewGroup设置layoutAnimation属性（譬如对LinearLayout设置子view在显示时出现逐行，随机等不同动画效果）
* 在获取画布剪切区（每个view的draw中传入的Canvas）时会自动处理掉padding,子view获取Canvas不用关注这些逻辑，只用关心如何绘制即可
* 默认情况下，子view的viewGroup.drawChild绘制顺序和子view被添加的顺序一致，但是你也可以重载ViewGroup.getChildDrawingOrder()方法提供不同的顺序

## canvas源码解析
### 从代码入手来看
好了，开始canvas之旅了，因此我们首先从ViewGroup的dispatchDraw开始入手，这里要传入一个Canvas，这个Canvas是由ViewRootImpl.java传入，此时的Canvas是一个画布
而dispatchDraw方法里面会调用了drawChild(canvas, transientChild, drawingTime);这个方法里可以找到child.draw(canvas, this, drawingTime);
继续看，指向了view的draw方法，这个函数不同于draw(Canvas canvas)函数，后者是view绘制开始的地方，下面是源码：
``` java
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
```

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
``` java
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
```

请注意该构造的第一句注释。官方不推荐通过该无参的构造方法生成一个canvas。如果要这么做那就需要调用setBitmap( )为其设置一个Bitmap。
为什么Canvas非要一个Bitmap对象呢？原因很简单：Canvas需要一个Bitmap对象来保存像素，如果画的东西没有地方可以保存，又还有什么意义呢？
既然不推荐这么做，那就接着有参的构造方法。
``` java
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
```
通过该构造方法为Canvas设置了一个Bitmap来保存所绘图像的像素信息。
好了，知道了怎么构建一个canvas就来看看怎么利用它进行绘图。 
下面是一个很简单的例子：
``` java
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
```

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

#### canvas.translate
从字面意思也可以知道它的作用是位移，那么这个位移到底是怎么实现的的呢？我们看段代码：
``` java
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
```

这段代码的主要操作： <br/>
1 画一句话，请参见代码第7行 <br/>
2 使用translate在X方向平移了100个单位在Y方向平移了300个单位，请参见代码第8行 <br/>
3 再画一句话，请参见代码第10行<br/>

在执行了平移之后所画的文字的位置=平移前坐标+平移的单位。 <br/>
比如，平移后所画文字的实际位置为：120(20+100)和380(80+300)。 <br/>
这就是说，canvas.translate相当于移动了坐标的原点，移动了坐标系。<br/> 
这么说可能还是不够直观，那就上图： <br/>

![github](https://github.com/heavenxue/SourceAnalysis/raw/master/pic/3.png "github")

#### canvas.rotate
与translate类似，可以用rotate实现旋转。canvas.rotate相当于把坐标系旋转了一定角度。
#### canvas.clipRect
canvas.clipRect表示剪裁操作，执行该操作后的绘制将显示在剪裁区域。<br/>
``` java
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    canvas.drawColor(Color.GREEN);
    Paint paint=new Paint();
    paint.setTextSize(60);
    paint.setColor(Color.BLUE);
    canvas.drawText("绿色部分为Canvas剪裁前的区域", 20, 80, paint);
    Rect rect=new Rect(20,200,900,1000);
    canvas.clipRect(rect);
    canvas.drawColor(Color.YELLOW);
    paint.setColor(Color.BLACK);
    canvas.drawText("黄色部分为Canvas剪裁后的区域", 10, 310, paint);
    }
```
当我们调用了canvas.clipRect( )后，如果再继续画图那么所绘的图只会在所剪裁的范围内体现。</br> 
当然除了按照矩形剪裁以外，还可以有别的剪裁方式，比如：canvas.clipPath( )和canvas.clipRegion( )。</br>

#### canvas.save和canvas.restore
刚才在说canvas.clipRect( )时，有人可能有这样的疑问：在调用canvas.clipRect( )后，如果还需要在剪裁范围外绘图该怎么办？</br>
是不是系统有一个canvas.restoreClipRect( )方法呢？去看看官方的API就有点小失望了，我们期待的东西是不存在的；</br>
不过可以换种方式来实现这个需求，这就是即将要介绍的canvas.save和canvas.restore。看到这个玩意，</br>
可能绝大部分人就想起来了Activity中的onSaveInstanceState和onRestoreInstanceState这两者用来保存</br>
和还原Activity的某些状态和数据。canvas也可以这样么？</br>  
#### canvas.save
它表示画布的锁定。如果我们把一个妹子锁在屋子里，那么外界的刮风下雨就影响不到她了；</br>  
同理，如果对一个canvas执行了save操作就表示将已经所绘的图形锁定，之后的绘图就不会影响到原来画好的图形。 </br>  
既然不会影响到原本已经画好的图形，那之后的操作又发生在哪里呢？ </br>  
当执行canvas.save( )时会生成一个新的图层(Layer)，并且这个图层是透明的。此时，所有draw的方法都是在这个图层上进行，</br>  
所以不会对之前画好的图形造成任何影响。在进行一些绘制操作后再使用canvas.restore()</br>  
将这个新的图层与底下原本的画好的图像相结合形成一个新的图像。</br>  
打个比方：原本在画板上画了一个姑娘，我又找了一张和画板一样大小的透明的纸(Layer)，然后在上面画了一朵花，</br>  
最后我把这个纸盖在了画板上，呈现给世人的效果就是：一个美丽的姑娘手拿一朵鲜花。 </br>  
看一下代码例子：</br>
``` java
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    canvas.drawColor(Color.GREEN);
    Paint paint=new Paint();
    paint.setTextSize(60);
    paint.setColor(Color.BLUE);
    canvas.drawText("绿色部分为Canvas剪裁前的区域", 20, 80, paint);
    canvas.save();
    Rect rect=new Rect(20,200,900,1000);
    canvas.clipRect(rect);
    canvas.drawColor(Color.YELLOW);
    paint.setColor(Color.BLACK);
    canvas.drawText("黄色部分为Canvas剪裁后的区域", 10, 310, paint);
    canvas.restore();
    paint.setColor(Color.RED);
    canvas.drawText("XXOO", 20, 170, paint);
}
```
这个例子由刚才讲canvas.clipRect( )稍加修改而来 </br>
1 执行canvas.save( )锁定canvas，请参见代码第8行 </br>
2 在新的Layer上裁剪和绘图，请参见代码第9-13行 </br>
3 执行canvas.restore( )将Layer合并到原图，请参见代码第14行</br> 
4 继续在原图上绘制，请参见代码第15-16行</br>

在使用canvas.save和canvas.restore时需注意一个问题：</br> 
    save( )和restore( )最好配对使用，若restore( )的调用次数比save( )多可能会造成异常
    
#### PorterDuffXfermode

 可以实现圆角图片，代码如下;
``` java 
/**
 * @param bitmap 原图
 * @param pixels 角度
 * @return 带圆角的图
 */
public Bitmap getRoundCornerBitmap(Bitmap bitmap, float pixels) {
    int width=bitmap.getWidth();
    int height=bitmap.getHeight();
    Bitmap roundCornerBitmap = Bitmap.createBitmap(width,height,Bitmap.Config.ARGB_8888);
    Canvas canvas = new Canvas(roundCornerBitmap);
    Paint paint = new Paint();
    paint.setColor(Color.BLACK);
    paint.setAntiAlias(true);
    Rect rect = new Rect(0, 0, width, height);
    RectF rectF = new RectF(rect);
    canvas.drawRoundRect(rectF, pixels, pixels, paint);
    PorterDuffXfermode xfermode=new PorterDuffXfermode(PorterDuff.Mode.SRC_IN);
    paint.setXfermode(xfermode);
    canvas.drawBitmap(bitmap, rect, rect, paint);
    return roundCornerBitmap;
}
```
主要操作如下： 
1 生成canvas，请参见代码第7-10行 </br>
注意给canvas设置的Bitmap的大小是和原图的大小一致的</br> 
2 绘制圆角矩形，请参见代码第11-16行 </br>
3 为Paint设置PorterDuffXfermode，请参见代码第17-18行</br> 
4 绘制原图，请参见代码第19行 </br>
纵观代码，发现一个陌生的东西PorterDuffXfermode而且陌生到了我们看到它的名字却不容易猜测其用途的地步；</br>
这在Android的源码中还是很少有的。 我以前郁闷了很久，不知道它为什么叫这个名字，</br>
直到后来看到《Android多媒体开发高级编程》才略知其原委。Thomas Porter和Tom Duff于1984年在ACM SIGGRAPH计算机图形学</br>
刊物上发表了《Compositing digital images》。在这篇文章中详细介绍了一系列不同的规则用于彼此重叠地绘制图像；</br>
这些规则中定义了哪些图像的哪些部分将出现在输出结果中。</br>

这就是PorterDuffXfermode的名字由来及其核心作用。</br>
现将PorterDuffXfermode描述的规则做一个介绍： </br>

![github](https://github.com/heavenxue/SourceAnalysis/raw/master/pic/4.png "github")

* PorterDuff.Mode.CLEAR 
绘制不会提交到画布上
* PorterDuff.Mode.SRC 
只显示绘制源图像
* PorterDuff.Mode.DST 
只显示目标图像，即已在画布上的初始图像
* PorterDuff.Mode.SRC_OVER 
正常绘制显示，即后绘制的叠加在原来绘制的图上
* PorterDuff.Mode.DST_OVER 
上下两层都显示但是下层(DST)居上显示
* PorterDuff.Mode.SRC_IN 
取两层绘制的交集且只显示上层(SRC)
* PorterDuff.Mode.DST_IN 
取两层绘制的交集且只显示下层(DST)
* PorterDuff.Mode.SRC_OUT 
取两层绘制的不相交的部分且只显示上层(SRC)
* PorterDuff.Mode.DST_OUT 
取两层绘制的不相交的部分且只显示下层(DST)
* PorterDuff.Mode.SRC_ATOP 
两层相交，取下层(DST)的非相交部分和上层(SRC)的相交部分
* PorterDuff.Mode.DST_ATOP 
两层相交，取上层(SRC)的非相交部分和下层(DST)的相交部分
* PorterDuff.Mode.XOR 
挖去两图层相交的部分
* PorterDuff.Mode.DARKEN 
显示两图层全部区域且加深交集部分的颜色
* PorterDuff.Mode.LIGHTEN 
显示两图层全部区域且点亮交集部分的颜色
* PorterDuff.Mode.MULTIPLY 
显示两图层相交部分且加深该部分的颜色
* PorterDuff.Mode.SCREEN 
显示两图层全部区域且将该部分颜色变为透明色

了解了这些规则，再回头看我们刚才例子中的代码，就好理解多了。<br/> 
我们先画了一个圆角矩形，然后设置了PorterDuff.Mode为SRC_IN，最后绘制了原图。</br> 
所以，它会取圆角矩形和原图相交的部分但只显示原图部分；这样就形成了圆角的Bitmap。</br>

#### Bitmap和Matrix
除了刚才提到的给图片设置圆角之外，在开发中还常有其他涉及到图片的操作，比如图片的旋转，缩放，平移等等，这些操作可以结合Matrix来实现。 </br>
在此举个例子，看看利用matrix实现图片的平移和缩放。
``` java
private void drawBitmapWithMatrix(Canvas canvas){
    Paint paint = new Paint();
    paint.setAntiAlias(true);
    Bitmap bitmap = BitmapFactory.decodeResource(getResources(),R.drawable.mm);
    int width=bitmap.getWidth();
    int height=bitmap.getHeight();
    Matrix matrix = new Matrix();
    canvas.drawBitmap(bitmap, matrix, paint);
    matrix.setTranslate(width/2, height);
    canvas.drawBitmap(bitmap, matrix, paint);
    matrix.postScale(0.5f, 0.5f);
    
    canvas.drawBitmap(bitmap, matrix, paint);
}
```   
梳理一下这段代码的主要操作： </br>
1 画出原图，请参见代码第2-8行</br> 
2 平移原图，请参见代码第9-10行 </br>
3 缩放原图，请参见代码第11-13行 </br>

在这里主要涉及到了Matrix。我们可以通过这个矩阵来实现对图片的一些操作。</br>
有人可能会问：利用Matrix实现了图片的平移(Translate)是将坐标系进行了平移么？</br>
不是的。Matrix所操作的是原图的每个像素点，它和坐标系是没有关系的。比如Scale是对每个像素点都进行了缩放，例如：</br>

    matrix.postScale(0.5f, 0.5f);

将原图的每个像素点的X的坐标都缩放成了原本的0.5</br> 
将原图的每个像素点的Y坐标也都缩放成了原本的0.5</br>

同样的道理在调用matrix.setTranslate( )时是对于原图中的每个像素都执行了位移操作。</br>

在使用Matrix时经常用到一系列的set，pre，post方法。它们有什么区别呢？它们的调用顺序又会对实际效果有什么影响呢？</br>

在此对该问题做一个总结： </br>
在调用set，pre，post时可视为将这些方法插入到一个队列。</br>

pre表示在队头插入一个方法</br>
post表示在队尾插入一个方法</br>
set表示清空队列 </br>
队列中只保留该set方法，其余的方法都会清除。</br>
当执行了一次set后pre总是插入到set之前的队列的最前面；post总是插入到set之后的队列的最后面。</br> 
也可以这么简单的理解： </br>
set在队列的中间位置，per执行队头插入，post执行队尾插入。</br> 
当绘制图像时系统会按照队列中从头至尾的顺序依次调用这些方法。 </br>
请看下面的几个小示例：</br>
``` java
Matrix m = new Matrix();
m.setRotate(45); 
m.setTranslate(80, 80);
```
只有m.setTranslate(80, 80)有效，因为m.setRotate(45)被清除.
``` java
Matrix m = new Matrix();
m.setTranslate(80, 80);
m.postRotate(45);
```
先执行m.setTranslate(80, 80)后执行m.postRotate(45)

    Matrix m = new Matrix();
    m.setTranslate(80, 80);
    m.preRotate(45);

先执行m.preRotate(45)后执行m.setTranslate(80, 80)
``` java
Matrix m = new Matrix();
m.preScale(2f,2f);    
m.preTranslate(50f, 20f);   
m.postScale(0.2f, 0.5f);    
m.postTranslate(20f, 20f);  
```   
执行顺序： 
m.preTranslate(50f, 20f)–>m.preScale(2f,2f)–>m.postScale(0.2f, 0.5f)–>m.postTranslate(20f, 20f)
``` java
Matrix m = new Matrix();
m.postTranslate(20, 20);   
m.preScale(0.2f, 0.5f);
m.setScale(0.8f, 0.8f);   
m.postScale(3f, 3f);
m.preTranslate(0.5f, 0.5f); 
```

执行顺序： 
m.preTranslate(0.5f, 0.5f)–>m.setScale(0.8f, 0.8f)–>m.postScale(3f, 3f)

#### Shader
有时候我们需要实现图像的渐变效果，这时候Shader就派上用场啦。 </br>
先来瞅瞅啥是Shader：</br>
 Android提供的Shader类主要用于渲染图像以及几何图形。 </br>
 Shader的主要子类如下：</br>
 
* BitmapShader———图像渲染
* LinearGradient——–线性渲染
* RadialGradient——–环形渲染
* SweepGradient——–扫描渲染
* ComposeShader——组合渲染

在开发中调用paint.setShader(Shader shader)就可以实现渲染效果，在此以常用的BitmapShader为示例实现圆形图片。</br>
``` java
protected void onDraw(Canvas canvas) {
     super.onDraw(canvas);
     Paint paint = new Paint();
     paint.setAntiAlias(true);
     Bitmap bitmap = BitmapFactory.decodeResource(getResources(),R.drawable.mm);
     int radius = bitmap.getWidth()/2;
     BitmapShader bitmapShader = new BitmapShader(bitmap,Shader.TileMode.REPEAT,Shader.TileMode.REPEAT);
     paint.setShader(bitmapShader);
     canvas.translate(250,430);
     canvas.drawCircle(radius, radius, radius, paint);
}
```
1 生成BitmapShader，请参见代码第7行 </br>
2 为Paint设置Shader，请参见代码第8行</br> 
3 画出圆形图片，请参见代码第10行</br>

在这段代码中，可能稍感陌生的就是BitmapShader构造方法。</br>

    BitmapShader(Bitmap bitmap, TileMode tileX, TileMode tileY)

第一个参数： </br>
bitmap表示在渲染的对象</br> 
第二个参数： </br>
tileX 表示在位图上X方向渲染器平铺模式(TileMode)</br> 
TileMode一共有三种：</br>

* REPEAT ：重复
* MIRROR ：镜像
* CLAMP：拉伸
这三种效果类似于给电脑屏幕设置屏保时，若图片太小可选择重复，拉伸，镜像。</br> 
若选择REPEAT(重复 )：横向或纵向不断重复显示bitmap </br>
若选择MIRROR(镜像)：横向或纵向不断翻转重复 </br>
若选择CLAMP(拉伸) ：横向或纵向拉伸图片在该方向的最后一个像素。这点和设置电脑屏保有些不同</br> 
第三个参数： </br>
tileY表示在位图上Y方向渲染器平铺模式(TileMode)。与tileX同理，不再赘述。</br>

#### PathEffect
我们可以通过canvas.drawPath( )绘制一些简单的路径。但是假若需要给路径设置一些效果或者样式，这时候就要用到PathEffect了。

PathEffect有如下几个子类：

* CornerPathEffect 用平滑的方式衔接Path的各部分
* DashPathEffect 将Path的线段虚线化
* PathDashPathEffect 与DashPathEffect效果类似但需要自定义路径虚线的样式
* DiscretePathEffect 离散路径效果
* ComposePathEffect 两种样式的组合。先使用第一种效果然后在此基础上应用第二种效果
* SumPathEffect 两种样式的叠加。先将两种路径效果叠加起来再作用于Path

在此以CornerPathEffect和DashPathEffect为示例：
``` java
protected void onDraw(Canvas canvas) {
     super.onDraw(canvas);
     canvas.translate(0,300);
     Paint paint = new Paint();
     paint.setAntiAlias(true);
     paint.setStyle(Paint.Style.STROKE);
     paint.setColor(Color.GREEN);
     paint.setStrokeWidth(8);
     Path  path = new Path();
     path.moveTo(15, 60);
     for (int i = 0; i <= 35; i++) {
          path.lineTo(i * 30, (float) (Math.random() * 150));
      }
     canvas.drawPath(path, paint);
     canvas.translate(0, 400);
     paint.setPathEffect(new CornerPathEffect(60));
     canvas.drawPath(path, paint);
     canvas.translate(0, 400);
     paint.setPathEffect(new DashPathEffect(new float[] {15, 8}, 1));
     canvas.drawPath(path, paint);
}
```   
分析一下这段代码中的主要操作： </br>
1 设置Path为CornerPathEffect效果，请参见代码第16行</br> 
在构建CornerPathEffect时传入了radius，它表示圆角的度数 </br>
2 设置Path为DashPathEffect效果，请参见代码第19行 </br>
在构建DashPathEffect时传入的参数要稍微复杂些。 </br>
DashPathEffect构造方法的第一个参数： </br>
数组float[ ] { }中第一个数表示每条实线的长度，第二个数表示每条虚线的长度。</br> 
DashPathEffect构造方法的第二个参数： </br>
phase表示偏移量，动态改变该值会使路径产生动画效果</br>



## 参照
https://developer.android.com/guide/topics/graphics/2d-graphics.html</br>
http://www.2cto.com/kf/201401/269901.html</br>
http://blog.csdn.net/yanbober/article/details/46128379</br>
http://gold.xitu.io/entry/57465c88c4c971005d6e4422</br>
http://p.codekk.com/blogs/detail/54cfab086c4761e5001b253f</br>

