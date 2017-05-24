---
title: Ubuntu 12.04 on XenServer 6.0.2 doesn't boot - RECOVERY
layout: post
date: 2017-03-17
---

#### Versions

* XenServer 6.0.2 Build 53456p
* Ubuntu 12.04

#### Problem

```
Mar 17 00:58:40 srv xapi: [error|srv|1959896 unix-RPC|VM.start R:7321d8dea9e4|xapi] Vmops.start_paused caught: Using <class 'grub.GrubConf.Grub2ConfigFile'> to parse /boot/grub/grub.cfg: [ Traceback (most recent call last):;   File "/usr/bin/pygrub", line 850, in ?;     raise RuntimeError, "Unable to find partition containing kernel"; RuntimeError: Unable to find partition containing kernel;  ]
```

The origin of this problem is incompatibility of XenServer with pygrub2 bootloader.

#### Recovery

As a prerequisite, install vim :-)

{% highlight bash %}
$ yum --enablerepo=base --disablerepo=citrix install vim-enhanced
{% endhighlight %}

##### Step 1

First we need to show Xen where is the kernel that we want.

```
$ xe vm-list
```

note down the VM UUID that you need to resurrect.

Then we need to edit the grub config on the VM's disk:

{% highlight bash %}
$ export EDITOR=vim
$ /opt/xensource/bin/xe-edit-bootloader -u fe661727-70e8-b2aa-8222-5548cf29456b -p 1
{% endhighlight %}

1. Comment out any submenu sections completely, leave only the default newest kernel.
2. Note down the:
-  kernel path
- the initramdisk path
- the root partition UUID/path

{% highlight bash %}
        echo    'Loading Linux 3.2.0-4-amd64 ...'
        linux   /boot/vmlinuz-3.2.0-4-amd64 root=UUID=3c43ad39-f91b-4beb-a09d-2ba37a6d0e11 ro console=hvc0 quiet
        echo    'Loading initial ramdisk ...'
        initrd  /boot/initrd.img-3.2.0-4-amd64
{% endhighlight %}


3. Now, just in case, note down VM params before changing them:

{% highlight bash %}
$ xe vm-param-get  uuid=fe661727-70e8-b2aa-8222-5548cf29456b param-name=PV-bootloader
pygrub
$ xe vm-param-get  uuid=fe661727-70e8-b2aa-8222-5548cf29456b param-name=PV-bootloader-args
$
$ xe vm-param-get  uuid=fe661727-70e8-b2aa-8222-5548cf29456b param-name=PV-args
-- quiet console=hvc0
{% endhighlight %}

4. Set the boot params for the VM with previously noted values:

{% highlight bash %}
$ xe vm-param-set  uuid=fe661727-70e8-b2aa-8222-5548cf29456b PV-bootloader-args="--kernel=/boot/vmlinuz-3.2.0-4-amd64 --ramdisk=/boot/initrd.img-3.2.0-4-amd64"
$ xe vm-param-set  uuid=fe661727-70e8-b2aa-8222-5548cf29456b PV-args="-- root=UUID=299b9872-0c06-4d84-9b2b-6196db648c1b quiet console=hvc0"
{% endhighlight %}

5. Now you should be able to start the VM:
```
xe vm-start vm=<name>
```

##### Step 2 - install standard grub

When you successfully boot up the VM, deinstall the problematic pygrub2 and replace with standard grub.

{% highlight bash %}
$ apt-get -y update
$ apt-get -y purge grub-pc
$ rm -rf /boot/grub
$ mkdir -p /boot/grub
$ apt-get -y install grub
$ grub-set-default default
$ update-grub -y
{% endhighlight %}

Credits: [ http://discussions.citrix.com/topic/310400-unable-to-find-partition-containing-kernel/]( http://discussions.citrix.com/topic/310400-unable-to-find-partition-containing-kernel/) 
