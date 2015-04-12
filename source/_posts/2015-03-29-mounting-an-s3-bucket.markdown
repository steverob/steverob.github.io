---
layout: post
title: "Mounting an S3 bucket using FUSE"
date: 2015-03-29 16:52:23 +0530
comments: true
categories: sysadmin
published: true
---  

Have you ever wanted to interact with your Amazon S3 bucket just like how you deal with the directories in the file system without having to mess around with an API? I recently came across a really cool tool that allows you to do just that. The technology that this tool uses, like some of you might have guessed, is FUSE (Filesystems in User Space). 
<!--more-->

So what is FUSE? I've heard about it before and I had even used it blindly in college in our FOSS lab but never had a concrete understanding. While I was trying to learn about it, I came across [this page](http://www.cs.nmsu.edu/~pfeiffer/fuse-tutorial/) which puts FUSE in plain terms and I am quoting from it - 

> One of the real contributions of Unix has been the view that "everything is a file". A tremendous number of radically different sorts of objects, from data storage to file format conversions to internal operating system data structures, have been mapped to the file abstraction. One of the more recent directions this view has taken has been Filesystems in User Space, or FUSE (no, the acronym really doesn't work. Oh well). The idea here is that if you can envision your interaction with an object in terms of a directory structure and filesystem operations, you can write a FUSE file system to provide that interaction. You just write code that implements file operations like open(), read(), and write(); when your filesystem is mounted, programs are able to access the data using the standard file operation system calls, which call your code.

This simply blew my mind! After learning about FUSE I could imagine how easily something like S3 or heck even a database like MySQL can be treated as a file system using FUSE.

Okay now the software I found is called [s3fs-fuse](https://github.com/s3fs-fuse/s3fs-fuse). It's written in C++ and although I'm not a big fan of the language reading through the codebase gave a really good insight on writing a FUSE filesystem. 

So here's how you use use s3fs-fuse. Download the project from the [github repo](https://github.com/s3fs-fuse/s3fs-fuse) and install it by following the instructions on the [wiki](https://github.com/s3fs-fuse/s3fs-fuse/wiki/Installation-Notes). Now I could not find a binary release so you'll have to build the project on your own. Its not as hard as it sounds.

Once you have it installed, you need to store your AWS credentials in a file at either `~/.passwd-s3fs` (600 permission) or `/etc/passwd-s3fs` (640 permission) in the format - `accessKeyId:secretAccessKey`.

Now we're all set. Let's say we need to mount our S3 bucket - `fuse-ftw`. To do that we need to run the following command:

```
/usr/bin/s3fs fuse-ftw /mnt
```

and that is it! s3fs also allows provides few options to control the way the bucket is mounted. Some of the useful ones are:

* **default_acl** - lets you set the default [canned access control policy](http://docs.aws.amazon.com/AmazonS3/latest/dev/acl-overview.html) for the objects stored in the bucket. The default value for this is `private`. You can set it to, say, `public-read` to allow everyone to read but restrict all other operations.

* **use_cache** - this is a cool option that lets you tell s3fs to use a local folder to cache the objects (files) so that it can avoid unnecessarily downloading files from S3. It uses MD5 checksums to make sure cache is valid.

* **retries** - this sets the number of times that you would like s3fs to retry an operation upon failure.

Check the wiki for more options and a much more detailed write up on the way this works and its limitations. 

And FUSE is totally cool. I mentioned about treating a database as a filesystem earlier right? Well, I found a python library called [mysqlfuse](https://github.com/clsn/mysqlfuse) that does just that! Check it out.

Have you used FUSE for doing something interesting? Let me know by dropping a comment below.  