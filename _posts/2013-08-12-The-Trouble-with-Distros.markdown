---
layout: post
title: "The Trouble with Distros"
date: 2013-08-12 15:07:14
published: true
categories: [tech]
tags: [openstack, python]
---

OpenStack is a config management nightmare. It's a beast of a system, with hundreds of options for each project. It is unmanagable without Puppet or Chef (I haven't used any of the 3rd wave config management stuff yet). There's quite a few (though I'll stick with 3 for the sake of this: RPC from Rackspace, RDO from RedHat, and Fuel from Mirantis), but by definiton, pre-boxed config managed environments suffer from one huge probelm:

You cannot enable an option they don't have templated.

I'll give you an example. It is not possible to put auto_assign_floating_ips=true in /etc/nova/nova.conf unless they explicitely put that option in there for you to enable. But because there's probably 2000+ different flags spanning the different OpenStack projects, it's not realistic for them to put in a wrapper for every option. So you have two options:

Either turn off chef/puppet and hate yourself it 2 months, or you submit a feature request and pray that someone gives you the time of day. Or you could learn their plumbing and submit a patch, but that's an aweful lot of work for what would take them 2 minutes to do and no guarantee they want it.

None of those are good options for most people haha. As it turns out, there's this fancy little config helper in OpenStack called [Oslo][oslo] that should help you out! Oslo is responsible for digging around in a few directories and finding folder locations for config files and merging them into a big fatty config object for your OpenStack service. Let's take a look at the code:

from oslo.config/oslo/config/cfg.py:
{% highlight python linenos=table %}
def _get_config_dirs(project=None):
    """Return a list of directors where config files may be located.

    :param project: an optional project name

    If a project is specified, following directories are returned::

      ~/.${project}/
      ~/
      /etc/${project}/
      /etc/

    Otherwise, these directories::

      ~/
      /etc/
    """
    cfg_dirs = [
        _fixpath(os.path.join('~', '.' + project)) if project else None,
        _fixpath('~'),
        os.path.join('/etc', project) if project else None,
        '/etc'
    ]

    return list(moves.filter(bool, cfg_dirs))


def _search_dirs(dirs, basename, extension=""):
    """Search a list of directories for a given filename.

    Iterator over the supplied directories, returning the first file
    found with the supplied name and extension.

    :param dirs: a list of directories
    :param basename: the filename, e.g. 'glance-api'
    :param extension: the file extension, e.g. '.conf'
    :returns: the path to a matching file, or None
    """
    for d in dirs:
        path = os.path.join(d, '%s%s' % (basename, extension))
        if os.path.exists(path):
            return path


def find_config_files(project=None, prog=None, extension='.conf'):
    """Return a list of default configuration files.

    :param project: an optional project name
    :param prog: the program name, defaulting to the basename of sys.argv[0]
    :param extension: the type of the config file

    We default to two config files: [${project}.conf, ${prog}.conf]

    And we look for those config files in the following directories::

      ~/.${project}/
      ~/
      /etc/${project}/
      /etc/

    We return an absolute path for (at most) one of each the default config
    files, for the topmost directory it exists in.

    For example, if project=foo, prog=bar and /etc/foo/foo.conf, /etc/bar.conf
    and ~/.foo/bar.conf all exist, then we return ['/etc/foo/foo.conf',
    '~/.foo/bar.conf']

    If no project name is supplied, we only look for ${prog.conf}.
    """
    if prog is None:
        prog = os.path.basename(sys.argv[0])

    cfg_dirs = _get_config_dirs(project)

    config_files = []
    if project:
        config_files.append(_search_dirs(cfg_dirs, project, extension))
    config_files.append(_search_dirs(cfg_dirs, prog, extension))

    return list(moves.filter(bool, config_files))
{% endhighlight %}

Sorry for the huge codeblock :/ 

Anyway, according to this, you should be able to put options in /etc/nova/service.conf where service is something cool like nova-network.conf. So let's try it out!

Create a file /etc/nova/nova-network.conf on all your compute hosts and add these lines to it:

{% highlight console %}
[DEFAULT]
auto_assign_floating_ip=true
{% endhighlight %}

Now bounce the nova network service and boot an instance.

Go ahead, I'll wait.

...

Oh it didn't work? That's odd....

Right. Both Ubuntu and RedHat effectively disable this feature by explicitely setting all the config files on the commandline in their init scripts. I happen to have a centos box handy....

{% highlight console %}
$ grep daemon /etc/init.d/openstack-nova-network
    daemon --user nova --pidfile $pidfile "$exec --config-file $config \
    --logfile $logfile &>/dev/null & echo \$! > $pidfile"
$
{% endhighlight %}

So just remove that --config-file $config part, as it isn't needed (thanks to Oslo). In Ubuntu, you'll want to make the necessary changes to /etc/init/nova-network. 

I personally think it's pretty tragic that the distros aren't getting rid of the flag. Ironically, Ubuntu actually creates a nova-compute.conf and puts stuff in there by default, but flat out prevents the use of the same files for other programs. So in order to get around perscriptive config managers and assertive ditro packagers, it looks like you are forced to keep your init-scripts manually patched for the time being, which is quite unfortunate. 



[oslo]: https://wiki.openstack.org/wiki/Oslo
