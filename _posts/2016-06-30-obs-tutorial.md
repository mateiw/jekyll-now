---
layout: post
title: Suse Open Build Service tutorial
---

Suse's [Open Build Service](https://build.opensuse.org/) (OBS)  is a tool for building and distributing software packages for various Linux distros. The main target is of course OpenSUSE and SLES but there's also support for RedHat, Debian, Ubuntu and Arch.

It consists of a web service that takes care of building the packages and publish them to repositories and of a command line tool (osc).

OpenSuse uses RPM as its packaging format. Building RPMs is possible using the rpmbuild command but the osc command line tool makes it much more convenient to build the package locally or remotely in OBS.

If building locally osc can pull the build dependencies of the package and prepare an isolated environment where the build will take place. Under the hood it (usually) creates an chroot with all the needed dependencies installed and it executes the rpmbuild in that environment. So no need to fiddle with the packages installed on your machine to make the build work.

Now let's see how it works in practice. Let's build our first package with OBS. Altough some of the steps outlined below can be performed using the web GUI I'll focus on the command line client utility `osc`.

### 1. Sign up to OBS and install the client tools

The first obvious step is to get an account on [Open Build Service](https://build.opensuse.org/). Let's asume for the rest of the tutorial that your OBS username is `obsuser`.

You will also need the client tools installed. On OpenSuse this is only a matter of running:

```
myuser@linux:~> zypper in osc
```
 
If you use another distro you may also get `osc` from [OpenSUSE Tools](http://download.opensuse.org/repositories/openSUSE:/Tools/).


### 2. Checkout your home project

Once you log in you can go to your home project. This is created by default by OBS and it has the name `home:obsuser`.

To check it out run:

```
myuser@linux:~> osc checkout home:obsuser
```

### 3. Create the package in OBS

Next is of course the package. For the sake of simplicity let's package [jupp](https://www.mirbsd.org/jupp), a portable version of JOE's Own Editor. To create the package execute the following in the directory where you've checked out your home project:

```
myuser@linux:~/home:obsuser> osc meta pkg -e home:obsuser jupp 
```

This will open your text default editor with an empty xml template. Fill in the details of the package like this:
{% highlight xml %} 
<package name="jupp">
  <title>A text editor</title> <!-- Title of package -->
  <description>
    This is the portable version of JOE’s Own Editor, which is currently developed at sourceforge
  </description> <!-- for long description --> 
. . .
  <url>https://www.mirbsd.org/jupp</url>
. . . 
</package>

{% endhighlight %}

After saving and exiting your editor osc will send the details to OBS.

Go to the directory that contains `home:obsuser` and update the project:

``` 
myuser@linux:~/> osc up 
```

This will create an empty directory home:obsuser/jupp.

### 4. Define repositories

You need at least one repository for your project. Usually here you specify a distro version against which you want you package built. In my case I want to build packages for OpenSUSE Leap 42.1.

``` 
myuser@linux:~/home:obsuser/jupp> osc meta prj -e home:obsuser
```

Define the repository inside the XML. The <path> element points to another OBS project, in this case the project for Leap 42.1.

{% highlight xml %} 
<project name="home:mateialbu">
  <title/>
  <description/>
  <person userid="obsuser" role="maintainer"/>
  <repository name="openSUSE_Leap_42.1">
    <path project="openSUSE:Leap:42.1" repository="standard"/>
    <arch>x86_64</arch>
    <arch>i586</arch>
  </repository>
</project>

{% endhighlight %}


### 5. Create the spec file

Now it's time for the real work. The .spec file contains both metadata and build instructions. This is the central piece of an RPM package.

So let's create one for our package. Use you favourite editor to create a file named jupp.spec. At least on OpenSUSE you get a template .spec file where you can fill the details. Paste the following into your file:

{% highlight bash %}

#
# spec file for package jupp
#
# Copyright (c) 2016 SUSE LINUX Products GmbH, Nuernberg, Germany.
#
# All modifications and additions to the file contributed by third parties
# remain the property of their copyright owners, unless otherwise agreed
# upon. The license for this file, and modifications and additions to the
# file, is the same license as for the pristine package itself (unless the
# license for the pristine package is not an Open Source License, in which
# case the license is the MIT License). An "Open Source License" is a
# license that conforms to the Open Source Definition (Version 1.9)
# published by the Open Source Initiative.

# Please submit bugfixes or comments via http://bugs.opensuse.org/
#

Name:           jupp
Version:        3.1jupp28
Release:        0
License:        GPL-1.0
Summary:        A Text Editor
Url:            https://www.mirbsd.org/jupp
Group:          Productivity/Editors/Other
Source:         https://www.mirbsd.org/MirOS/dist/jupp/joe-3.1jupp28.tgz
Conflicts:      joe
BuildRequires:  automake
BuildRequires:  ncurses-devel
BuildRoot:      %{_tmppath}/%{name}-%{version}-build

%description
This is the portable version of JOE’s Own Editor, which is currently developed at sourceforge, licenced under the GNU General Public License, Version 1, using autoconf/automake. This version has been enhanced by several functions intended for programmers or other professional users, and has a lot of bugs fixed. It is based upon an older version of joe because these behave better overall.

%prep
%setup -q -n jupp
chmod +x configure

%build
%configure \
  --prefix=%{_prefix}

make %{?_smp_mflags}

%install
make install DESTDIR=%{buildroot} %{?_smp_mflags}
for i in jmacs jpico jstar rjoe; do
  ln -s joe.1.gz %{buildroot}%{_mandir}/man1/$i.1.gz
done
rm -rf %{buildroot}/%{_datadir}/%{name}/lang

%post

%postun

%files
%defattr(-,root,root)
%{_mandir}/man1/*
%config %{_sysconfdir}/joe/*
%dir %{_sysconfdir}/joe
%{_bindir}/*

%changelog

{% endhighlight %}

A detailed explanation of the .spec file is out of scope of this tutorial but the file is pretty self explanatory.

### 6. Test the build locally

To test the spec file, trigger the build locally. In the package directory run:

```
myuser@linux:~/home:obsuser/jupp> osc build
```
 

If everything goes well osc will inform you that the resulted rpm is in:

```
/var/tmp/build-root/openSUSE_Leap_42.1-x86_64/home/abuild/rpmbuild/RPMS/x86_64/jupp-3.1jupp28-0.x86_64.rpm
```

 
### 7. Add the files to the project

osc works similarly to a VCS like git. Firs you have to add the files:

```
myuser@linux:~/home:obsuser/jupp> osc add <file>
```

or simply

```
myuser@linux:~/home:obsuser/jupp> osc addremove 
```

to add/remove all files in the package dir.

### 8. Add the changelog

```
myuser@linux:~/home:obsuser/jupp> osc vc
```

This will open your default editor with a template. Fill the initial message:

```
------------------------------------------------------------------- 
Wed Jun 29 13:40:29 UTC 2016 - <user.email>
- Initial version of the package
```

 

A new file jupp.changes will be written in the package directory.

To check the status before committing run:
```
osc status
```

To add jupp.changes do a:

```
myuser@linux:~/home:obsuser/jupp> osc add jupp.changes 
A    jupp.changes 
```

Now finally everything ready to commit:

``` 
myuser@linux:~/home:obsuser/jupp> osc commit 
Sending    joe-3.1jupp28.tgz 
Sending    jupp.spec 
Sending    jupp.changes 
Transmitting file data .. 
Committed revision 1.

```

 
### 9. Check the build status

After you commit OBS will schedule the build. To check the status you can either go to the web interface or use osc:

```
myuser@linux:~/home:obsuser/jupp> osc results
openSUSE_Leap_42.1   x86_64     finished*
openSUSE_Leap_42.1   i586       succeeded
```

### 10. Download and install the package

You can download and install the package from the web interface. Go to the package page and look for the aptly named "Download package" link. On OpenSuse you'll get the option to use One Click Install.

If you prefer the command line another option is to use add the repository and install the package. First get the repo url:

``` 
myuser@linux:~/home:obsuser/jupp> osc repourls 
http://download.opensuse.org/repositories/home:/obsuser/openSUSE_Leap_42.1/home:obsuser.repo
```

Then simply add the repository using zypper and install the package:

``` 
myuser@linux:~/home:obsuser/jupp> sudo zypper ar http://download.opensuse.org/repositories/home:/obsuser/openSUSE_Leap_42.1/home:obsuser.repo
Repository 'home:obsuser (openSUSE_Leap_42.1)' successfully added 
Enabled     : Yes
Autorefresh : No
GPG Check   : Yes
Priority    : 99
URI         : http://download.opensuse.org/repositories/home:/obsuser/openSUSE_Leap_42.1/ 

myuser@linux:~/home:obsuser/jupp> sudo zypper ref
. . . 
myuser@linux:~/home:obsuser/jupp> sudo zypper in jupp
```
 
And voila! You have jupp installed on your machine. 

### Further resources

* [OpenSuse's Build Service Tutorial](https://en.opensuse.org/openSUSE:Build_Service_Tutorial)
* [Build Service Tips and Tricks](https://en.opensuse.org/openSUSE:Build_Service_Tips_and_Tricks)
* [Fedora's RPM Guide](https://docs.fedoraproject.org/en-US/Fedora_Draft_Documentation/0.1/html/RPM_Guide/index.html)

