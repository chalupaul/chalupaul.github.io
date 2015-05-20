---
layout: post
title: "The Problem with Proxies"
date: 2014-07-16 01:57:17
categories: [tech]
tags: [infosec, ssh]
---

I gotta hand it to the infosec guys. It has got to be hard to feign interest in security as hard as they do--year after year writing papers about warnings that nobody cares about. But then something like heartbleed comes out and they're all, "I TOLD YOU SO!"

The world I live in now has no network-egress ports open but 80 and 443. My oh my how infuriating this is. I can't get my personal email, I can't watch some multimedia content (anything that streams over non 80 or 443), oh and most importantly: I can no longer ssh to anything.

We have bastion servers sure. And those are great when I am actually logging in to something myself. But who does that anymore? The problem: virtually no tools work well with ssh jump hosts. But luckily, this is 2014 and we have 30 solutions to this problem (most of them are depressingly inadequate for me to not miss my precious port 22).

First order of business is to check if my tooling can use an ssh proxycommand. Vagrant (the tool I'm using currently) sure can, but it is extremely buggy. I found out thru trial and error that when an instance becomes available for ssh login can be falsely triggered by the proxy host's key exchange. This is a loveley race condition. Not normally a problem in Vagrant, but I'm using the rackspace-vagrant driver because I have a 2nd gen macbook air and can't run vbox locally.

Next, set my osx system proxy to a socks5 proxy on my bastion server. Problem: Unless the tooling specifically looks for that, nothing changes (I'm looking at you, ssh).

Third, there's a [fancy hack][ssh_config_sed] that uses ssh config in a creative way. If you drop in a ```*.bast``` entry to your ssh config with the relevent passthru to a ProxyCommand. So you can have a web1 host that you connect to normally, and you ssh to web1.bast when you want to get to that host thru the bastion. 

There are 2 problems with this. 1) the %h as an argument from ProxyCommand is actually Hostname and not Host. This means that whatever gets sed'd out in the backticks has to be resolvable either by hostfile or dns. So sure this would work for ip addresses or dns names. But if I have chalupa1 and chalupa2 in my ssh config alreday, I'll just get nslookup failures. 

The second problem is that I am not always chalupaul on every box. Many boxes I'm just "paul" or "sims" or "psims" or whatever. The netcat hack in that file doesn't let you set a username for the target side (becaues netcat knows nothing of your silly username). The ```-W``` option to ssh is pretty nice, as it lets you pass in a remote host to connect stdin and stdout to, just like a built-in netcat mode. So if you know the user (again, no config available for this), and the hostname value, and port, you're good to go.

Here's a ruby 1-liner that assembles the necessary line for that (actually, you can just drop this in your ssh config file if you replace bastion\_server with your actual server.%h and %p come from ssh variables external to the shell command):

{% highlight bash %}
ProxyCommand `ruby -e "require 'net/ssh'; 
n=Net::SSH::Config.load('$HOME/.ssh/config', 
'%h'.sub(/\.bast$/, '')); 
print \" ssh -W #{n['user']}@#{n['hostname']}:%p bastion_server\""`
{% endhighlight %}

This totally would work... Except with ```-W```, you have to auth at every hop. That means you're either copying around your private key or agent forwarding (you shouldn't do that on a bastion server, probably never in fact).

So what's a boy left to do? With all the somewhat decent options not really working, I am left with good old proxychains. 

Proxychains works by basically hijacking the net libs and setting up routes custom to a specific program. That means you can use it as a launcher, and any application you launch with proxychains will use your socks proxy as if it was a regular old network. Pretty neat.

It can be installed by homebrew adn is simple to use:

{% highlight bash %}
$ proxychains vagrant up
{% endhighlight %}

My buddy wilk said it was best to just run it manually, but I'm on a mac and I want magic. So I tossed together a few scripts, and this solution I feel is pretty livable. Here's how it works:

1) Shared functions to check on the tunnel and bastion server health. Make sure you change bastion\_server to something real. Also, I like to source this file in my .bash_profile so I can troubleshoot network problems a bit better ($HOME/bin/proxyfuncs.sh):

{% highlight bash %}
bastion_server=YOUR.BASTION.SERVER:22

function is_bastion_available {
	[[ `curl --connect-timeout 1 $bastion_server 2>&1 | grep SSH | wc -l` -eq 1 ]] \
	&& echo "true" || echo "false"
}

function is_tunnel_established {
	[[ `lsof -Pni4 | grep 9050 | grep LISTEN | grep ssh | wc -l` -eq 1 ]] \
	&& echo "true" || echo "false"
}
{% endhighlight %}

2) A script sets up a socks proxy if the conditions are right ($HOME/bin/startproxy.sh):

 {% highlight bash %}
 #!/bin/bash -l
 source $HOME/bin/proxyfuncs.sh

 [[ $(is_bastion_available) == true && $(is_tunnel_established) == false ]] \
 && ssh -D 9050 bastion -N &
 {% endhighlight %}
 
3) A script to spawn a login shell if the proxy is running:
 {% highlight bash %}
 #!/bin/bash -l
 source $HOME/bin/proxyfuncs.sh

[[ $(is_tunnel_established) == true ]] \
&& exec proxychains4 -q bash -l -i || exec bash -l -i
{% endhighlight%}

4) A helper script to stop the proxy ($HOME/bin/stopproxy.sh):

 {% highlight bash %}
 #!/bin/bash -l
 source $HOME/bin/proxyfuncs.sh
 [[ $(is_tunnel_established) == true ]] \
 && kill `ps auxwww | grep ssh | grep $bastion_server | awk '{print $2}'`
 {% endhighlight %}

5) Finally, go download [controlplane][cp_link] and set it up to run that script when you're at a location that you need to be tunneled (you can do all sorts of things with that app). I just have it run the script every time it detects a location change (including on startup).

6) The last step is to set up iTerm2 to launch $HOME/bin/proxyshell.sh instead of a login shell, and you're good to go. If you don't want every process to be a child process of proxychains (this is very understandable if they have weird networking things that are lame like Preview.app and just won't run right), you can skip this step and just run proxyshell.sh in your current terminal to "turn it on". So the ssh tunnel will just sit there in the background.

Now every shell command you fire off will automagically use the tunnel if you're in a location where you can reach the bastion\_server. 



[cp_link]: http://www.controlplaneapp.com/
[ssh_config_sed]: https://journal.paul.querna.org/articles/2014/06/09/ssh-proxy-using-sed/