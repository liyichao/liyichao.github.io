---
layout: post
title: "R10K"
category: 
tags: []
---
{% include JB/setup %}

基于[librarion-puppet](https://github.com/rodjek/librarian-puppet)的puppet部署工具。它的特点有以下几个：

* 实现了dynamic environment
* puppet module管理。不同environment可能共享module，如果用dynamic environment，那么模块是在各environment各自的文件夹里，如/etc/puppet/environments/production/modules，/etc/puppet/environments/testing/modules，不能共享，这导致了部署时要下载多份(用puppetfile指定的module要从github或puppet forge上下载)，对此，R10K做了如下的事：
    
    * cache a repository。相同版本的module采用[alternate object directories](https://www.kernel.org/pub/software/scm/git/docs/gitrepository-layout.html)共享一个repository。
    * 同一个repository的不同版本，可以共享commit history。
* 细粒度部署:

        r10k deploy environment <environment name>
        r10k deploy module <module name>
        
* 比librarian-puppet更好的错误处理：librarian-puppet在更新module前把所有已安装的都删除了，发生错误时，系统处于partial deployment状态，而R10K:
>If a critical error occurs when deploying an environment or module, it tries to leave that module in the last good state, and then moves on

* 支持Multiple Environment Repositories，由配置文件很好理解

        # The location to use for storing cached Git repos
        :cachedir: '/var/cache/r10k'

        # A list of git repositories to create
        :sources:
          # This will clone the git repository and instantiate an environment per
          # branch in /etc/puppet/environments
          :app1:
            remote: 'git@github.com:my-org/app1-environments'
            basedir: '/etc/puppet/environments'
          :app2:
            remote: 'git@github.com:my-org/app2-environments'
            basedir: '/etc/puppet/environments'
            
就是每个environment可能都是一个repository，简单的动态环境是只有一个repository，它的所有分支都在/etc/puppet/environments有对应文件夹。

##我的理解

* /etc/puppuet/environemts下每个文件夹都是独立的环境，不通过/etc/puppet/modules共享module，每个独立环境都只用自己的module文件夹，而module共享通过R10k的cache和alternate object directories实现。我的实际使用经验是，当modules文件夹有7.4M时，部署新分支的时间大概在5分钟左右，如果不满意，可以指定部署的module，或者环境。

