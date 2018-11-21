---
layout: post
title: Autoscaling, Hostnames, and Puppet
summary: Automatically set hostnames and CNAME records for instances as they boot up. Connect/remove instances from the puppetmaster automatically on boot/shutdown.
keywords: autoscaling puppet route53 aws ec2
date: 2014-07-21 8:00:28
class: post-body
---
One of the advantages of using a cloud provider such as Amazon Web Services is the flexibility to quickly terminate and create new instances. You can even do this automatically with autoscaling.
But then you are faced with the problem of provisioning; do you create a base <a href="http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html" target="_blank">AMI</a> for each server's "role"?
This is one option but could easily turn into dozens of AMIs to keep track of, not to mention
there's no way to really keep things up-to-date, roll out changes across the infrastructure, etc. Configuration management is the obvious solution here, and <a href="http://puppetlabs.com/" target="_blank">Puppet</a>
is a popular choice. One "issue" with AWS's EC2 instances is the machine's hostname will be something like `ip-10-0-0-113` while the public hostname would be somthing like `ec2-75-101-128-23.compute-1.amazonaws.com`. This is rather
inconvenient.

I'm going to explain how you can automatically set sane hostnames and CNAME DNS records for instances as they boot up and connect/remove them from the puppetmaster automatically on boot/shutdown.

In my infrastructure, I identify a server's "role" by a file, `/etc/ROLE`. This would contain something like `someapp_webserver` which identifies the application to which the server belongs and it's
role, in this case "webserver". Since I wanted a machine's hostname to identify what it does and what application it's a part of,  I decided to format the hostnames as follows:

    someapp-somerole-xxx.environment.mydomain.net

where `xxx` is the lowest available number beginning with 000 and `environment` is something like `prod` or `staging` (I use the environment variable from puppet to set this). So, for example, if a server's
role (content of /etc/ROLE) is `myawesomeapp_webserver` and there are already the hostnames `myawesomeapp-webserver-000.prod.mydomain.net` and `myawesomeapp-webserver-001.prod.mydomain.net`, then the next
availble hostname would be `myawesomeapp-webserver-002.prod.mydomain.net`. To accomplish setting this automatically, we need some bash script magic to run at boot which queries Route53, figures out which
hostname to use, sets it on the server and creates the CNAME record. Here's what my bash script looks like:
{% highlight bash %}
#!/bin/bash
#
# /usr/sbin/onboot.sh
#
# This script figures out a hostname based on the value in the role file.
# E.g, "someapp_webserver" would result in the hostname someapp-webserver-xxx
# where "xxx" is the next available number. The hostname is set locally and
# a CNAME record in Route53 is created.
# 
# The FQDN becomes HOSTNAME . ENVIRONMENT . DOMAIN
# (e.g. someapp-webserver-000.prod.example.com)
#
# The environment portion is taken from the puppet configuration file. "Prod"
# is used if the environment is "production" (puppet's default).
# 
# Once the hostname is set, the puppet service is restarted so that the certs
# are generated for the correct hostname.
# Finally the hostname is set on the "Name" tag of the ec2 instance
# 
# Dependencies:
# This script assumes puppet is configured and running, and that the following
# packages are installed:
# * route53-cli
# * aws cli
# * ec2-metadata
#
# Date: 2013-12-26
# Author: Tyler Stroud <tyler@tylerstroud.com>

# BEGIN CONFIGURATION
# Set these variables accordingly
HOSTED_ZONE_ID=
DOMAIN=
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=

ROLE_FILE=/etc/role
EC2_METADATA=/opt/aws/bin/ec2-metadata
AWS=/usr/bin/aws
ROUTE53=/usr/bin/route53
HOSTNAME_CMD=/bin/hostname
LOGGER=/usr/bin/logger
CAT=/bin/cat
SED=/bin/sed
AWK=/bin/awk
PRINTF=/usr/bin/printf
CURL=/usr/bin/curl 
# END CONFIGURATION

export AWS_ACCESS_KEY_ID
export AWS_SECRET_ACCESS_KEY

# Set environment value from the puppet config
ENV=$(awk -F "=" '/^environment/ {print $2}' /etc/puppet/puppet.conf | sed 's/[\t ]*//g')
# Use "prod" if env is "production" or empty
if [ "$ENV" == "production" ] || [ -z "$ENV" ]; then
    ENV=prod
fi

# Exit and do nothing if SYSTEM_ROLE is empty
SYSTEM_ROLE=`[ -f $ROLE_FILE ] && $CAT $ROLE_FILE`
if [ -z $SYSTEM_ROLE ]; then
    exit 0
fi

# The system role must be in the form of application_role (e.g., someapp_webserver, someapp_mongod, etc)
app=${SYSTEM_ROLE%_*}
role=${SYSTEM_ROLE#*_}

hosts=`$ROUTE53 get $HOSTED_ZONE_ID | $AWK -v pattern=^$app-$role.*$ENV.$DOMAIN '$0~pattern { print $1 }'`
IFS=$'\n' sorted=($(sort <<<"${hosts[*]}"))
hosts=($hosts)

ipv4=`$CURL -fs http://169.254.169.254/latest/meta-data/public-ipv4`
i=0
while [ true ]; do
    num=`$PRINTF "%03d" $i`
    hostname="$app-$role-$num.$ENV"
    fqdn="$hostname.$DOMAIN"
    if [ "${hosts[$i]}" != "$fqdn." ]; then
        $SED -i "s/^HOSTNAME=.*$/HOSTNAME=$fqdn/" /etc/sysconfig/network
        $HOSTNAME_CMD $fqdn
        # Add fqdn to hosts file
        $CAT<<EOF > /etc/hosts
# This file is automatically generated by /usr/sbin/onboot.sh script
127.0.0.1 localhost
$ipv4 $fqdn $hostname
EOF
        $LOGGER "Setting hostname: $fqdn"
        $LOGGER "Creating CNAME record in Route53"
        ec2_public_hostname=`$EC2_METADATA -p | $SED 's/public-hostname: //'`

        $ROUTE53 add_record $HOSTED_ZONE_ID $fqdn CNAME $ec2_public_hostname 30 > /tmp/create-route53-cname.out

        # Restart puppet after hostname change
        /sbin/service puppet start
        break
    fi

    let i+=1
done

# Finally, set hostname as "Name" tag
instance_id=`$EC2_METADATA -i | awk '{ print $2 }'`
aws ec2 create-tags --region=$AWS_REGION --resources=$instance_id --tags Key=Name,Value=$hostname
{% endhighlight %}    

There's alot going on there, so let me break it down a bit. This script looks at `/etc/ROLE` to determine the role and the pattern of what the  hostname should look like.
It queries Route53 for a list of all the CNAME records matching the pattern. Then it begins iterating over the sorted hostnames from Route53, and when it finds a missing number,
it uses that as the machine's hostname. The puppet environment is used to set the `environment` portion of the hostname. Once the hostname is set, puppet is started. The puppet
daemon should not run automatically at boot (`chkconfig puppet off`), but should be started by this script to ensure that it contacts the puppetmaster using the correct hostname
(not the EC2 assigned hostname).

You will need an accompanying script to handle removing the CNAME record from Route53 and deactivating the node on the puppetmaster on shutdown/reboot. Here's my script for this:

{% highlight bash %}
#!/bin/bash
#
# /usr/sbin/shutdown.sh
#
# Deletes this host's CNAME record from route53
# 
# Author: Tyler Stroud <tyler@tylerstroud.com>

# BEGIN CONFIGURATION
HOSTED_ZONE_ID=
DOMAIN=
TTL=
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
PUPPETMASTER=
# END CONFIGURATION

export AWS_ACCESS_KEY_ID
export AWS_SECRET_ACCESS_KEY

ROUTE53=/usr/bin/route53
LOGGER=/usr/bin/logger
CURL=/usr/bin/curl

ec2_public_hostname=`$CURL -fs http://169.254.169.254/latest/meta-data/public-hostname`
hostname=$(hostname)

$LOGGER "Deactivating node on the puppet master"
puppetmaster_command_url=https://$PUPPETMASTER:8081/v2/commands
cert=/var/lib/puppet/ssl/certs/$hostname.pem
key=/var/lib/puppet/ssl/private_keys/$hostname.pem
ca=/var/lib/puppet/ssl/certs/ca.pem

$CURL -H 'Accept: application/json' $puppetmaster_command_url --cert $cert --cacert $ca --key $key --data-urlencode 'payload={"command":"deactivate node","version": 1,"payload":"\"'$hostname'\""}'

$LOGGER "Deleting certificate for $hostname"
$CURL -k -X DELETE -H "Accept: pson" https://$PUPPETMASTER:8140/production/certificate_status/$hostname > /tmp/delete-cert 2>&1

$LOGGER "Deleting CNAME record for $hostname"
$ROUTE53 del_record $HOSTED_ZONE_ID $hostname CNAME $ec2_public_hostname $TTL > /tmp/delete-route53-cname.out 2>&1
{% endhighlight %}

This script issues a couple of API calls to the puppetmaster to deactivate the node and remove it's certificates and removes the CNAME record from Route53. You'll need to make sure you've got the puppetmaster
configured to accept these incoming API calls.

With these scripts baked into your server image, you can now configure autoscaling and have human readable hostnames set automatically. You just need the user-data used by the autoscaling group to
set the value of `/etc/ROLE`.

Hopefully someone finds this helpful. Happy DevOps-ing!.

