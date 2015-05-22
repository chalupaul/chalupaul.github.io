---
layout: post
title: "Python Argparse"
date: 2013-07-02 14:53:49
published: true
category: tech
tags: [python]
---

Python's argparse has always felt a lot like java to me. Any time I'm making collecion objects, calling methods to make new objects to attach to said collection objects, and making yet another layer of collection objects on top of that, I think java.

Here's an example of a brief little mocked up cli tool for doing things to users (crud stuff). You can [download it][tarball] here and easy_install it.:

{% highlight python linenos=table %}

import sys
import argparse

class Foo(object):
    def init_parser(self):
        parser = argparse.ArgumentParser(
            prog='noose',
            description='hi, i am a description',
            epilog='hi there, i\'m an epilog'
        )
        parser_subs = parser.add_subparsers()
        parser_user = parser_subs.add_parser('user', help='manage users')
        parser_user_subs = parser_user.add_subparsers()

        parser_user_create = parser_user_subs.add_parser('create',
                                                         help='create user')
        parser_user_create.add_argument('--name', help='specify user name')
        parser_user_create.add_argument('--age', help='specify user age')
        parser_user_create.add_argument('-user_action',
                                      help=argparse.SUPPRESS,
                                      default='create')


        parser_user_delete = parser_user_subs.add_parser('delete',
                                                         help='delete user')
        parser_user_delete.add_argument('--name', help='name to delete')
        parser_user_delete.add_argument('-user_action',
                                      help=argparse.SUPPRESS,
                                      default='delete')

        parser_user_list = parser_user_subs.add_parser('list',
                                                       help='list users')
        parser_user_list.add_argument('-user_action',
                                      help=argparse.SUPPRESS,
                                      default='list')
        self.args = vars(parser.parse_args())

{% endhighlight %}

If you want to use it directly, you'll have to wrap it up into a cli or fudge it by passing an array to parse_args().

So a few tacky things:

1) There's a flow in here, but it's really hard to represent in code. It'd be super nice to be like "**mycommand user list** and **mycommand user create --name bob --age 42**" 

2) You can use the bulit-in callback stuff by adding set_default(func=yourfunction) to any parser, but just like any callback, it happens immediately when it's triggered. So in this design, having code leave __init__() to go chase down other logics I believe is a bad idea.

3) Because of #2, I'm overloading user_action so I can know the context of the request. Also tacky! In order to hide this variable, I'm suppressing it, but I am adding a "-" in front of it in order to make it not required (also tacky).

So how did this come up?

Well, I'm writing a generic cli to hook up to a service catalog for some engineer folk to use. But, if they change the API, they want it to be magically reflected in the CLI. Not a big deal really. But there are obviously some problems inasmuch as I'll have to change the assumptions that argparse is using. 

There's a fancy pants library out there which you should be using for cli options (for most cases). It's called [docopt][docopt]. I really like this approach (minus the template generation stuff). So my decision? Do the same thing with json.... I'll have something posted later on when I get a prototype. There is a problem of converting xml <-> json (api is in both), but we'll cross that bridge when we get there.



[tarball]: /images/posts/2013-07-02-Python-Argparse/foo.tar.gz
[docopt]: https://github.com/docopt/docopt