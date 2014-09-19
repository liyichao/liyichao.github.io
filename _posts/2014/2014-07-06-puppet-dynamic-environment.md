---
layout: post
title: "puppet dynamic environment"
category: 
tags: []
---
{% include JB/setup %}

## puppet environment

puppet environment是独立的puppet环境，每个环境有自己的modules和manifests目录。agent可以在配置文件或者命令行指定自己要哪个环境的配置。

如agent运行:

    $ puppet agent --environment testing
    
那么就应用testing环境的配置。


## puppet dynamic environment

puppet静态环境是指以下配置：

    [main]
      server = puppet.example.com
      environment = production
      confdir = /etc/puppet
    [agent]
      report = true
      show_diff = true
    [production]
      manifest = /etc/puppet/environments/production/manifests/site.pp
      modulepath = /etc/puppet/environments/production/modules
    [testing]
      manifest = /etc/puppet/environments/testing/manifests/site.pp
      modulepath = /etc/puppet/environments/testing/modules
    [development]
      manifest = /etc/puppet/environments/development/manifests/site.pp
      modulepath = /etc/puppet/environments/development/modules
    
对于每个环境，都要在puppet.conf里有一个配置项，相比之下，使用fact变量$environment的配置就是动态的：

        [main]
          server = puppet.example.com
          environment = production
          confdir = /etc/puppet
        [master]
          environment = production
          manifest    = $confdir/environments/$environment/manifests/site.pp
          modulepath  = $confdir/environments/$environment/modules
        [agent]
          report = true
          show_diff = true
          environment = production

puppet v3.5.0以后，配置更为简单：

        [main]
          environmentpath = $confdir/environments
这种只指定目录的设计具有扩展性，用户可以随意增加和删除环境，只需要在$confdir/environments目录下创建目录和删除目录就可以了，而不用改puppet.conf，这也使得自动部署变得方便。

## puppet动态环境的自动部署

利用[post-receive hook](http://puppetlabs.com/blog/git-workflow-and-puppet-environments)。工作流程是开发者git clone puppet仓库，新建分支，如my_feature，然后git push。有了那个post-receive hook，远程仓库收到push请求，会ssh到puppet master所在的机器，在$confdir/environments/目录下新建目录my_feature，然后把my_feature分支里的内容clone过来。

使用gitlab的Webhooks自动部署实质也是post-receive，只是webhooks的接口是url post请求，所以需要写个小服务器。