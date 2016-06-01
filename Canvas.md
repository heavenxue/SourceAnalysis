# canvas源码解析

- [简介](#简介)
- [探索](#探索)
    - [思路](#思路)
- [draw源码解析](#draw源码解析)
- [canvas解析](#canvas解析)
    - [从代码入手来看](#从代码入手来看)
    - [canvas介绍](#canvas介绍)
    - [canvas方法](#canvas方法)
- [参照](#参照)
        
    
  
  
## 简介

>Android中使用图形处理引擎，2D部分是android SDK内部自己提供，3D部分是用Open GL ES 1.0。今天我们主要要了解的是2D相关的，
如果你想看3D的话那么可以跳过这篇文章。<br/>
    大部分2D使用的api都在android.graphics和android.graphics.drawable包中。
他们提供了图形处理相关的： `Canvas`、`ColorFilter`、`Point`(点)和`RetcF`(矩形)等，还有一些动画相关的：`AnimationDrawable`、
`BitmapDrawable`和`TransitionDrawable`等。</br>
以图形处理来说，我们最常用到的就是在一个View上画一些图片、形状或者自定义的文本内容，这里我们都是使用Canvas来实现的。</br>
你可以获取View中的Canvas对象，绘制一些自定义形状，然后调用`View.invalidate`方法让View重新刷新，然后绘制一个新的形状，这样达到2D动画效果。</br>
下面我们就主要来了解下Canvas的使用方法。Canvas对象的获取方式有两种：
一种我们通过重写`View.onDraw`方法，View中的Canvas对象会被当做参数传递过来，我们操作这个Canvas，效果会直接反应在View中。</br>
另一种就是当你想创建一个Canvas对象时使用的方法： </br>

``` java
Canvas c =new Canvas(Bitmap.createBitmap(100,100,Bitmap.Config.ARGB_88880));
```

## 探索

### 思路
  我们知道一个View的绘制过程都必须经历三个重要的过程，也就是`measure`,`layout`和`draw`
既然一个view的绘制主要是这三步，那一定有一个开始的地方啊，就像一个类是从`main`函数执行一样，对于view的绘制开始，这里先给出结论，后面会分析原因，具体结论如下：
整个view的绘制流程是在`ViewRootImpl`类的`performTraversals()`方法（这个方法巨长）开始的，该函数做的执行过程主要是根据之前设置的状态，判断是否重新计算视图大小（measure）、
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
   刚才上面我们已经提到了入口，所以，上面的注释说是用来测RootView的，上面传入参数后这个函数走的是`MATCH_PARENT`,使用`MeasureSpec.makeMeasureSpec`方法组装一个`MeasureSpec`,`MeasureSpec`的`SpecMode`等于`EXACTLY`,`specSize`等于`WindowSize`,也就是为何根视图总是全屏的原因。
整个流程如下：
![github](https://github.com/heavenxue/SourceAnalysis/raw/master/pic/1.png "github")

所以draw过程也是在`ViewRootImpl`的`performTraversals()`方法内部调用的，其调用顺序在`measure()`和`layout()`之后，这里的mView对于Activity来说就是PhoneWindow.DectorView,ViewRootImpl中的代码会创建一个Canvas对象，然后调用View.draw()来执行具体的绘制工作。
view递归draw流程图如下:<br/>
![github](https://github.com/heavenxue/SourceAnalysis/raw/master/pic/2.png "github")</br>
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
可以看见，draw方法通过调用`drawBackground(canvas)`实现了背景绘制，看下源码：
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
可以看到，这里去调用了View的`onDraw()`方法，所以我们看下view的`onDraw()`方法（ViewGroup没有重写这个方法），如下：
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
我们来看下view的draw方法中的`dispatchDraw(canvas)`方法源码，可以看到：
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
view的`dispatchDraw`方法也是一个空方法，而且注释说明了如果view包含子类需要重写它，所以我们有必要看下ViewGroup的`dispatchDraw()`方法源码</br>
（这也就是说刚刚说的当前View的所有子view进行绘制，如果当前的View没有子view就不需要进行绘制的原因，因为如果是View调用该方法是空的，而viewGroup才实现），如下：

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


`dispatchDraw(Canvas)`核心代码就是通过for循环调用drawChild(canvas, child, drawingTime)方法对ViewGroup的每个子视图运用动画以及绘制。
可以看出，ViewGroup确实重写了View的`dispatchDraw()`方法，该方法内部会遍历每个子View,然后调用`drawChild()`方法，我们可以看下ViewGroup的`drawChild()`方法，如下：

    protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        return child.draw(canvas, this, drawingTime);
    }

可以看出，drawChild方法调用了子view的`draw`方法，所以说viewGroup类已经为我们重写了`dispatchDraw()`功能实现，我们一般不需要重写这个方法
### 第六步，对view的滚动条进行绘制
 可以看到，这里去调用了一下view的`onDrawScrollBars()`方法，所以看下它的源码如下：
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
总而言之，整个绘制流程就是：</br>
View的背景绘制---->保存Canvas的layers --->View本身内容的绘制---->子视图的绘制---->绘制渐变框---->滚动条的绘制
当不需要绘制Layer的时候第二步和第五步可能跳过。因此在绘制的时候，能省的layer尽可省，可以提高绘制效率
`onDraw()`和`dispatchDraw()`分别为View本身内容和子视图绘制的函数。
View和ViewGroup的`onDraw()`都是空实现，因为具体View如何绘制由设计者来决定的，默认不绘制任何东西。
ViewGroup复写了`dispatchDraw()`来对其子视图进行绘制，通常你自己定义的ViewGroup不应该对`dispatchDraw()`进行复写
因为它的默认实现体现了View系统的绘制流程，该流程所做的一系列工作你不用去管，你要做的就是复写`View.onDraw(Canvas)`方法或者`ViewGroup.draw(Canvas)`方法，
但在`ViewGroup.draw(Canvas)`方法调用前，记得先调用`super.draw(canvas)`方法，先去绘制基础的View，然后你可以在`ViewGroup.draw(Canvas)`方法里做一些自己的绘制，
在高级的自定义中会有这样的需求。

#### draw原理总结
可以看见，绘制过程就是把view对象绘制在屏幕上，整个`draw()`过程需要注意如下细节:

* 如果该view是一个ViewGroup，则需要递归绘制其包含的所有子view
* view默认不会绘制任何内容，真正的绘制都需要自己在子类中实现。
* view的绘制是借助`onDraw()`传入canvas类进行的
* 区分view动画和ViewGroup布局动画，前者指的是View自身的动画，可以通过setAnimation添加，后者专门针对ViewGroup显示内部子视图时设置动画，可以在xml布局文件中对ViewGroup设置layoutAnimation属性（譬如对LinearLayout设置子view在显示时出现逐行，随机等不同动画效果）
* 在获取画布剪切区（每个view的draw中传入的Canvas）时会自动处理掉padding,子view获取Canvas不用关注这些逻辑，只用关心如何绘制即可
* 默认情况下，子view的`viewGroup.drawChild`绘制顺序和子view被添加的顺序一致，但是你也可以重载`ViewGroup.getChildDrawingOrder()`方法提供不同的顺序

## canvas解析
### 从代码入手来看
好了，开始canvas之旅了，因此我们首先从ViewGroup的`dispatchDraw`开始入手，这里要传入一个Canvas，这个Canvas是由`ViewRootImpl.java`传入，此时的Canvas是一个画布
而`dispatchDraw`方法里面会调用了`drawChild(canvas, transientChild, drawingTime)`;这个方法里可以找到`child.draw(canvas, this, drawingTime)`;
继续看，指向了view的`draw`方法，这个函数不同于`draw(Canvas canvas)`函数，后者是view绘制开始的地方，下面是源码：
``` java
boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {
    ......
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
            ......
        }
    }
    ......
}
```
`child.draw(canvas, this,drawingTime)`肯定是处理了和父视图相关的逻辑，但对于View的绘制，最终调用的还是`View.draw(Canvas)`方法。</br>
上面这段代码，有两个需要指出的地方，一个就是`canvas.translate()`函数，这里表明了ViewRootImpl传进来的Canvas需要根据子视图本身的布局大小进行裁减，<br/>
也就是说屏幕上所有子视图的canvas都只是一块裁减后的大小的canvas，当然这也就是为什么子视图的canvas的坐标原点不是从屏幕左上角开始，而是它自身大小的左上角开始的原因。</br>

第二个需要指出的是如果ViewGroup的子视图仍然是ViewGroup，那么它会回调`dispatchDraw(canvas)`进行分法，如果子视图是View，那么就调用View的`draw(canvas)`函数进行绘制。
* invalidate()
请求重绘 View 树，即 draw 过程，假如视图发生大小没有变化就不会调用`layout()`过程，并且只绘制那些调用了`invalidate()`方法的 View。</br>
* requestLayout()
当布局变化的时候，比如方向变化，尺寸的变化，会调用该方法，在自定义的视图中，如果某些情况下希望重新测量尺寸大小，应该手动去调用该方法，它会触发measure()和layout()过程，但不会进行 draw。
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
比如，`canvas.drawCircle()`，`canvas.drawLine()`，`canvas.drawPath()`将我们需要的图像画出来。

知道了绘图过程中必不可少的四样东西，我们就要看看该怎么样构建一个canvas了。 
在此依次分析canvas的两个构造方法`Canvas( )`和`Canvas(Bitmap bitmap)`
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
我们平常用得最多的View的`onDraw()`方法，为什么没有Bitmap也可以画出各种图形呢？ 
请注意`onDraw( )`的输入参数是一个canvas，它与我们自己创建的canvas不同。这个系统传递给我们的canvas来自于ViewRootImpl的Surface，
在绘图时系统将会SkBitmap设置到SkCanvas中并返回与之对应Canvas。所以，`在onDraw()`中也是有一个Bitmap的，只是这个Bitmap是由系统创建的罢了。

### canvas方法

我们可以调用canvas画各种图形，我们有时候还有对canvas做一些操作，比如旋转，剪裁，平移等等；有时候为了达到理想的效果，我们可能还需要一些特效。在此，对相关内容做一些介绍。

* `canvas.translate`
* `canvas.rotate`
* `canvas.clipRect`
* `canvas.save和canvas.restore`
* `PorterDuffXfermode`
* `Bitmap和Matrix`
* `Shader`
* `PathEffect`

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
2 使用`translate`在X方向平移了100个单位在Y方向平移了300个单位，请参见代码第8行 <br/>
3 再画一句话，请参见代码第10行<br/>

在执行了平移之后所画的文字的位置=平移前坐标+平移的单位。 <br/>
比如，平移后所画文字的实际位置为：120(20+100)和380(80+300)。 <br/>
这就是说，`canvas.translate`相当于移动了坐标的原点，移动了坐标系。<br/> 
这么说可能还是不够直观，那就上图： <br/>

![github](https://github.com/heavenxue/SourceAnalysis/raw/master/pic/3.png "github")

#### canvas.rotate
与translate类似，可以用rotate实现旋转。`canvas.rotate`相当于把坐标系旋转了一定角度。
#### canvas.clipRect
`canvas.clipRect`表示剪裁操作，执行该操作后的绘制将显示在剪裁区域。<br/>
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
当我们调用了`canvas.clipRect( )`后，如果再继续画图那么所绘的图只会在所剪裁的范围内体现。</br> 
当然除了按照矩形剪裁以外，还可以有别的剪裁方式，比如：`canvas.clipPath( )`和`canvas.clipRegion( )`。</br>

#### canvas.save和canvas.restore
刚才在说`canvas.clipRect( )`时，有人可能有这样的疑问：在调用`canvas.clipRect( )`后，如果还需要在剪裁范围外绘图该怎么办？</br>
是不是系统有一个`canvas.restoreClipRect( )`方法呢？去看看官方的API就有点小失望了，我们期待的东西是不存在的；</br>
不过可以换种方式来实现这个需求，这就是即将要介绍的`canvas.save`和`canvas.restore`。看到这个玩意，</br>
可能绝大部分人就想起来了Activity中的`onSaveInstanceState`和`onRestoreInstanceState`这两者用来保存</br>
和还原Activity的某些状态和数据。canvas也可以这样么？</br>  
#### canvas.save
它表示画布的锁定。如果我们把一个妹子锁在屋子里，那么外界的刮风下雨就影响不到她了；</br>  
同理，如果对一个canvas执行了`save`操作就表示将已经所绘的图形锁定，之后的绘图就不会影响到原来画好的图形。 </br>  
既然不会影响到原本已经画好的图形，那之后的操作又发生在哪里呢？ </br>  
当执行`canvas.save( )`时会生成一个新的图层(Layer)，并且这个图层是透明的。此时，所有draw的方法都是在这个图层上进行，</br>  
所以不会对之前画好的图形造成任何影响。在进行一些绘制操作后再使用`canvas.restore()`</br>  
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
这个例子由刚才讲`canvas.clipRect( )`稍加修改而来 </br>
1 执行`canvas.save( )`锁定canvas，请参见代码第8行 </br>
2 在新的Layer上裁剪和绘图，请参见代码第9-13行 </br>
3 执行`canvas.restore( )`将Layer合并到原图，请参见代码第14行</br> 
4 继续在原图上绘制，请参见代码第15-16行</br>

在使用`canvas.save`和`canvas.restore`时需注意一个问题：</br> 
    `save( )`和`restore( )`最好配对使用，若`restore( )`的调用次数比`save( )`多可能会造成异常
    
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
3 为Paint设置`PorterDuffXfermode`，请参见代码第17-18行</br> 
4 绘制原图，请参见代码第19行 </br>
纵观代码，发现一个陌生的东西`PorterDuffXfermode`而且陌生到了我们看到它的名字却不容易猜测其用途的地步；</br>
这在Android的源码中还是很少有的。 我以前郁闷了很久，不知道它为什么叫这个名字，</br>
直到后来看到《Android多媒体开发高级编程》才略知其原委。Thomas Porter和Tom Duff于1984年在ACM SIGGRAPH计算机图形学</br>
刊物上发表了《Compositing digital images》。在这篇文章中详细介绍了一系列不同的规则用于彼此重叠地绘制图像；</br>
这些规则中定义了哪些图像的哪些部分将出现在输出结果中。</br>

这就是PorterDuffXfermode的名字由来及其核心作用。</br>
现将PorterDuffXfermode描述的规则做一个介绍： </br>

![github](https://github.com/heavenxue/SourceAnalysis/raw/master/pic/4.png "github")

* `PorterDuff.Mode.CLEAR ` 绘制不会提交到画布上
* `PorterDuff.Mode.SRC` 只显示绘制源图像
* `PorterDuff.Mode.DST` 只显示目标图像，即已在画布上的初始图像
* `PorterDuff.Mode.SRC_OVER` 正常绘制显示，即后绘制的叠加在原来绘制的图上
* `PorterDuff.Mode.DST_OVER` 上下两层都显示但是下层(DST)居上显示
* `PorterDuff.Mode.SRC_IN` 取两层绘制的交集且只显示上层(SRC)
* `PorterDuff.Mode.DST_IN` 取两层绘制的交集且只显示下层(DST)
* `PorterDuff.Mode.SRC_OUT` 取两层绘制的不相交的部分且只显示上层(SRC)
* `PorterDuff.Mode.DST_OUT` 取两层绘制的不相交的部分且只显示下层(DST)
* `PorterDuff.Mode.SRC_ATOP` 两层相交，取下层(DST)的非相交部分和上层(SRC)的相交部分
* `PorterDuff.Mode.DST_ATOP` 两层相交，取上层(SRC)的非相交部分和下层(DST)的相交部分
* `PorterDuff.Mode.XOR` 挖去两图层相交的部分
* `PorterDuff.Mode.DARKEN` 显示两图层全部区域且加深交集部分的颜色
* `PorterDuff.Mode.LIGHTEN` 显示两图层全部区域且点亮交集部分的颜色
* `PorterDuff.Mode.MULTIPLY` 显示两图层相交部分且加深该部分的颜色
* `PorterDuff.Mode.SCREEN` 显示两图层全部区域且将该部分颜色变为透明色

了解了这些规则，再回头看我们刚才例子中的代码，就好理解多了。<br/> 
我们先画了一个圆角矩形，然后设置了`PorterDuff.Mode`为`SRC_IN`，最后绘制了原图。</br> 
所以，它会取圆角矩形和原图相交的部分但只显示原图部分；这样就形成了圆角的Bitmap。</br>

#### Bitmap和Matrix
除了刚才提到的给图片设置圆角之外，在开发中还常有其他涉及到图片的操作，比如图片的旋转，缩放，平移等等，这些操作可以结合`Matrix`来实现。 </br>
在此举个例子，看看利用`matrix`实现图片的平移和缩放。
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

在这里主要涉及到了`Matrix`。我们可以通过这个矩阵来实现对图片的一些操作。</br>
有人可能会问：利用`Matrix`实现了图片的平移`Translate`是将坐标系进行了平移么？</br>
不是的。`Matrix`所操作的是原图的每个像素点，它和坐标系是没有关系的。比如Scale是对每个像素点都进行了缩放，例如：</br>

    matrix.postScale(0.5f, 0.5f);

将原图的每个像素点的X的坐标都缩放成了原本的0.5</br> 
将原图的每个像素点的Y坐标也都缩放成了原本的0.5</br>

同样的道理在调用`matrix.setTranslate( )`时是对于原图中的每个像素都执行了位移操作。</br>

在使用`Matrix`时经常用到一系列的`set`，`pre`，`post`方法。它们有什么区别呢？它们的调用顺序又会对实际效果有什么影响呢？</br>

在此对该问题做一个总结： </br>
在调用`set`，`pre`，`post`时可视为将这些方法插入到一个队列。</br>

`pre`表示在队头插入一个方法</br>
`post`表示在队尾插入一个方法</br>
`set`表示清空队列 </br>
队列中只保留该`set`方法，其余的方法都会清除。</br>
当执行了一次`set`后`pre`总是插入到set之前的队列的最前面；post总是插入到set之后的队列的最后面。</br> 
也可以这么简单的理解： </br>
`set`在队列的中间位置，`per`执行队头插入，`post`执行队尾插入。</br> 
当绘制图像时系统会按照队列中从头至尾的顺序依次调用这些方法。 </br>
请看下面的几个小示例：</br>
``` java
Matrix m = new Matrix();
m.setRotate(45); 
m.setTranslate(80, 80);
```
只有`m.setTranslate(80, 80)`有效，因为`m.setRotate(45)`被清除.
``` java
Matrix m = new Matrix();
m.setTranslate(80, 80);
m.postRotate(45);
```
先执行`m.setTranslate(80, 80)`后执行`m.postRotate(45)`
``` java
    Matrix m = new Matrix();
    m.setTranslate(80, 80);
    m.preRotate(45);
```

先执行`m.preRotate(45)`后执行`m.setTranslate(80, 80)`
``` java
Matrix m = new Matrix();
m.preScale(2f,2f);    
m.preTranslate(50f, 20f);   
m.postScale(0.2f, 0.5f);    
m.postTranslate(20f, 20f);  
```   
执行顺序： 
`m.preTranslate(50f, 20f)`–>`m.preScale(2f,2f)`–>`m.postScale(0.2f, 0.5f)`–>`m.postTranslate(20f, 20f)`
``` java
Matrix m = new Matrix();
m.postTranslate(20, 20);   
m.preScale(0.2f, 0.5f);
m.setScale(0.8f, 0.8f);   
m.postScale(3f, 3f);
m.preTranslate(0.5f, 0.5f); 
```

执行顺序： 
`m.preTranslate(0.5f, 0.5f)`–>`m.setScale(0.8f, 0.8f)`–>`m.postScale(3f, 3f)`

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

在开发中调用`paint.setShader(Shader shader)`就可以实现渲染效果，在此以常用的BitmapShader为示例实现圆形图片。</br>
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
``` java
BitmapShader(Bitmap bitmap, TileMode tileX, TileMode tileY)
```
第一个参数： </br>
bitmap表示在渲染的对象</br> 
第二个参数： </br>
tileX 表示在位图上X方向渲染器平铺模式(TileMode)</br> 
TileMode一共有三种：</br>

* REPEAT ：重复
* MIRROR ：镜像
* CLAMP：拉伸
这三种效果类似于给电脑屏幕设置屏保时，若图片太小可选择重复，拉伸，镜像。</br> 
若选择`REPEAT`(重复 )：横向或纵向不断重复显示bitmap </br>
若选择`MIRROR`(镜像)：横向或纵向不断翻转重复 </br>
若选择`CLAMP`(拉伸) ：横向或纵向拉伸图片在该方向的最后一个像素。这点和设置电脑屏保有些不同</br> 
第三个参数： </br>
tileY表示在位图上Y方向渲染器平铺模式(TileMode)。与tileX同理，不再赘述。</br>

#### PathEffect
我们可以通过`canvas.drawPath( )`绘制一些简单的路径。但是假若需要给路径设置一些效果或者样式，这时候就要用到`PathEffect`了。

PathEffect有如下几个子类：

* CornerPathEffect 用平滑的方式衔接`Path`的各部分
* DashPathEffect 将`Path`的线段虚线化
* PathDashPathEffect 与`DashPathEffect`效果类似但需要自定义路径虚线的样式
* DiscretePathEffect 离散路径效果
* ComposePathEffect 两种样式的组合。先使用第一种效果然后在此基础上应用第二种效果
* SumPathEffect 两种样式的叠加。先将两种路径效果叠加起来再作用于`Path`

在此以`CornerPathEffect`和`DashPathEffect`为示例：
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
1 设置`Path为CornerPathEffect`效果，请参见代码第16行</br> 
在构建`CornerPathEffect`时传入了`radiu`s，它表示圆角的度数 </br>
2 设置`Path`为`DashPathEffect`效果，请参见代码第19行 </br>
在构建`DashPathEffect`时传入的参数要稍微复杂些。 </br>
`DashPathEffec`t构造方法的第一个参数： </br>
数组float[ ] { }中第一个数表示每条实线的长度，第二个数表示每条虚线的长度。</br> 
`DashPathEffect`构造方法的第二个参数： </br>
`phase`表示偏移量，动态改变该值会使路径产生动画效果</br>



## 参照
https://developer.android.com/guide/topics/graphics/2d-graphics.html</br>
http://www.2cto.com/kf/201401/269901.html</br>
http://blog.csdn.net/yanbober/article/details/46128379</br>
http://gold.xitu.io/entry/57465c88c4c971005d6e4422</br>
http://p.codekk.com/blogs/detail/54cfab086c4761e5001b253f</br>

