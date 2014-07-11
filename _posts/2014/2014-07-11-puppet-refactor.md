---
layout: post
title: "puppet 配置重构"
category: 
tags: []
---
{% include JB/setup %}

##目前打算部署的变更

对zhihu_puppet重构，目前第一步变更如下：

* 现有/etc/puppet/manifests和/etc/puppet/modules移到/etc/puppet/environments/production文件夹，zhihu\_puppet仓库只管理production文件夹的文件
* 支持本地开发，自动部署测试环境和生产环境，具体工作流程见[githooks](http://git.in.zhihu.com/lyc/githooks/tree/master)

如果大家觉得没什么问题，我就部署到puppet主机去了。

##重构思路

* 把所有与站点相关，经常变动的内容，都放到[Hiera](http://docs.puppetlabs.com/hiera/1/#why-hiera)，具体包括：
    * 用户
        * key
        * email
        * phone
        * name
    * 主机
        * 允许登录的用户列表
        * 有sudo权限的用户列表
        * ip地址
    * 节点分类：指一个节点包含哪些class
 
* nagios所有配置都由puppet管理，nagiosplugins，zhihu_config/nagios将不再存在，而会成为puppet一个module
* 用[配置导出](http://docs.puppetlabs.com/puppet/latest/reference/lang_exported.html)实现nagios自动报警

