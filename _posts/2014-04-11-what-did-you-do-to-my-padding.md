---
layout: post
title: Hey setBackgroundResource(), what did you do to my padding?
---

While playing around with a pretty simple list view item layout, I had just perfected my margins and positioning for all the components and it was time to update the background with a custom nine patch image. For implementation specific reasons, I had to do this in code and after pushing the APK to the device I noticed that all my padding was gone.

What gives?

Well, since the nine patches define their own padding, Android correctly stomped out my XML padding and used the nine patchs' own settings. In this case, I didn't want to create a custom nine patch just for this one view so I simply stored the padding values before setting the drawable and reapplied them afterwards.

{% highlight java %}
private void setBackgroundResourceAndKeepPadding(View view, int resourceId) {
    int left = view.getPaddingLeft();
    int right = view.getPaddingRight();
    int top = view.getPaddingTop();
    int bottom = view.getPaddingBottom();

    view.setBackgroundResource(resourceId);
    view.setPadding(left, top, right, bottom);
}
{% endhighlight %}