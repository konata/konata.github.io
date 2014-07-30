---
layout: post
title:  "NineOldAndroid 原理分析"
date:   2014-07-30 01:06:25
categories: Mobile
---

### NineOldAndroid 原理分析

在NineOldAndroids中提供了一系列方法设置属性和做Animator兼容的动画,我们以设置属性(scaleX)为例,分析整个过程

{% highlight java %}
ViewHelper.setScaleX(view,1.5f);
{% endhighlight %}

执行的代码如下

{% highlight java %}
public static void setScaleX(View view, float scaleX){
	if (NEEDS_PROXY) {
		wrap(view).setScaleX(scaleX);
	} else {
		Honeycomb.setScaleX(view, scaleX);
	}
}
{% endhighlight %}
 
其中wrap代码如下

	 
{% highlight java %} 
public static AnimatorProxy wrap(View view) {
	AnimatorProxy proxy = PROXIES.get(view);
	if (proxy == null || proxy != view.getAnimation()) {
		proxy = new AnimatorProxy(view);
		PROXIES.put(view, proxy);
	}
	return proxy;
 }
{% endhighlight %}
 
AnimatorProxy构造函数代码如下
 
 
{% highlight java %}
private AnimatorProxy(View view) {
	setDuration(0); //perform transformation immediately
	setFillAfter(true); //persist transformation beyond duration
	view.setAnimation(this);
	mView = new WeakReference<View>(view);
}
{% endhighlight %}
 
 
 需要注意AnimatorProxy是Animation的子类
 
 
{% highlight java %}
 public final class AnimatorProxy extends Animation {
	 @Override
	 protected void applyTransformation(float interpolatedTime, Transformation t) {
		 View view = mView.get();
		 if (view != null) {
			 t.setAlpha(mAlpha);
			 transformMatrix(t.getMatrix(), view);
		 }
	 }
 }
{% endhighlight %}
 
 
 所以到目前为止我们涉及到view的部分我们只做过一件事情,就是给它加了一个animation,这一点很重要,因为后面的所有事情都是围绕这一点来做的,
 继续来看setScaleX做过些什么
 
 
{% highlight java %}
public void setScaleX(float scaleX) {
	if (mScaleX != scaleX) {
		prepareForUpdate();   // step 1
		mScaleX = scaleX;
		invalidateAfterUpdate(); // step 2
	}
}
{% endhighlight %}
 
 
 step 1处的调用关系比较多,我就只把关键的部分整理出来
 
{% highlight java %}
transformMatrix(m, view);
mTempMatrix.mapRect(r);
{% endhighlight %}
 
 
transformMatrix这个函数比较重要，他在整个过程中将被调用三次，基本就是这个库的核心了，我们需要注意两点，包括他传进来的参数以及函数内部做的操作，函数的过程其实倒也不复杂，对于不熟悉线性代数的人（比如我）也可以大致猜出它的作用，
首先看一下他的全部实现，然后我们在详细讲解具体的每一步

 
{% highlight java %}
 private void transformMatrix(Matrix m, View view) {
	 final float w = view.getWidth();
	 final float h = view.getHeight();
	 final boolean hasPivot = mHasPivot;
	 final float pX = hasPivot ? mPivotX : w / 2f;
	 final float pY = hasPivot ? mPivotY : h / 2f;
	 final float rX = mRotationX;
	 final float rY = mRotationY;
	 final float rZ = mRotationZ;
	 if ((rX != 0) || (rY != 0) || (rZ != 0)) {
		 final Camera camera = mCamera;
		 camera.save();
		 camera.rotateX(rX);
		 camera.rotateY(rY);
		 camera.rotateZ(-rZ);
		 camera.getMatrix(m);
		 camera.restore();
		 m.preTranslate(-pX, -pY);
		 m.postTranslate(pX, pY);
	 }
	 final float sX = mScaleX;
	 final float sY = mScaleY;
	 if ((sX != 1.0f) || (sY != 1.0f)) {
		 m.postScale(sX, sY);
		 final float sPX = -(pX / w) * ((sX * w) - w);
		 final float sPY = -(pY / h) * ((sY * h) - h);
		 m.postTranslate(sPX, sPY);
	 }
	 m.postTranslate(mTranslationX, mTranslationY);
}
{% endhighlight %}


先看对于rotateX/Y/Z的处理，拿到mRotationX，Y，Z，并且对于没有设置旋转中心点的情况，设置默认的几何中心为旋转中心点，
首先直接用camera获取rotate变化的矩阵，再对矩阵进行preTranslate和postTranslate变化，这样就相当于先移到中心点，再做rotate变换，变化之后再移动到原点，这样得出来的变换矩阵就在m里面了，
	
{% highlight java %}
final float w = view.getWidth();
	 final float h = view.getHeight();
	 final boolean hasPivot = mHasPivot;
	 final float pX = hasPivot ? mPivotX : w / 2f;
	 final float pY = hasPivot ? mPivotY : h / 2f;
	 final float rX = mRotationX;
	 final float rY = mRotationY;
	 final float rZ = mRotationZ;
	 if ((rX != 0) || (rY != 0) || (rZ != 0)) {
		 final Camera camera = mCamera;
		 camera.save();
		 camera.rotateX(rX);
		 camera.rotateY(rY);
		 camera.rotateZ(-rZ);
		 camera.getMatrix(m);
		 camera.restore();
		 m.preTranslate(-pX, -pY);
		 m.postTranslate(pX, pY);
	 }
{% endhighlight %}



然后是scale和translate的处理，因为先做的scale，scale处理完成之后也要注意旋转中心点对于scale的矫正，也就是

{% highlight java %}
m.postTranslate(sPX, sPY);
{% endhighlight %}
的作用,最后在根据mTranslationX，mTranslationY做translate操作，整个过程结束

{% highlight java %}
final float sX = mScaleX;
final float sY = mScaleY;
if ((sX != 1.0f) || (sY != 1.0f)) {
	m.postScale(sX, sY);
	final float sPX = -(pX / w) * ((sX * w) - w);
	final float sPY = -(pY / h) * ((sY * h) - h);
	m.postTranslate(sPX, sPY);
}
m.postTranslate(mTranslationX, mTranslationY);
{% endhighlight %}

在分析这个函数开始的时候我们说了注意调用的时机以及参数还记得么？这个函数总共会被调用三次，我们目前分析的是在step 1的代码，
step 1后面的代码才是  
	
{% highlight java %}
mScaleX = scaleX;
{% endhighlight %}
而默认的scaleX是1，也就是我们刚才做的整个过程中mScale为之前设置的数值或者默认值1，还没有使用我们新设置的scaleX，现在我们的transformMatrix做完了，继续看代码

{% highlight java %}
	private void computeRect(final RectF r, View view) {
	// compute current rectangle according to matrix transformation
	final float w = view.getWidth();
	final float h = view.getHeight();

	// use a rectangle at 0,0 to make sure we don't run into issues with scaling
	r.set(0, 0, w, h);

	final Matrix m = mTempMatrix;
	m.reset();
	transformMatrix(m, view); //step 1.1.1
	mTempMatrix.mapRect(r);

	r.offset(view.getLeft(), view.getTop());

	// Straighten coords if rotations flipped them
	if (r.right < r.left) {
		final float f = r.right;
		r.right = r.left;
		r.left = f;
	}
	if (r.bottom < r.top) {
		final float f = r.top;
		r.top = r.bottom;
		r.bottom = f;
	}
}
{% endhighlight %}

我们刚刚走完了step 1.1.1的流程，记住这里传进来RectF，就是mBefore的RectF，
mTempMatrix.mapRect(r); 是对r做mTempMatrix对应的变幻，得到的新的r直接写到r里面去
于是mBefore就是经过上面这个变换的Rect的结果，还是刚才说的那句话，因为我们的mScale还没设置，所以其实这个就是变换前的view对应的矩形的位置，那么为什么这么还没有设置之前就要求出RectF的变换呢？难道它不就是(0,0,w,h)而已么，这里我的理解是第一次设置这些属性，得出来肯定是(0,0,w,h)，但是如果是已经先设置过一次translate，然后在调用了一次scaleX，这样的话mBefore应该是translate的矩阵对应于(0,0,w,h)的矩形，而因为3.0以前的动画系统我们并不是真正修改了view的实际位置，所以就算我们先做了translate或者scaleX，然后直接对于view拿出来的rect也是原始的rect，而不是变换之后的Rect位置，也就是因为这个原因，他才必须在mScaleX = scaleX之前，就把现在已经设置过的动画变化对于view的位置做一次运算，得出这次动画之前的view的位置，后面的代码就很容易理解了，就是对参数做一些简单的修正，继续看代码step2

{% highlight java %}
    mScaleX = scaleX;
    invalidateAfterUpdate(); // step 2
{% endhighlight %}

有了前面的分析，理解这一步就毫不费力了，一句话，把做动画之后的矩阵位置设置到mAfter中

{% highlight java %}
     final RectF after = mAfter;
     computeRect(after, view);
     after.union(mBefore);
     ((View)view.getParent()).invalidate(
             (int) Math.floor(after.left),
             (int) Math.floor(after.top),
             (int) Math.ceil(after.right),
             (int) Math.ceil(after.bottom));
{% endhighlight %}

求出mAfter之后，用after对before做union操作，求出两个矩形的外接最小矩形，并且写到了after里面去，
然后就开始重点了，invalidate求出的外接最小矩形区域


我们知道invalidate会同步或者异步导致view的重绘,直接导致的函数是View#draw
在view.draw里面
跟我们相关的是有三步的

1. drawAnimation()
2. __applyMatrix__
3. onDraw()

中间的__applyMatrix__没有独立成一个函数,我是为了方便理解取名的
先看drawAnimation的调用,这里只整理关键代码
 
{% highlight java %}
13095         if (a != null) {
13096             more = drawAnimation(parent, drawingTime, a, scalingRequired);
{% endhighlight %}

其实现的关键内容
 
{% highlight java %}
12962             invalidationTransform = parent.mInvalidationTransformation;
12963             a.getTransformation(drawingTime, invalidationTransform, 1f);
{% endhighlight %}

 
看见没有,我们刚刚设置的animation(这里就是变量a)开始被调用了,getTransformation其实就是回调applyTransformation,这里需要注意的是我们回调时候设置的matrix内容被写到parent.mInvalidationTransformation这个变量里面去了,然后是__applyMatrix__关键代码
 
{% highlight java %}
13115 transformToApply = transformType != Transformation.TYPE_IDENTITY ?
13116    parent.mChildTransformation : null;
13231 if (concatMatrix) {
13232    if (useDisplayListProperties) {
13233 	     displayList.setAnimationMatrix(transformToApply.getMatrix());
13234    } else {
13237      canvas.translate(-transX, -transY);
13238      canvas.concat(transformToApply.getMatrix());
13239      canvas.translate(transX, transY);
13240    }
13241    parent.mGroupFlags |= ViewGroup.FLAG_CLEAR_TRANSFORMATION;
{% endhighlight %}

注意这里的transformToApply就是我们门刚刚的parent.mInvalidationTransformation,也就是我们的动画回掉applyTransformation修改的内容,于是一切都串起来了,我们在setScaleX里面先设置时长为0的动画,并且在动画回调里面返回了我们setScaleX应该做的矩阵变换,然后调用invalidate,经过一系列冗长的步骤之后在onDraw之前,draw函数顺利拿到这个matrix并且设置到canvas里面去,保证实现了最后的效果,为了确认,我们再看applyTransformation函数

{% highlight java %}
 @Override
 protected void applyTransformation(float interpolatedTime, Transformation t) {
	 View view = mView.get();
	 if (view != null) {
		 t.setAlpha(mAlpha);
		 transformMatrix(t.getMatrix(), view);
	 }
 }
{% endhighlight %}

果然是吧我们做的变换扔到T里面去
   
   
总结几点
   1. nineoldandroid没有什么特殊的能力做android原生做不到的东西,他能做的只是canvas能做的而已
   2. 对于alpha,不是做形变,而是直接设置到Transformation里面去,步骤其实跟上面类似
   3. 上面是针对android4.2的源码进行分析的,我调试时候发现不同版本的draw函数差距很大,其他版本估计具体流程不是这样,不过大致原理是相同的
   
   
   
   
   
   
   
   
   
   
   
   
