---
layout: post
title:  "Some words about patches"
date:   2015-09-30 12:06:31
tags: git svn patches
published: false
---

There are a lot of old systems that not use Control Version System even in 2015. Or you have no access to repository where you need to edit something. And here is time for patches.

Even if you are using Git and SVN you need to prepare patch to send it repository owner.

# Situation 1
You have your Git repository and you need to prepare patch for antoher Git repository. And Git here is really awesome. [git-format-patch](http://git-scm.com/docs/git-format-patch) is what you need.

{% highlight bash %}
git format-patch HEAD~5..HEAD
{% endhighlight %}

It will generate patch file for each commit (5 in this case). If you have separate branch for your commits you can use

{% highlight bash %}
git format-patch master --stdout > patch-name.patch
{% endhighlight %}

In this variant you will have only one file. In both cases then you need just to apply each patch with [git-am](http://git-scm.com/docs/git-am)

{% highlight bash %}
git am /path/to/patch/file.patch
{% endhighlight %}

but if you have a problems with this or (as in my case) your repositories not sync [git-apply](http://git-scm.com/docs/git-apply) with right arguments can help you

{% highlight bash %}
git apply --reject --ignore-space-change --ignore-whitespace /path/to/patch/file.patch
{% endhighlight %}

It will apply patch and generate `*.rej` files in case of something can't be merged automaticly.

# Situation 2

You have SVN. Mm, not so good. But let's try. To prepare patch you need to use [svn-diff](http://svnbook.red-bean.com/en/1.7/svn.ref.svn.c.diff.html)

{% highlight bash %}
svn diff --revision REVISION_NUMBER > /path/to/patch/file.patch
{% endhighlight %}

and apply patch

{% highlight bash %}
patch -p0 -i /path/to/patch/file.patch
{% endhighlight %}

But be careful, please, it's SVN and it not forgive mistakes.

# Situation 3

You have Git repository and you need to prepare patch for SVN. Welcome to the hell! You can use [git-svn](http://git-scm.com/docs/git-svn). It seems like perfect variant. You can work with your SVN repository as with Git repository. So, prepare patch in Git repository, apply it with `git-apply` or `git-am` and commit with `git svn dcommit`. There is no any universal variants.
