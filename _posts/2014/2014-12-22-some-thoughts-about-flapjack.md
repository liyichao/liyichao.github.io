---
layout: post
title: "some thoughts about the design of flapjack"
category:
tags: []
---
{% include JB/setup %}

It seems to me that every checks have to be pre defined. Checks seem to be more of a nagios thing, and not that of a notification engine. For example, I may use an intelligent check engine that checks anomal metrics automatically and  it is unlikely to define checks for every metric because there are too many.

As to the question `How long has a check been failing`, I think it is not a notification engine should care. It is better to let other programs to decide whether there is a problem, flapjack just need to care who should be notified via what media.

Event should contain tags and people just tell which tags they are interested and events will be routed by looking at their tags.

In my view, a notification engine just needs to listen on events coming in and send them to the right persons via the right media. It should not care about checks, entity, they should all be viewed as tags to provide more variability. It is irrelevant to a notification engine what the tag means, because it only uses the tags to decide who should receive the event via which media.

As to the summary field of the event, it is too narrow. For example, I want to include cpu usage, memory usage, iostat, etc into an notification message to help solving the problem. It should just be a tag of event, for example, a tag named notification_body, flapjack should not care about its content.

The event, check data structure are more like from the nagios' view of the metric world which is too old.
