NineOldAndroid 原理解析

1. 在NineOldAndroids中提供了一系列方法设置属性和做Animator兼容的动画,我们以设置属性(scaleX)为例,分析整个过程

   ```
   ViewHelper.setScaleX(view,1.5f);
   ```
   
   执行的代码如下
   
	```
     public static void setScaleX(View view, float scaleX)		{
        if (NEEDS_PROXY) {
            wrap(view).setScaleX(scaleX);
        } else {
            Honeycomb.setScaleX(view, scaleX);
        }
    }
    ```
    
    其中wrap代码如下
    
    ```    
	 public static AnimatorProxy wrap(View view) {
        AnimatorProxy proxy = PROXIES.get(view);
        if (proxy == null || proxy != view.getAnimation()) {
            proxy = new AnimatorProxy(view);
            PROXIES.put(view, proxy);
        }
        return proxy;
    }
    ```
    
    AnimatorProxy构造函数代码如下
    
    
    ```
     private AnimatorProxy(View view) {
        setDuration(0); //perform transformation immediately
        setFillAfter(true); //persist transformation beyond duration
        view.setAnimation(this);
        mView = new WeakReference<View>(view);
    }
    ```
    
    
    需要注意AnimatorProxy是Animation的子类
    
    
    ```
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
    ```
    
    
    所以到目前为止我们涉及到view的部分我们只做过一件事情,就是给它加了一个animation,这一点很重要,因为后面的所有事情都是围绕这一点来做的,
    继续来看setScaleX做过些什么
    
    
    ```
      public void setScaleX(float scaleX) {
        if (mScaleX != scaleX) {
            prepareForUpdate();   // step 1
            mScaleX = scaleX;
            invalidateAfterUpdate();
        }
	  }
    ```
    
    
    step 1处的调用关系比较多,我就只把关键的部分整理出来
    
    ```
     transformMatrix(m, view);
     mTempMatrix.mapRect(r);
    ```
    
    
    而transformMatrix是这样的
    
    ```
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
    
    ```
    这里面是把你可能设置的rotateX/Y/Z,tranlsteX/Y,以及scaleX/Y数据收集到matrix里面,一切都还与view无关,然后在setScaleY的最后一步
    
    ```
	    final RectF after = mAfter;
        computeRect(after, view);
        after.union(mBefore);
        ((View)view.getParent()).invalidate(
                (int) Math.floor(after.left),
                (int) Math.floor(after.top),
                (int) Math.ceil(after.right),
                (int) Math.ceil(after.bottom));
    ```
    
    这里computeRect里面其实还有调用一次transformMatrix,这里我还没看懂为什么还需要在调用一次,暂时忽略他,看最后一步invalidate,我们知道invalidate会同步或者异步导致view的重绘,直接导致的函数是View#draw
    在view.draw里面
	跟我们相关的是有三步的
	drawAnimation()
	__applyMatrix__
	onDraw()
	中间的applyMatrix没有独立成一个函数,我是为了方便理解取名的
	先看drawAnimation的调用,这里只整理关键代码
	
	```
	13095         if (a != null) {
    13096             more = drawAnimation(parent, drawingTime, a, scalingRequired);
    ```
    其实现的关键内容
    
    ```
    12962             invalidationTransform = parent.mInvalidationTransformation;
	12963             a.getTransformation(drawingTime, invalidationTransform, 1f);
	```
	
	看见没有,我们刚刚设置的animation(这里就是变量a)开始被调用了,getTransformation其实就是回调applyTransformation,这里需要注意的是我们回调时候设置的matrix内容被写到parent.mInvalidationTransformation这个变量里面去了,然后是__applyMatrix__关键代码
	
	```
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
    ```
   注意这里的transformToApply就是我们门刚刚的parent.mInvalidationTransformation,也就是我们的动画回掉applyTransformation修改的内容,于是一切都串起来了,我们在setScaleX里面先设置时长为0的动画,并且在动画回调里面返回了我们setScaleX应该做的矩阵变换,然后调用invalidate,经过一系列冗长的步骤之后在onDraw之前,draw函数顺利拿到这个matrix并且设置到canvas里面去,保证实现了最后的效果,为了确认,我们再看applyTransformation函数
   
   ```
    @Override
    protected void applyTransformation(float interpolatedTime, Transformation t) {
        View view = mView.get();
        if (view != null) {
            t.setAlpha(mAlpha);
            transformMatrix(t.getMatrix(), view);
        }
    }
    ```
   果然是吧我们做的变换扔到T里面去
   
   
   总结几点
   1. nineoldandroid没有什么特殊的能力做android原生做不到的东西,他能做的只是canvas能做的而已
   2. 对于alpha,不是做形变,而是直接设置到Transformation里面去,步骤其实跟上面类似
   3. 上面是针对android4.2的源码进行分析的,我调试时候发现不同版本的draw函数差距很大,其他版本估计具体流程不是这样,不过大致原理是相同的
   
   
   
   
   
   
   
   
   
   
   
   