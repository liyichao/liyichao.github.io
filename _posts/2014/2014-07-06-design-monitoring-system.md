---
layout: post
title: "design monitoring system"
category: 
tags: []
---
{% include JB/setup %}

##监控系统现状

###架构

* 每台机器上部署:
  * [statsd](https://github.com/armon/statsite)
      * 收集的数据：计时和计数
      * 工作模式：被动监听statsd API发来的metric，汇总一段时间的数据，然后发给graphite
  * [collectd](http://collectd.org/)
      * 收集的数据：CPU load，memory，disk usage
      * 工作模式：每隔一段时间主动收集数据，发给graphite
  * kids
  * [nagios nrpe daemon](http://nagios.sourceforge.net/docs/nrpe/NRPE.pdf)
      * 收集的数据：CPU load，memory，disk usage
      * 工作模式：被动监听监控主机发来的命令，返回ok，warning，critical等，监控主机根据返回值和报警策略决定是否报警

* graphite server。监听各机器送来的指标，并完成：
    * 存储
    * 绘图
* 小脚本机器。从kids，sink订阅消息，把一些指标发往graphite
* nagios server。实现报警功能，运行：
    * nagios插件：check_ping，check_ssh，check_redis，check_http等
    * nagios nrpe插件：发命令给被监控主机的nrpe daemon，检测cpu load，memory，connection等

* 其他杂项：应用程序直接往graphite打数据而不经过statsd

###现状

* 相关仓库：
	* zhihu_puppet：nrpe模块和collectd模块，管理所有被监控机器的设置
		* nrpe模块：
			* check\_\*脚本：供command调用
			* nrpe.cfg：供监控主机nrpe插件调用的command定义
	* zhihu_config：该项目nagios目录管理了监控主机的设置，还保存本应该在nagiosplugins中的插件脚本，如plugins/check_zookeeper.py
	* nagiosplugins：监控主机上的nagios插件脚本

* 相关机器：
	* zindex：statusplugins仓库的代码的部署地，用于从sink，kids订阅数据，打到graphite
	* status：graphite主机
	* nagios：nagios监控主机
		
* nagios加报警流程：
	* nrpe：
		1. nagios主机：在zhihu_config定义新command和新service，运行zhihu_config/deploy.py
		2. 被监控主机：写检测脚本以及定义command，放到zhihu_puppet/nrpe模块，运行puppet agent
	* nagios插件：
		1. 在zhihu_config/commands加入命令定义
		2. 在nagiosplugins加入自定义插件脚本，登录nagios主机存放nagiosplugins仓库的目录/data/apps/nagiosplugins，运行git pull，./bin/buildout

###监控的数据分类


 
##要做的事

####Nagios

把所有报警系统的设置，包括nagios，collectd，nagios nrpe，nagios plugins，都放到puppet，可以叫monitor模块，具体见puppet小节。
 
####puppet

拟对现在puppet配置作以下修改：

* 报警模块隔离出来，设为单独仓库，并通过puppetfile导进整体的puppet配置
* 使用[puppet动态环境](http://puppetlabs.com/blog/git-workflow-and-puppet-environments)。目前的静态环境production，testing，office，不方便配置的测试