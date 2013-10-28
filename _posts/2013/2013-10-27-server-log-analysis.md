---
layout: post
title: "服务器log分析"
category: 
tags: []
---
{% include JB/setup %}

>追逐自己喜欢的变化，让字符踩着梦想的鼓点翩翩起舞。

用Ruby做。这样调用`ruby analysis.rb *.log`，在程序里遍历文件名数组`ARGV`，对每个文件，用以下正则表达式选到需要的行

    %r{\s+ GET \s+ (/topic/\d\d\d) \s+ \((.*)\) \s+}x
两个括号里捕获的分别是URL和客户IP地址，分别从`Regexp.last_match`数组的第2，3个元素，建立哈希表`day_access`

    {IP地址=>{访问的URL=>访问次数, ...}, ...} =
                            {:"8.8.9.9"=>{:"/topic/100"=>1}, 
	                         :"8.8.9.10"=>{:"/topic/456"=>1}}
     

每24个文件选出符合当天要求的用户`day_candidate`，维护备选用户名列表`candidate_user`和备选路径列表`candidate_url`。备选用户很好维护，但是备选路径不好维护，因为它依赖于最终的用户列表。假设用户名列表只保存至目前为止符合要求的用户名，路径列表只保存至目前为止符合要求的路径列表，考虑`candidate_user`的用户i不在`day_candidate`里面，这时候i就该从`candidate_user`中删除，那么`candidate_url`呢，有些url是被i访问的，没i的话有些url就不符合以前的日子的要求了。所以`candidate_url`必须保存以前所有天访问它的用户，当然，只要保存在`candidate_user`里的用户。如下

    {:"/topic/100" => [Set[:"8.8.9.9", :"8.8.9.10"], Set[:"8.8.9.9", :"8.8.9.12"], ...]
     :"/topic/101" => [Set[:"8.8.9.9", :"8.8.9.10"], Set[:"8.8.9.9", :"8.8.9.12"], ...], ...}

程序遍历`candiate_user`:

- 用户i不在当天的`day_candidate`。这时，先从`candidate_user`删除用户i，然后遍历`candidate_url`每个url，从它每一天的访问用户中删除用户i，如果删除后该天访问用户数少于2，则从`candidate_url`中删除这个url。
- 用户i在当天`day_candidate`。这时，对于用户i访问的每个url，添加i到`candidate_url`当天的访问用户中去。

        candidate_url[url] << Set[]
        candidate_url[url][-1] << i

