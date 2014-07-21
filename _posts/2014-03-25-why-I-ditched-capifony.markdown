---
layout: post
title: Why I Ditched Capifony As A Deployment Tool
keywords: symfony2 symfony capifony capistrano devops deployment php
date: 2014-06-24 00:07:57
---
In this post I will explain how I use a few tools to deploy applications as OS packages (in this example, RPMs).
I primarily deal with applications written in the <a href="http://www.symfony.com" target="_blank">Symfony2</a> PHP framework. One of the most popular tools for deployment
in the symfony2 community is <a href="http://capifony.org" target="_blank">Capifony</a>. I implemented capifony into our continuous integration workflow without any real problems. Everyone on the development team can deploy by simply issuing
a command (`cap production deploy`), and any customization can be done by writing some ruby. However, as our infrastructure grew, it became clear that something like capifony wasn't really the best choice for my use case.

The typical way people use capifony (and similar tools) is to run something like `cap production deploy` which will ssh into each server hardcoded into your `production.rb` file, run a bunch of commands, etc. In a dynamic environment, this doesn't work as
you need to figure out on the fly what the hostnames are for however many production servers happen to be up at the time of deployment. I could, of course, do this with some ruby (and I tested this out), but then there is still the issue that capifony
by default runs a number of commands on each server (composer install, assets install, assetic dump, etc etc). I wasn't happy with the number of points of failure in the process. I found coercing capifony into performing all of these tasks in a temporary location locally before copying the files to the deploy target servers was
a but too cumbersome for my taste.

My solution? Package-based deployments.

The idea is to hook into the continuous-integration process and, after a successful build, package up the code into an RPM and upload it to a private yum repository hosted on S3. Here's the tools I used
to accompilsh this:

- <a href="https://github.com/jordansissel/fpm" target="_blank">fpm</a> for building the RPM package (you can easily build deb packages with fpm too)
- <a href="https://github.com/jbraeuer/yum-s3-plugin" target="_blank">yum-s3-plugin</a> for authenticating with the private S3 bucket
- <a href="http://docs.aws.amazon.com/cli/latest/userguide/using-s3-commands.html" target="_blank">AWS CLI</a> for pushing up the RPM to the yum repository
- <a href="http://fabfile.org" target="_blank">Fabric</a> for executing commands (yum install my_app) on the servers

The whole process can be broken down into 3 steps: Package, Distribute, Deploy

### Step 1 - Package
First, I need to build the rpm package. I use <a href="http://jenkins-ci.org/" target="_blank">Jenkins CI</a> along with <a href="http://ant.apache.org/" target="_blank">Ant</a> to handle triggering steps of my builds
(code analysys, unit tests, deployments, etc), so I added the following target to my ant buildfile:

```xml
<!-- build.xml -->
<project>
<property environment="env"/>
<!-- ... -->
    <target name="fpm" description="Invokes fpm to build the RPM package">
        <exec executable="fpm">
            <arg value="-s"/>
            <arg value="dir"/>
            <arg value="-t"/>
            <arg value="rpm"/>
            <arg value="-n"/>
            <arg value="my.application.com"/>
            <arg value="--prefix=/var/www/my.application.com/releases/${env.BUILD_NUMBER}"/>
            <arg value="--directories=."/>
            <arg value="-x"/>
            <arg value="**/.git*"/>
            <arg value="-x"/>
            <arg value=".git*"/>
            <arg value="-x"/>
            <arg value="dist"/>
            <arg value="-x"/>
            <arg value="repo"/>
            <arg value="--iteration"/>
            <arg value="${env.BUILD_NUMBER}"/>
            <arg value="-p"/>
            <arg value="dist/${environment}/"/> <!-- The value of ${environment} should be one of "production", "staging", or "dev" -->
            <arg value="--after-install"/>
            <arg value="after_install.sh"/>
            <arg value="--rpm-user"/>
            <arg value="deployer"/>
            <arg value="--rpm-group"/>
            <arg value="deployer"/>
            <arg value="--template-scripts"/>
            <arg value="."/>
        </exec>
    </target>
</project>
```
The options passed to the `fpm` command set the install directory to `/var/www/my.application.com/releases/${env.BUILD_NUMBER}`. (`BUILD_NUMBER` is one of the environment variables
exposed by jenkins. I use this as a "release" number). See the <a href="https://github.com/jordansissel/fpm/wiki#usage" target="_blank">fpm documentation</a> for more info on the
options passed to `fpm`.

You'll notice that I pass the `--after-install` option to fpm which sets a script that will be run after the package installation.

Here's the `after_install.sh`:

```sh
#!/bin/bash -ex
DEPLOY_USER=deployer
RELEASE=<%= iteration %>

rm -rf /var/www/my.application.com/releases/$RELEASE/app/cache/* && /usr/bin/php /var/www/my.application.com/releases/$RELEASE/app/console cache:clear --env=prod
chown -R $DEPLOY_USER:$DEPLOY_USER /var/www/my.application.com/releases/$RELEASE/
test -d /dev/shm/my.application.com/app/cache/ && chown -R $DEPLOY_USER:$DEPLOY_USER /dev/shm/my.application.com/app/cache/
su -c "ln -nfs /var/www/my.application.com/releases/$RELEASE/ /var/www/my.application.com/current" $DEPLOY_USER && service php-fpm reload
```
The `after_install.sh` script should take care of any tasks that need to be run after the code is deployed. In this case, I need to warm the symfony2 cache and flip the `current` symlink.
A caveat I ran into is that symfony2's cache warm command generates absolute filenames in the cache files, which prevented me from running this on my jenkins box before packaging up the rpm.

If the fpm ant target executes successfully, it creates an rpm package in `PROJECT_ROOT/pkg`.


### Step 2 - Distribute

The next step is to distribute the package to the yum repository. Since I'm hosting the yum repository in a private S3 bucket, I need a way for yum to authenticate with S3. Meet
<a href="https://github.com/jbraeuer/yum-s3-plugin" target="_blank">yum-s3-plugin</a>. With this plugin, all you need to do is set up your yum repository per usual with the addition of a 
couple of keys:

```
[repoprivate-pool-noarch]
name=repoprivate-pool-noarch
baseurl=http://<YOURBUCKET>.s3-website-us-east-1.amazonaws.com/<YOURPATH>
gpgcheck=0
priority=1
s3_enabled=1
key_id=<YOURKEY>
secret_key=<YOURSECRET>
```

Note: you need to enable website feature on your s3 bucket. See the <a href="https://github.com/jbraeuer/yum-s3-plugin" target="_blank">docs</a> for more information.


### Step 3 - Deploy

Now that I have the application packaged up into an RPM and available in my yum repository, deployment is as simple as executing `yum install my.application.com` on each host.
I use <a href="http://fabfile.org" target="_blank">Fabric</a> for this because of it's simplicity. My fabfile looks something like this:

```python
from fabric.api import parallel, env, local, run
import yaml

yum_command = 'yum --disablerepo=* --enablerepo=myrepo -y'

def production():
    """ Setup hosts for the production environment """
    env.user = 'deployer'
    env.hosts = ['host1.prod', 'host2.prod', 'host3.prod', 'host4.prod']
    env.disable_known_hosts = True

def staging():
    """ Setup hosts for the staging environment """
    env.user = 'deployer'
    env.hosts = ['host1.staging']

def __expire_yum_cache():
    """ Expire yum cache """ 
    run('sudo %s clean expire-cache' % yum_command)

@parallel
def deploy():
    """ Deploy the latest rpm package version """
    __expire_yum_cache()
    run('sudo %s install my.application.com' % yum_command)

@parallel
def rollback():
    """ Deploy the previous rpm package version """
    run('sudo %s downgrade my.application.com' % yum_command)
```

Something to note here is the expire yum cache bit. Without this, new packages added to the yum repository won't be picked up, so `yum install my.application.com` will just say "nothing to do".
For performance's sake, I also tell yum to only look at my repository by passing the appropriate `--disablerepo` and `--enablerepo` options to yum.

I've been using this deployment method for a few months and haven't ran into any real issues with it. I will be migrating all of our applications to use this method in the coming weeks.

