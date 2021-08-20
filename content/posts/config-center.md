---
title: "Config Center"
date: 2021-08-16T16:37:06+08:00
draft: true
---




环境隔离、版本匹配、版本管理

配置分类
按配置的来源划分，主要有源代码（俗称hard-code），文件，数据库和远程调用。
按配置的适用环境划分，可分为开发环境，测试环境，预发布环境，生产环境等。
按配置的集成阶段划分，可分为编译时，打包时和运行时。编译时，最常见的有两种，一是源代码级的配置，二是把配置文件和源代码一起提交到代码仓库中。打包时，即在应用打包阶段通过某种方式将配置（一般是文件形式）打入最终的应用包中。运行时，是指应用启动前并不知道具体的配置，而是在启动时，先从本地或者远程获取配置，然后再正常启动。
按配置的加载方式划分，可分为单次加载型配置和动态加载型配置。






## 参考

* <https://developer.51cto.com/art/202102/645471.htm>
* <https://zhuanlan.zhihu.com/p/286100357>
* <https://zhuanlan.zhihu.com/p/66051135>
* <https://github.com/kubernetes/kubernetes/issues/22368#issuecomment-421141188>
* <https://zhuanlan.zhihu.com/p/66051135>
* <https://blog.didispace.com/12factor-zh-cn/>
