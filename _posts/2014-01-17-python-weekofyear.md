---
layout: post
title: python 与 java 关于时间认定的差异
keywords: python
category:  python
---
###1.  区间范围差异###

有时数据会通过不同语言进行收集，如果生成周数据报表需要对多语言的时间进行时间一致处理，需要知道如下差异

#### java
	xxxx-01-01 所在周为第一周
	dayofweek [sunday-saturday]-[1-7]
	weekofyear[1-53],谋年的最后一周可能与其下一年1.1号同属于第一周

#### python
	weekofyear [00-53]
	dayofweek  [monday-sunday]-[0-6]



###2. 通过strftime读取python时间属性
{% highlight python %}

when = datetime.date(1981, 6, 16)
print "16/6/1981 was:"
print when.strftime("Day %w of the week (a %A). Day %d of the month (%B).")
print when.strftime("Day %j of the year (%Y), in week %W of the year.")

#=> 16/6/1981 was:
#=> Day 2 of the week (a Tuesday). Day 16 of the month (June).
#=> Day 167 of the year (1981), in week 24 of the year.

{% endhighlight %}