---
layout: post
title: "puppet dynamic environment"
category: 
tags: []
---
{% include JB/setup %}

###master的puppet.conf

在puppet.conf里：
    
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

`$environment`将跟agent指定的环境一致，如agent运行:

    $ puppet agent --test --environment testing
    
那么$environment将是testing。

###puppet的git仓库

加上[post-receive hook](http://puppetlabs.com/blog/git-workflow-and-puppet-environments)。工作流程是开发者clone puppet仓库，新建分支，如my_feature，然后git push。有了那个post-receive hook，远程仓库收到push请求，会ssh到puppet master所在的机器，在/etc/puppet/environments/目录下新建目录my_feature，然后把my_feature分支里的内容clone过来，接着，当agent指定my_feature环境时，将会得到my_feature分支的配置。