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

>可能的分支来源：develop,必须合并回：develop

>解决问题场景：当开始开发一个特性的时候，该特性会成为哪个发布版本的一部分，往往还不知道。多人开发产品多个独立功能，不同功能发布的时间节点不一致，
所以需要根据feature集合建立不同的特性分支,解决并行开发的同时，也方便后期对feature分支代码合并构成不同节点的待发布代码的集合。
特性分支的重点是，只要特性还在开发，该分支就会一直存在，不过它最终会被合并回develop分支（将该特性加入到发布版本中），或者被丢弃（如果试验的结果令人失望）。
{% highlight java %}
$ git checkout -b myfeature develop
#TODO  这个feature代码开发完后，需要把代码merge会develop分支，同时删除该分支
$ git checkout develop
$ git merge --no-ff myfeature
$ git branch -d myfeature
$ git push origin develop
{% endhighlight %}

* 发布分支（release branch）

>可能的分支来源：develop,必须合并回：develop

>解决问题场景：
发布分支为准备新的产品版本发布做支持，它允许你在最后时刻检查所有的细节，它还允许你修复小bug以及准备版本发布的元数据（例如版本号，构建日期等等）。
正是在发布分支创建的时候，才创建基于项目版本号规则的发布版本号。在发布分支做这些事情之后，develop分支就会显得比较干净，功能完整，也方便为下一大版本发布接受特性。
修订完bug合并到master,master分支上的每一个commit都对应一个新版本，master分支上的commit必须被打上标签（tag），以方便将来寻找历史版本。


{% highlight java %}
#创建分支
$ git checkout -b releases-1.2 develop
$ ./bump-version.sh 1.2
$ git commit -a -m “Bumped version number to 1.2”

$ git checkout master
$ git merge –no-ff release-1.2
$ git tag -a 1.2

#发布分支上的变更需要合并回develop，这样将来的版本也能包含相关的bug修复
$ git checkout develop
$ git merge –no-ff release-1.2

#避免太乱，删除这个发布分支
$ git branch -d release-1.2
{% endhighlight %}

* 热补丁分支（hotfix branch）

>可能的分支来源：master 必须合并回：develop和master  分支命名约定：hotfix-*

>解决问题场景：
从对应版本的标签创建出一个热补丁分支。使用热补丁分支的主要作用是团队成员在develop可以继续工作，而另外的人可以在热补丁分支上进行快速的产品bug修复。
热补丁分支从master分支创建。例如，假设1.2是当前正在被使用的产品版本，由于一个严重的bug，产品引起了很多问题。同时，develop分支还处于不稳定状态，
无法发布新的版本。这时我们可以创建一个热补丁分支，并在该分支上修复问题，同时变更版本号。如果1.2不是出现问题的版本，则需要先根据出现问题的 tag checkout
出热补丁分支，方法类似（$git checkoug master  $git checkout -b hotfix-1.1.1 1.1）
{% highlight java %}

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

* 新功能开发在不确定新版本的功能集合时，建议在特性分支上来完成开发需求,这样团队可以保持并行开发，功能组合灵活。develop分支干净，功能完整。

* 将构成发布版本的所有功能的特性分支合并到dev分支后，需要创建发布分支，在该分支上定版本\修bug\打tag\再合并到master、dev

* 上线后的紧急bug,需要通过master分支（如果master最新版本不是出问题的版本，需要通过tag来产生hotfix分支） checkout 补丁分支来修复

#### 小团队开发思考
如果并行开发不同版本的情况极少，可以简化一下流程，去掉特性分支，所有新功能均在dev来开发、提测、改bug、定版本、打tag和并到master,dev也可以通过tag
信息追溯的不同时期的稳定代码，有需求可以通过tag产生分支来开发，master分支保持干净的提交，每次提交都保证是一个完整的版本，所有的hotfix分支都需要
通过master来产生，这要维护成本会少很多。

#### 小技巧

git merge --no-ff myfeature

上述代码中的--no-ff标记会使合并永远创建一个新的commit对象，这么做可以避免丢失特性分支存在的历史信息，同时也能清晰的展现一组commit一起构成一个特性。

