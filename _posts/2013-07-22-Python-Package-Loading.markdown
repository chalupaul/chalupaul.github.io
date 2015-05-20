---
layout: post
title: "Python Package Loading"
date: 2013-07-22 02:11:07
categories: [tech]
tags: [python]
---

One of the things I find quite frustrating about python is the lack of decent module/package loading. To me, it feels like it started being almost strictly filesystem based, and then a ton of hacks were shoved in it to make it do other things. Hence, \_\_all\_\_ came to be. I mean, most of \_\_init\_\_.py is for weird reasons that other languages don't have.

So here's a very typical example. Let's say you have a python package named "pypackage." In it, you have your files broken up by some arbitrary reason that makes your code more legible. So you basically have this:

{% highlight console %}
$ ls -l pypackage/
total 24
-rw-r--r--  1 chalupaul  staff    0 Jul 22 02:17 __init__.py
-rw-r--r--  1 chalupaul  staff  103 Jul 22 02:18 __init__.pyc
-rw-r--r--  1 chalupaul  staff   28 Jul 22 02:18 one.py
-rw-r--r--  1 chalupaul  staff   29 Jul 22 02:18 two.py
$
{% endhighlight %}

These classes are pretty simple:

{% highlight console %}
$ cat pypackage/one.py
class One(object):
    pass
$
{% endhighlight %}

So, let's say for example that we want to load these classes. We have a few options, tho none of them really work. You can't "from pypackage import \*" without cluttering up your namespace. Also, you have to wire up \_\_all\_\_ or you get nothing. You can import them by name sure (like, "import pypackage.one, pypackage.two" etc). 

But what if you don't know what module you are looking for? For example, I recently wrote some code that was aggregating a lot of API services into one interface. So I wrote a bunch of filter classes and dropped them into a package. They have a naming convention: _parent.filters.XxxFilter_. So there are user filters that show how to interpret the User API, Auth filters to consume an Auth endpoint, etc. 

The standards between these APIs aren't exactly similar enough to be able to follow one set of filters, so I wanted to write the specific filters with 3 methods: One method to load the endpoints of a particular service, one method to generate the proper url to reach an endpoint, and one method to interpret API details of an endpoint. It's totally OK to shove all sorts of custom code into here as long as they return a dict in the right format: {noun: {verb: {option: flags: flag_values}}}.

There is no built in way with python to do these imports. What I wanted is to basically do "import filters" and it would import filters.UserFilter, filters.XxxFilter, etc. I can do "from filters import \*", but then it merges all that crap into my current namespace (I don't feel like it's appropriate to have classes like "User" floating around when they should really be namespace'd at "filters.User"). So other than the real answer (use ruby lol), you can put things like this in a package's \_\_init\_\_.py:

{% highlight python linenos=table %}
__all__ = ["one", "two"]
[__import__('.'.join([__name__, x])) for x in __all__]
{% endhighlight %}

Before, you had this:

{% highlight console %}
>>> import pypackage
>>> pypackage.one.One()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'module' object has no attribute 'one'
>>>
{% endhighlight %}

Now you can do this:

{% highlight console %}
>>> import pypackage
>>> pypackage.one.One()
<pypackage.one.One object at 0x108652d50>
>>>
{% endhighlight %}

As a side effect, "from pypackage import \*" will now work as well because you populated \_\_all\_\_.

So that's all well and good. Now you can load a package and it will auto-load anything in your \_\_all\_\_. Probably the best hack possible for a crummy situation. But there's one thing missing. You need to be able to do something with these modules. Let's add a method that prints out a random URL for you to do something with to each class. For this example, we can put the same method in both One() and Two():

pypackage/one.py:
{% highlight python linenos=table %}
class One(object):
    @staticmethod
    def url():
        return "http://localhost:3000/api/v1.0/{}".format(__name__)
{% endhighlight %}

pypackage/two.py:
{% highlight python linenos=table %}
class Two(object):
    @staticmethod
    def url():
        return "http://localhost:3000/api/v2.0/{}".format(__name__)
{% endhighlight %}

Now we have 2 classes that implement a url() method that will print out a different url (both in name and in the url (v1.0 vs v2.0). Let's say we have some other code that can take a url and go fetch that api. Pretty fun. How do we collect all the URLs for our different classes (in this case One() and Two()) _without_ explicitely knowing the classes available to us (in case someone adds a Three() or Four() or FortyThousandAndTwelve())?

The most "pythonic" way to do this is something I am actually not fond of doing: putting more things into pypackage/\_\_init\_\_.py:

{% highlight python linenos=table %}
__all__ = ["one", "two"]
[__import__('.'.join([__name__, x])) for x in __all__]

def urls():
    return [getattr(globals()[x], x.capitalize()).url()
            for x in __all__]
{% endhighlight %}

So globals() is ironically used in this place, because it is "global" in the scope of the _calling module_, so it's not really global haha. But it's a dict of the symbol table of the current module. I don't particularly like this becaues I kind of think \_\_init\_\_.py is an abomination, and a certainly a questionable place to put code. I suppose that's just my opinion.

Anyway, now you can do this:

{% highlight console %}
>>> import pypackage
>>> pypackage.urls()
['http://localhost/api/v1.0/pypackage.one', 'http://localhost/api/v2.0/pypackage.two']
>>>
{% endhighlight %}

And that is called "making the best of a poorly designed language" haha.

Oh one more thing. As you inevitably have to hack more and more stuff like this into your growing codebase, you're going to feel the need to pull that out of \_\_init\_\_.py into something a little more dev friendly. There's probably a few ways to skin that cat though. We're going to create a helper file that gets loaded int othe module directly to provide urls(), but won't be callable directly (often desired for stuff that just init things). You have to refactor the urls() function to be external to pypackage directly:

pypackage/__bootstrap__.py:
{% highlight python linenos=table %}
import pypackage
def urls():
    return [getattr(
                getattr(pypackage, x),
                x.capitalize()
            ).url()
            for x in pypackage.__all__]
{% endhighlight %}

and the final pypackage/__init__.py:
{% highlight python linenos=table %}
__all__ = ["one", "two"]
[__import__('.'.join([__name__, x])) for x in __all__]

from __bootstrap__ import *
del __bootstrap__
{% endhighlight %}

So here's the finished results. The functions were imported, and there's no \_\_bootstrap\_\_ in there to confuse users. I added some pprint stuff in here so it would display nicely.:

{% highlight console %}
>>> from pprint import pprint
>>> import pypackage
>>> pprint.(dir(pypackage))
['__all__',
 '__builtins__',
 '__doc__',
 '__file__',
 '__name__',
 '__package__',
 '__path__',
 'one',
 'pypackage',
 'two',
 'urls',
 'x']
>>> pypackage.urls()
['http://localhost/api/v1.0/pypackage.one', 'http://localhost/api/v2.0/pypackage.two']
>>>
{% endhighlight %}

Oops, it's 430am. Time to crash before work in.... 3.5 hours. :( :( :(