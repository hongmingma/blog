---
layout: post
title:  git 分支管理经验总结
category: [代码管理]
---
[参考：A successful Git branching model](http://nvie.com/posts/a-successful-git-branching-model/)

[参考：一个成功的Git分支模型](http://www.juvenxu.com/2010/11/28/a-successful-git-branching-model/)


### git分支管理全景图
![Alt text](/images/git-qjt.png)


#### 主要分支

master和develop是俩个主要分支,这俩个分支是一直要维护的

> master 分支是产品相对稳定的代码。

> develop分支反映了开发过程中最新变更，当develop分支上的源码到达一个可发布的稳定的状态时需要根据版本号打上tag,把所有变更都合并回master分支
（建议每次合并到master分支，都根据版本号打tag,方便版本确认,master的合并不要太随意）

#### 支持性分支

通过生命周期短暂的支持性分支来帮助团队成员间解决并行开发的问题，如追踪产品特性、准备产品发布版本、以及快速修复bug，这些分支最终都会被删掉，这些分支
来源于哪个分支，在删除前需要再把变更合并回原来的分支，需要严格遵守。
根据解决问题的场景主要包括

* 特性分支（feature branch）

>>可能的分支来源：develop,必须合并回：develop

>>解决问题场景：

多个人开发产品多个独立功能，不同功能发布的时间节点不一致，所以需要根据feature集合建立不同的特性分支，建议在多个人开发同一个产品的时候，
  develop分支也尽量保持不同feature的稳定代码，不稳定的代码尽量拿到不同的feature分支去做，这样别人就很容易通过稳定的develop分支checkout自己的特性分支出来
  （在小团队的小项目里，大家为了方便所有的feature都直接在develop上开发，在新特性开发一半过程中，一旦有更紧急上线的特性插进来想找一份干净的develop分支再checkout
   一个新特性分支都很困难，这时只能通过master分支checkout一个新特性分支出来，但是一旦master分支与develop分支代码相差版本，而正好这个新特性依赖的develop的某个
   版本没有merge到master,这时候就很崩溃,只能硬着头皮在改的乱七八糟的develop分支checkout新分支出来，有代码洁癖的人是会疯的，注释掉代码、后期还有解决冲突...）
{% highlight java %}
$ git checkout -b myfeature develop
$ git checkout develop
$ git merge --no-ff myfeature
$ git branch -d myfeature
$ git push origin develop
{% endhighlight %}

> 发布分支（release branch）

>>可能的分支来源：develop,必须合并回：develop

>>解决问题场景：多个人开发产品多个独立功能已经完毕，代码基本稳定，这时可以根据版本号打tag,创建发布分支用于修bug,使dev保持相对稳定的功能集合
{% highlight java %}
#创建分支
$ git checkout -b releases-1.2 develop
$ ./bump-version.sh 1.2
$ git commit -a -m “Bumped version number to 1.2”
#修订完bug合并到master
$ git checkout master
$ git merge –no-ff release-1.2
$ git tag -a 1.2
#避免太乱，删除这个发布分支
$ git branch -d release-1.2
{% endhighlight %}

> 热补丁分支（hotfix branch）

>>可能的分支来源：master 必须合并回：develop和master  分支命名约定：hotfix-*

>>解决问题场景：
从对应版本的标签创建出一个热补丁分支。使用热补丁分支的主要作用是团队成员在develop可以继续工作，而另外的人可以在热补丁分支上进行快速的产品bug修复。
{% highlight java %}
#创建分支，不要忘了在创建热补丁分之后设定一个新的版本号！
$ git checkout -b hotfix-1.2.1 master
$ ./bump-version.sh 1.2.1
$ git commit -a -m “Bumped version number to 1.2.1″

#bug修复后merge到master同时更新tag
$ git checkout master
$ git merge –no-ff hotfix-1.2.1
$ git tag -a 1.2.1

#代码合并到develop分支
$ git checkout develop
$ git merge –no-ff hotfix-1.2.1

#删除分支
$ git branch -d hotfix-1.2.1
{% endhighlight %}

#### 总结

* 新功能开发，建议在特性分支上完成

* 构成新版本的特性开发完毕，将特性分支代码合并到dev分支，同时创建发布版本分支，根据发布版本打tag,并在其上修改bug

* 上线后的紧急bug,需要通过master分支checkout 补丁分支来修复





