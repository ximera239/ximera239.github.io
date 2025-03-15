---
title: CDH5 and Cloudera Manager
author: Evgeny
layout: post
permalink: /2014-12-09-cdh5-and-cloudera-manager/
categories:
  - Uncategorized

tags:
  - CDH5

---

This is my third attempt to write this post.

The first version of this post started (on 23.11!) with:

> I tried Hortonworks [tutorial](http://hortonworks.com/hadoop-tutorial/introducing-apache-ambari-deploying-managing-apache-hadoop/) which shows how to install Ambari in a virtual machine, and then install a stack of HDP products.
The tutorial itself, in my opinion, is not good enough. It's just a sequence of steps with no understanding at the end about what was really done and why.

> From another point of view, it introduces a set of new (for me) products, like Vagrant. OK, looks like a useful product.
First of all - Vagrant has its own library of boxes available [here](https://vagrantcloud.com/discover/featured).

> I got an idea to try to install CDH on top of Vagrant. It's something similar to my previous experience, but more automated (I want to try Cloudera Manager).

> I'll use [this](http://www.cloudera.com/content/cloudera/en/documentation/core/latest/topics/cm_ig_install_path_a.html#cmig_topic_6_5_unique_2) tutorial about CDH. I'll also reference the Hortonworks [tutorial](http://hortonworks.com/hadoop-tutorial/introducing-apache-ambari-deploying-managing-apache-hadoop/) in case of problems =)

I expected that I could finish with something like "click then just next/accept buttons." But that wasn't the case.

This was not so easy. However, by solving installation problems, I started to understand the process a little bit better.

### Summary

1. I was not able to install Cloudera Manager in a Vagrant box with 4 GB of RAM. Probably because of VM limitations.
2. I did not understand how to continue an interrupted installation (at some step, for example after installing the agent). With Vagrant, I just re-created the VM each time. With a set of 3 servers (also virtual, but on top of a more powerful cluster with 8GB RAM on each), I needed to uninstall everything from each node (scm-agent, scm-server, scm-server-db) and start almost from the beginning. [This guide was very useful](http://www.cloudera.com/content/cloudera/en/documentation/archives/cloudera-manager-4/v4-5-3/Cloudera-Manager-Enterprise-Edition-Installation-Guide/cmeeig_topic_18_1.html)

In general, the process was not so difficult (when you repeat it 10 or 15 times, of course).

* I used a cluster of 3 Ubuntu Precise servers:

```
Distributor ID: Ubuntu
Description:    Ubuntu 12.04.5 LTS
Release:        12.04
Codename:       precise
```

* On the node which will be used as a server, you probably need to manually install PostgreSQL 8.4 (because of conflicting dependencies, it cannot be installed automatically by Cloudera Manager Installer). More precisely: 

```
PostgreSQL 8.4 depends: libc6 (>= 2.15), libcomerr2 (>= 1.01), libgssapi-krb5-2 (>= 1.8+dfsg), libkrb5-3 (>= 1.6.dfsg.2), libldap-2.4-2 (>= 2.4.7), libpam0g (>= 0.99.7.1), libpq5 (>= 8.3~rc1-1~), libssl1.0.0 (>= 1.0.0), libxml2 (>= 2.7.4), postgresql-client-8.4, postgresql-common (>= 130~), tzdata, ssl-cert, locales
# no exact versions are specified. And the candidate in our corporate network was something not applicable
```
        
~~~ bash
$ sudo apt-get install postgresql-8.4 postgresql-common=154.pgdg12.4+1 postgresql-client-common=154.pgdg12.4+1
~~~
        
* Add a new user with sudo privileges on each node, and generate and distribute SSH keys:

~~~
cdh@dev26i:~$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/cdh/.ssh/id_rsa):
Created directory '/home/cdh/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/cdh/.ssh/id_rsa.
Your public key has been saved in /home/cdh/.ssh/id_rsa.pub.
The key fingerprint is:
35:e3:7d:ff:b0:2a:01:e3:a8:9f:8c:bf:a1:47:ee:34 cdh@dev26i.vs.os.yandex.net
The key's randomart image is:
+--[ RSA 2048]----+
|                 |
|                 |
|          +      |
|        oo +     |
|       oSo. . .  |
|      o . .  . . |
|     +E    .  . .|
|    .=o+  .    o.|
|    o=B.   .... .|
+-----------------+

cdh@dev26i:~$ cd .ssh/
cdh@dev26i:~/.ssh$ cat id_rsa.pub >> authorized_keys
cdh@dev26i:~/.ssh$ cd ../
cdh@dev26i:~$ tar cvf ssh.tar .ssh
.ssh/
.ssh/authorized_keys
.ssh/id_rsa
.ssh/id_rsa.pub
cdh@dev26i:~$
~~~

Download the id_rsa file to your laptop.

* Check that each node has an IPv4 address and that it is in the /etc/hosts file. Also verify that both of these commands work:

~~~
root@dev28i:~# hostname
dev28i
root@dev28i:~# hostname -f
dev28i.vs.os.yandex.net
~~~

Related [issue with hosts](http://community.cloudera.com/t5/CDH-Manual-Installation/Problem-during-installation-receiving-heartbeat-from-agent/td-p/1109) and [with IPv6-only host](http://www.cloudera.com/content/cloudera/en/documentation/cdh4/latest/CDH4-Installation-Guide/cdh4ig_topic_11_1.html)

* Check if the NTP daemon is installed and running (required for agent heartbeat check)

* Download and run the Cloudera Manager binary file:

~~~ bash
$ wget http://archive.cloudera.com/cm5/installer/latest/cloudera-manager-installer.bin
--2014-11-23 17:12:31--  http://archive.cloudera.com/cm5/installer/latest/cloudera-manager-installer.bin
Resolving archive.cloudera.com (archive.cloudera.com)... 54.230.98.120, 54.230.98.191, 54.230.98.203, ...
Connecting to archive.cloudera.com (archive.cloudera.com)|54.230.98.120|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 514202 (502K) [application/octet-stream]
Saving to: 'cloudera-manager-installer.bin'

100%[================================================================================================================================================================>] 514,202     81.8KB/s   in 6.6s

2014-11-23 17:12:44 (76.0 KB/s) - 'cloudera-manager-installer.bin' saved [514202/514202]
$ chmod u+x cloudera-manager-installer.bin
$ ./cloudera-manager-installer.bin
~~~

* Install until you see a link for your browser. After that, I did almost everything with the default settings.

By the way, just after installation, I had health check problems because of host names (I just turned this check off). I also had some reported problems with memory settings (probably because of the unbalanced installation across hosts).