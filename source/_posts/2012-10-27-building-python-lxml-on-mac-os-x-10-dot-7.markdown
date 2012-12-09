---
layout: post
title: "Building Python lxml in a virtualenv on Mac OS X 10.7"
date: 2012-10-27 22:52
comments: true
categories: programming
---

Today I attempted to install **Scrapy** for one of my personal project ideas. It took me hours to work out how to get a working installation. This post is for anyone who googles their error messages hoping for a solution. If you have done just that I have good news! The solution is at the bottom of the post.

So what happened? Although the installation went fine, I found myself getting this error when running Scrapy.

``` bash
ImportError: dlopen([path to my site-packages]/lxml/etree.so, 2):
    Symbol not found: ___xmlStructuredErrorContext.
```

I get this same error if I run ```from lxml import etree```. It would seem **lxml**, a library Scrapy depends on, is unhappy with dynamic linking of another library. But which one?

I had a look with ```otool -L [path to my site-packages]/lxml/etree.so``` (```otool -L``` is for mac what ```ldd``` on any other *nix). In that list, I found something xml-related: ```libxml2.2.dylib```. Looking at the documentation for lxml, it states it requires **libxml2** 2.6.21 or later. Promising! 

We now know we’re linking against an outdated installation of libxml2. Now we could update it (in fact, I believe upgrading to mountain lion will do just that). However, given libxml2 was shipped with the OS, we shouldn’t risk breaking any part of the OS that may rely on version 2.2 (and today is not the day to upgrade to mountain lion).

The lxml installation guide for mac provides some insight and actually suggests setting the environment variable ```STATIC_DEPS=true``` for our install. This will link against a static version of libxml2, so we don’t have to worry about the installed dynamic library. It’ll even go as far as download the newest version, header files and all. How nice!

Unfortunately, the latest version of libxml2 is version 2.9.0, and that happens to not work with lxml on Mac, as noted in the answer to [this stack overflow question](http://stackoverflow.com/questions/12484664/what-am-i-doing-wrong-when-installing-lxml-on-mac-os-x-10-8-1). Someone got their assignment and initialisation mixed up when using POSIX threads. Oh dear!

This is the error you might get at compile time:

```
threads.c: In function 'xmlCleanupThreads':

threads.c:918: error: expected expression before '{' token
```

The answer provided on stackoverflow suggests specifying a version of the build as an option in the ```setup.py``` file. I had a go asking pip to pass that option as follows: ``` STATIC_DEPS=true pip install --install-option="--libxml2-version=2.7.8" lxml```

Unfortunately, that does not work. For some unfathomable reason, it will still attempt to use the latest version of lbixml2 anyway, making the build fail. 

However, if we do attempt a static build like that, what we can then do is browse to the failed build pip graciously kept for us in build/lxml, and run the ```setup.py``` manually. Like so: ```python setup.py build --static-deps --libxml2-version=2.7.8```.

I chose to just build and not install with the ```setup.py``` script, so that I could run pip install and give pip a chance to delete all the temporary files left over from the failed builds. As long as you’re in the virtualenv, the ```setup.py``` script should install in your virtualenv prefix, if you’re rather do that.

In summary, if you are struggling to install lxml on your Mac with anything earlier than mountain lion, the easiest solution, I believe is to run the following

```STATIC_DEPS=true pip install lxml; cd build/lxml && python setup.py build --static-deps --libxml2-version=2.7.8 && pip install lxml```

This should also work outside of a virtualenv, if you wish to install it globally (adding sudo where appropriate).

