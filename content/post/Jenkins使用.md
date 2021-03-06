---
title: Jenkins的使用
date: 2016-10-11
tags: ["Jenkins"]
---

**前提: 使用 Git 作为公司的版本控制工具，使用 GitLab 作为对应的服务器**

首先安装 Jenkins plugins, 下面列出两个关键的 Plugins

[GitLab Plugin](https://wiki.jenkins-ci.org/display/JENKINS/GitLab+Plugin)

[Git plugin](https://wiki.jenkins-ci.org/display/JENKINS/Git+Plugin)

### 打包 APK

对于 APK 的打包，相信大家公司内部都有不同的测试环境，不同的功能需要在不同的测试环境上测试，不同的功能又对应不同的分支，如果开发人员负责根据测试人员的需求打包的话，测试人员麻烦，开发人员也麻烦。程序员一个极好的特质就是懒，那么，这时候，Jenkins 提供的`This build is parameterized`功能就派上了用处。

![](/img/jenkins/build_params.png)

勾选了这个选项之后，就可以添加自己需要的参数了，比如:

![测试环境地址 ](/img/jenkins/url.png)
![git的分支名称](/img/jenkins/git_branch.png)

记住这个地方填写的`name`， 这里的名称将会是 shell 里的变量，使用$来引用对应具体的变量，下面将会看到这些变量的使用。

既然环境和分支都已经可选，第二步是配置 Jenkins 去哪个地址拉源代码，即 SCM 配置，如下图:
![](/img/jenkins/scm_git.png)

可以看到，在`Branches to build`的地方，我填的内容是`*/$git_branch`，这个参数就是从前面`This build is parameterized`位置添加的参数，所以用户填了哪个分支，这里就能拿到用户填写的分支，在 Build 的时候也就会拉对应分支的代码。

按照我们的意向拉了对应分支的代码，开头所说的另外一个问题，更改对应的环境，当然，可以使用`ProductFlavor`来解决这个问题，再最后打包的情况下选择对应的`ProductFlavor`，数量少了还好说，数量多了，就会变得很麻烦。因此，这里在`Build Environment`的地方，打勾
`Executor shell script on remote host using ssh`，如下图
![](/img/jenkins/change_env.png)

这里需要插件[SSH Plugin](https://wiki.jenkins.io/display/JENKINS/SSH+plugin?focusedCommentId=43713480)的支持

在`Pre build script`的地方，修改你工程中决定使用哪个环境的文件，这里就可以使用最开头的`$url_host`那个设定的参数。

当然还有`Post build script`，需要的话就同样写一段 shell 来达到你的目的。

有些情况下，你需要获取这次打包对应的 git 信息，可以看到，每个打包都有一个 Git Build Data，这里面有对应的 git 信息，所以，勾选`inject environment variables to the build process`即可将对应的信息添加到这次打包的环境变量中。
这个需要[Environment Injector Plugin](https://wiki.jenkins-ci.org/display/JENKINS/EnvInject+Plugin)插件的支持。
这里有个问题，就是`Evaluated Groovy Script`脚本的执行过程中，使用正常方式无法获取所有的 Jenkins 环境变量，那么需要使用`currentBuild.getEnvironment(currentListener).get('var')`来获取 var 的环境变量。currentBuild 和 currentListener 在 Evaluated Groovy Script 右边的`?`说明里

![](/img/jenkins/build_env_inject.png)

最后，测试人员肯定希望能下载打包完成的 APK，那么在最后的`Post-build Actions`即可实现该需求。

![](/img/jenkins/post_build_action.png)

在`Archive the artifacts`里面填入打包完成的 apk 的路径，则 Jenkins 会以可以下载的形式输出该 Apk 的链接在对应的打包完成页面。

### 打包 AAR

打包 AAR 和打包 APK 的需求是不同的，对于 AAR 的输出，一定希望对开发透明，不需要任何人去点一下 build，才去打包 AAR，这个时候，我们前面安装的 GitLab Plugin 就十分有用。有了它，我们可以设定触发打包任务的条件:

![](/img/jenkins/build_trigger.png)

画红线的 URL 代表 GitLab WebHook 需要回调的 URL，即 GitLab 收到某些事件(push, merge request ..etc)，将会调用该 URL，此时将会触发打包。在最下面还有个红线，就是指定相应的分支，在这些分支另在 GitLab WebHook 回调的时候才进行打包，公司的特殊需求即可在这里定制。

有可能你想流式的打包，也可能你的 aar 有优先级，换句话说，A aar 可能依赖 B aar，那么一定要 B aar 打包完毕后，才能打包 A aar，那样的话你就需要下面这个插件
[Jenkins Parameterized Trigger plugin](https://wiki.jenkins-ci.org/display/JENKINS/Parameterized+Trigger+Plugin)
这个插件将前一个打包 aar 的 Build Env 传给后一个打包 aar 的过程中，那样，就拥有一致的 Build Env 了。
