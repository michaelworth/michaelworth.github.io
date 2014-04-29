---
layout: post
title: Animating a header out of a ListView
---

When we enabled Social Sign In for our mobile users in the HootSuite app, we wanted to keep the flow simple and light with as few screens and dialogs as possible. This meant that we needed to defer collecting the user’s email address in certain cases until after sign-up.

As such, we wanted to surface an inline notification asking users to enter their email. The design was pretty straightforward: put a dialog inline with the main tab view content where the user can insert the information or dismiss the dialog.

I’ve done this a few times before, and the cleanest way to implement this in my opinion is to create the inline element as a view and add it as a header to the `ListView` object. What I never really had to do before was dismiss it away with a collapsing animation.

The logic behind such an animation is not rocket science. First, you’ll want to fade the content out using an `AlphaAnimation`. Second, use a custom animation to apply a transformation on the view’s `LayoutParams.height` property. Finally, combine the two by using an `AnimationSet`, apply an appropriate duration and interpolator, and mark `fillAfter` as true to keep the final state after the animation completes. The resulting code is shown here:

{% highlight java %}
final int height = this.getMeasuredHeight();
final AbsListView.LayoutParams layoutParams = (AbsListView.LayoutParams) this.getLayoutParams();

AlphaAnimation alphaAnimation = new AlphaAnimation(1, 0);
alphaAnimation.setDuration(400);

Animation translationAnimation = new Animation() {

    @Override
    protected void applyTransformation(float interpolatedTime, Transformation t) {
        layoutParams.height = (int) (height * (1 - interpolatedTime));
        TwitterEmailView.this.setLayoutParams(layoutParams);
        Log.i(TAG, "applyTransformation: height = " + layoutParams.height);
    }

    @Override
    public boolean willChangeBounds() {
        return true;
    }
};

translationAnimation.setDuration(400);
translationAnimation.setStartOffset(300);

AnimationSet set = new AnimationSet(true);
set.setInterpolator(new AccelerateDecelerateInterpolator());
set.addAnimation(alphaAnimation);
set.addAnimation(translationAnimation);
set.setFillAfter(true);

set.setAnimationListener(new Animation.AnimationListener() {
    @Override
    public void onAnimationStart(Animation animation) {

    }

    @Override
    public void onAnimationEnd(Animation animation) {
        Log.i(TAG, "onAnimationEnd");
        ((ListView) TwitterEmailView.this.getParent()).removeHeaderView(TwitterEmailView.this);
    }

    @Override
    public void onAnimationRepeat(Animation animation) {

    }
});

this.startAnimation(set);
{% endhighlight %}

But when I ran it on my test device, the result was less than satisfactory:
<iframe width="420" height="315" src="http://www.youtube.com/embed/_ya63Fg6BkA" frameborder="0" allowfullscreen></iframe>

As you can see in the video above, the height property resets after the animation completes. I added some logcat messages and verified that the `LayoutParams.height` at the end of the animation was still 0. Adding a call to set the view as `GONE` had the same result.

My assumption here is that the `ListView`  reverts the header cell to it's original height, or to the height of a normal list item, due to the layout parameter not being preserved by the `fillAfter` setting. The best solution then appeared to be to remove the header from the `ListView` after the animation completes.

<iframe width="420" height="315" src="http://www.youtube.com/embed/sXSyHeH_txw" frameborder="0" allowfullscreen></iframe>

This basically works except that occasionally I can see a noticeable flicker after the animation finishes and before the view is removed. Here's the same video slowed down by 50% so YouTube doesn't filter out the frames showing the flicker.

<iframe width="420" height="315" src="http://www.youtube.com/embed/Use-4hvB1nw" frameborder="0" allowfullscreen></iframe>

I then decided to try re-writing the code using the `ValueAnimator` APIs that were made available after Honeycomb.

{% highlight java %}
final int height = this.getMeasuredHeight();
final AbsListView.LayoutParams layoutParams = (AbsListView.LayoutParams) this.getLayoutParams();

ObjectAnimator alphaAnimator = ObjectAnimator.ofFloat(this, "alpha", 0f).setDuration(300);

ValueAnimator heightAnimator = ValueAnimator.ofInt(height, 0).setDuration(400);
heightAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
    @Override
    public void onAnimationUpdate(ValueAnimator valueAnimator) {
        layoutParams.height = (Integer) valueAnimator.getAnimatedValue();
        TwitterEmailView.this.setLayoutParams(layoutParams);
    }
});

AnimatorSet set = new AnimatorSet();
set.playSequentially(alphaAnimator, heightAnimator);
set.setInterpolator(new AccelerateDecelerateInterpolator());
set.addListener(new AnimatorListenerAdapter() {
    @Override
    public void onAnimationEnd(Animator animation) {
        ((ListView) TwitterEmailView.this.getParent()).removeHeaderView(TwitterEmailView.this);
    }
});

set.start();
{% endhighlight %}

Essentially the same thing, just cleaner and using modern APIs. The interesting bit is that the resulting animation doesn’t have the flicker anymore. The other good news is that this code can be used on Gingerbread as well by using the <a title="NineOldAndroids" href="https://github.com/JakeWharton/NineOldAndroids/" target="_blank">NineOldAndroids</a> library.

<iframe width="420" height="315" src="http://www.youtube.com/embed/LUJFIPK_HLs" frameborder="0" allowfullscreen></iframe>

To try and understand the difference between the two variants better, I built out a new app to host a simple `ListView` and apply the same animation source to it and see what the results were. 

Using this new codebase, I still saw the flicker of the old pre-Honeycomb animation technique as I did in the HootSuite application which proved to me that it wasn't occurring due to some other layout issue. As expected, the Honeycomb variant worked as it did before with no flicker.

I theorized that maybe there was a delay in the call to `onAnimationEnd` for the pre-Honeycomb method so I added in some logging to measure the time delta of both methods from the last update/transformation call to the `onAnimationEnd` call. The timing was rather variable but the V1 method ranged from 20ms-63ms and V2 ranged from 50ms-160ms. Clearly, the delay wasn't involved in that way.

I then created a class that extended `FrameLayout` so that I could put in logging in the `onLayout()` method to see if that might shed some light on the situation. The V1 method showed the layout bottom progressing towards 0 but jumps up suddenly when the flicker occurs. This layout update comes *before* the final `onAnimationEnd` callback is called. V2 doesn't show this and instead stays at zero and then calls the callback.

At this point, I reached out to some trusted friends in the Android community for insight and got a huge tip from my pal Ryan Wheedon at AutoTrader. He had run into this in the past and pointed out that `ListView` has a bug where it handles list children with a height of 0 as `MeasureSpec.UNSPECIFIED` which in effect causes the view to measure as `WRAP_CONTENT`. 

I verified this by first adding more logging to my extended `FrameLayout` which indeed showed that the `MeasureSpec` for the `FrameLayout` was changing from `MeasureSpec.EXACTLY` to `MeasureSpec.UNSPECIFIED` when the flicker occurred. This resulted in the `onMeasure` method measuring how much height is required by the view rather than just setting the height to 0 as requested. 

So why does this happen? Well take a look at the following code taken from `setupChild(...)` from `ListView`:

{% highlight java %}
if (needToMeasure) {
    int childWidthSpec = ViewGroup.getChildMeasureSpec(mWidthMeasureSpec,
            mListPadding.left + mListPadding.right, p.width);
    int lpHeight = p.height;
    int childHeightSpec;
    if (lpHeight > 0) {
        childHeightSpec = MeasureSpec.makeMeasureSpec(lpHeight, MeasureSpec.EXACTLY);
    } else {
        childHeightSpec = MeasureSpec.makeMeasureSpec(0, MeasureSpec.UNSPECIFIED);
    }
    child.measure(childWidthSpec, childHeightSpec);
} else {
    cleanupLayoutState(child);
}
{% endhighlight %}

Therefore, if the height specified in the `measureSpec` is 0, it forces the new `measureSpec` to be `MeasureSpec.UNSPECIFIED`. Not exactly desired behaviour for this animation.

So how to fix V1? I was able to get away with overriding `onMeasure` to first check if the height `measureSpec` is `MeasureSpec.UNSPECIFIED` and simply force it back to `MeasureSpec.EXACTLY`. This might not be a good solution for all situations since it prevents the use of `WRAP_CONTENT` in the layout XML. 

The final mystery is why this doesn't happen for V2 if the problem is due to `ListView`? Ryan also pointed out that header height change calculation was slightly different between the two versions. This caused V1 to reach 0 before the final animation cycle and then measure itself with the incorrect `MeasureSpec` just before before being removed. V2 set the view height to 0 in the final cycle and removed it before it can measure itself again. If the math in V2 were changed to match exactly it would show the same flicker as V1.

I’ve posted the source of the demo application to GitHub for people to take a look at and play with.
[https://github.com/michaelworth/HeaderAnimationDemo](https://github.com/michaelworth/HeaderAnimationDemo)