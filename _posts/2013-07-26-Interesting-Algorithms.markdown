---
layout: post
title: "Interesting Algorithms"
date: 2013-07-26 03:27:21
published: true
categories: [tech]
tags: [python]
---

So in writing my argparse api client, I ended up with a few interesting algorithms for matching dictionary representations of json schema that can be used for other things. I replaced a lot of the data and give-aways with kesha lyrics, so please enjoy ;)


The problem: The api definition flattens elements.

So instead of seeing: 

{% highlight console %}
{
    "party": {
        "dance": {
            "funky": {
                "required": true,
                "type": "int"
            }
        }
    }
}
{% endhighlight %}

you see:

{% highlight console %}
{
    "party[location][name]": {
        "required": true,
        "type": "string"
    },
    "party[location][funky]": {
        "required": true,
        "type": "boolean"
    }
}
{% endhighlight %}

Now imagine there's N number of those. Job one is unifying all those a\[b\[c\[\]\]\] strings into one huge dictionary. We're iterating over a fat list of these things, so it makes sense to make a tiny dictionary and merge it in with the mothership dict. 

{% highlight python linenos=table %}
'''Turn an array into a nested dict. So [1,2,3] -> {3:{2:{1:{}}}}.
So yeah, you probably want to [::-1] that list. 
If you send it a 3rd dict (z), set the innermost value to it.'''
def dance(x={},y=[],z={}):
    if y == []:
        return x
    else:
        glitter = {}
        if x == {}:
            glitter[y[0]] = z.copy()
        else:
            glitter[y[0]] = x.copy()
        return dance(glitter, y[1::])
{% endhighlight %}

Now that we can generate little dicts of these elements, we need to unite them. This is a somewhat challenging problem because you can potentially run into conflicting values when you merge 2 dicts with the same named properties. In the case of an api json schema though, you actually don't have to worry about that though.

{% highlight python linenos=table %}
'''merge two dictionaries. So {'a': {'b': {'c': {}}}} + {'a':{'b':{'x':{}}}} =
{'a': {'b': {'c': {}, 'x':{}}}}'''
def go_hard(x,y):
    who_we_are = dict(x,**y)
    for key in x.keys():
        if type(x[key]) is types.DictType and y.has_key(key):
            who_we_are[key] = go_hard(x[key],y[key])
    return who_we_are
{% endhighlight %}

Now you can do cool things like this:

{% highlight python linenos=table %}
'''Simulate the api snippet.'''
import types
from pprint import pprint
animals = [{u'required': True, u'type': u'string', u'name': u'party[drinks][whiskey]'},
{u'required': True, u'type': u'string', u'name': u'party[drinks][wine_cooler]'},
{u'required': True, u'type': u'string', u'name': u'party[lights][strobe]'},
{u'required': True, u'type': u'string', u'name': u'party[basement][girls_is]'}]


cannibal = {}
for i in animals:
    diamonds = [x.rstrip(']') for x in i['name'].split('[')]
    sleazy = dance({}, diamonds[::-1], {"dj": "turn it up"})
    cannibal = go_hard(cannibal, sleazy)

pprint(cannibal)
{% endhighlight %}

And the results....

{% highlight console %}
{u'party': {u'basement': {u'girls_is': {'dj': 'turn it up'}},
            u'drinks': {u'miller_lite': {'dj': 'turn it up'},
                        u'whiskey': {'dj': 'turn it up'}},
            u'lights': {u'strobe': {'dj': 'turn it up'}}}}
{% endhighlight %}