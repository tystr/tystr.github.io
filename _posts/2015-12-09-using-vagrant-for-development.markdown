---
layout: post
title: Using Vagrant for Development
summary: Vagrant facilitates a consistent, reproducible, and simple workflow for managing development environments.
keywords: Vagrant
date: 2015-12-08
class: post-body
---
## Why Use Vagrant?

Before I implemented Vagrant development environments for all our projects, it
was a nightmare getting everyone set up with a new project. I was frequently
dealing with "bugs" that ultimately were caused by some weird configuration on
developers' workstations. Developing against a virtual machine solves these
(and other!) problems. Here are some of the reasons I use Vagrant:

1. Consistency

    I could not tell you how many times I've heard "it worked for me locally", or
    "it works on my machine" when inevitably someone had different version of
    some library installed, or were using some different server than production
    uses. Having individual Vagrant environments for each project ensures that
    every engineer on the project is developing on an environment that is identical
    to production.

2. Reproducibility
    
    Not only is a Vagrant environment consistent with production, but you can
    reproduce it on any workstation or laptop as long as you have Vagrant and
    Virtualbox installed. Fiddled with the webserver config on the VM and now
    it's broken? No problem, just blow it away and bring up a fresh VM. Had to
    borrow someone's computer for the day? Just grab Vagrant and Virtualbox and
    you're set.

3. Simplicity

    It's so much simpler to bring up a Vagrant box instead of having to install
    and maintain a bunch of webserver and database dependencies on one's
    workstation. Developers should not have to spend time maintaining their
    development environments. A concise `vagrant up` command is usually all
    that's needed to get the environment up and running.


I won't go into too much detail about how to set up a fresh VM from scratch;
this isn't meant to be a comprehensive tutorial on what you can do with
Vagrant. However, there are a few practices that I employ that really help
keep the developers' process as streamlined as possible:

## Use a Configuration Management Tool

I highly recommend handling any provisioning tasks (installing PHP, configuring Nginx, etc. by using a configuration
management tool. Many are supported by Vagrant as you can see from their
<a href="https://docs.Vagrantup.com/v2/provisioning/index.html" target="_blank">
provisioning documentation</a>. In a perfect world, this would be the same tool
used (same code/scripts too) to provision the production infrastructure, but that may not always be the
case. I could write a series of posts on how to choose a particular tool, but at the end of the day it doesn't really matter, as long as you and your
team are familiar with it. One of the benefits here is that your configuration
is documented because it is in your version control.

## Use a YML File to Configure Common Settings
Use a `yml` file to configure parameters such as ip address, memory, cpus, etc.
This allows developers to customize things to better suite their host machine,
for example if the host does not have a lot of RAM, or they're running a bunch
of other things simultaneously and want to limit the resources allocated to the
VM.

I typically use a file that looks something like this:

{% highlight yaml %}
# Vagrantparams.yml.dist

hostname: myproject.dev
ip: 192.168.10.10
nfs: true
memory: 2048
cpus: 4
{% endhighlight %}

This file should contain some sane default values and be committed to your VCS.
When cloning the project, one should copy this file to `Vagrantparams.yml` so it
can be read from the `Vagrantfile`:

{% highlight ruby %}
# Vagrantfile

if (!File.exist?('Vagrantparams.yml'))
    puts "\nCould not find the file \"Vagrantparams.yml\". Did you forget to copy it from \"Vagrantparams.yml.dist\"?\n\n"
    exit
end

require "yaml"
params = YAML::load_file("./Vagrantparams.yml")

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "hashicorp/precise32"
  config.vm.hostname = params["hostname"]
  config.vm.network :private_network, ip: params["ip"]

  # Ensure nfs premissions are set correctly
  if (/darwin/ =~ RUBY_PLATFORM) != nil
      config.vm.synced_folder ".", "/Vagrant", :nfs => params["nfs"], :bsd__nfs_options => ["-maproot=0:0"], :map_uid => 0, :map_gid => 0
  else
      config.vm.synced_folder ".", "/Vagrant", :nfs => params["nfs"], :linux__nfs_options => ["no_root_squash"], :map_uid => 0, :map_gid => 0
  end

  config.vm.provider :virtualbox do |vb|
    vb.customize ["modifyvm", :id, "--memory", params["memory"]]
    vb.customize ["modifyvm", :id, "--cpus", params["cpus"]]
  end

  # ... more provisioning stuff ...

end
{% endhighlight %}
Notice how I've required "yaml", read the `Vagrantparams.yml` file into the `params` variable,
and referenced that instead of hard coding the values.

You'll also want to add `Vagrantparams.yml` to your project's `.gitignore` file so no one
accidentally commits their local parameters file.

## Setup SSH Forwarding 

If you are not working with dependencies that live in private git repositories,
then you can skip this bit. If you _are_, then you will likely need to be
able to clone these from within the Vagrant VM. To do this without hardcoding ssh
keys on the VM (don't do that), you need to first set up ssh forwarding on the
_host_ machine as described in this
<a href="https://developer.github.com/guides/using-ssh-agent-forwarding" target="_blank">Github article</a>.
Essentially all you should need to do is add the following snippet to your `~/.ssh/config`
file on the host machine:

{% highlight bash %}
Host myproject.dev
  ForwardAgent yes
{% endhighlight %}

Note that this `Host` value must match the hostname you've given your VM.

Then you'll need to enable ssh forwarding in your `Vagrantfile`:
{% highlight ruby %}
# Vagrantfile

# ...

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
    config.ssh.forward_agent = true
end

# ...

{% endhighlight %}

Now you should be able to clone repositories using the ssh key forwarded from 
the host machine.

## Document All The Things

Document everything. The barrier for getting a development environment up and
running should be as minimal as possible. The workflow for interacting
with the environment should be as streamlined as possible. For example, I typically
instruct engineers in the README (you do have a README, right?) to add an entry to
`/etc/hosts` on the host machine containing the ip and hostname of the VM, so the
project is accessible via `http://myproject.dev`. A small but easy thing to do
to improve the developer experience. Is ssh forwarding needed?
Explain so in the README.

The README file of your project should contain clear, step-by-step instructions
on how to get the Vagrant environment up and running, and point out any possible
caveats.

## Profit!

Following these guidelines has drastically cut down on the time and frustration
of both bring new engineers onto a project and maintaining a working development
environment.

