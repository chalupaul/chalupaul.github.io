---
layout: post
title: "Silly Api Designs"
date: 2013-08-01 15:31:26
categories: [tech]
tags: [python, api, json]
---

In my last post, I mentioned that this api flattens elements, and it has been causing me no end of problems (besides having to write specialized recursive functions to handle them instead of simple for loops).

So the problem with this api is that there is no difference in this schema:

{% highlight console %}
{
    "party[location]": [
        {
            "name": "funky",
            "required": true,
            "type": "boolean"
        },
        {
            "name": "name",
            "required": true,
            "type": "string"
        }
    ]
}
{% endhighlight %}


between this:

{% highlight console %}
{
    "party": {
        "location": [
            {
                "funky": false,
                "name": "fancy_place"
            },
            {
                "funky": true,
                "name": "basement"
            }
        ]
    }
}
{% endhighlight %}

and this:

{% highlight console %}
{
    "party": {
        "location": {
            "funky": true,
            "name": "rich_guys_house"
        }
    }
}
{% endhighlight %}

While it might not seem like much, not knowing whether or not something is an object or a collection of objects is REALLY important. I tried using [inflect][inflect] for it, but then ran into this intersting little problem.

What do you do when a param is called "shots" since it *could* be either plural or singular.

- You could have shots be a list of "shot" objects, each with properties.
- You could just want to give common properties for all shots

Sure you can inject all sorts of supposed behavior like "if any of the properties are "name" or "id" then you have a list. But that's exceedingly fragile. What about if it gets changed to "_id" or something like that? Then you're boned.

There isn't a real good solution to this problem witout fixing the API to indicate if things are arrays or not. I would ask them to switch to jsonschema, but then I'd have to rewrite all that dict generating code and it's some heady stuff :P Recursive algorithms take like, 2 hours to write 10 lines of code.

So I'm just going to ask the devs to turn that original schema into this:

{% highlight console %}
{
    "party[location]": [
        {
            "funky": {
                "required": true,
                "type": "string"
            },
            "name": {
                "required": true,
                "type": "boolean"
            }
        }
    ]
}
{% endhighlight %}

And that will be super clear to use without ambiguity.


[inflect]: https://pypi.python.org/pypi/inflect
