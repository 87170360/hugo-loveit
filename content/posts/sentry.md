---
title: "接入Sentry那些事"
date: 2023-03-24T15:56:44+08:00
draft: false
tags: ["sentry"]
series: ["技术"]
categories: ["工作"]
---

### sentry解决了什么问题
场景：客户端程序崩溃了，没有任何调试信息，无法排查问题。
sentry: 在客户端崩溃时候，捕获程序运行堆栈，并且把堆栈上传.可以在控制后台查看崩溃堆栈信息.
适用范围: windows, mac os, linux

### c/s架构
sentry 分后台管理和前端接入两块

### 后台管理搭建
[教程](https://theappsguy.dev/setting-up-sentry-self-hosted)


### 前端接入

#### [下载sentry源码 ](https://github.com/getsentry/sentry-native)
下载源码，把源码集成到你的项目中，这个步即繁琐有跟项目密切相关，就跳过了

#### [SDK文档](https://docs.sentry.io/platforms/native/)

#### 组件介绍crashpad breakpad
捕获崩溃核心功能由由组件crashpad和breakpad实现, 这两个都是google公司开源代码。
sentry包装了这两个组件，科技以套壳为本。
这两个组件区别


### 客户端接入代码, 假定你已经完成了接入SDK
``` C++
	//在main函数启动马上调用
	void InitSentry() {
		sentry_options_t* options = sentry_options_new();
		sentry_options_set_dsn(options, "http://改成你自己的");
		sentry_options_set_release(options, version.c_str());
		sentry_options_set_debug(options, 1);
		sentry_options_set_environment(options, "production");

		sentry_options_set_database_pathw(options, "设置一个目录存放dump临时文件");

		sentry_options_set_handler_pathw(options, "指定crashpad_handler.exe绝对路径");

		//sentry_options_set_shutdown_timeout(options, 30000);
		sentry_set_level(SENTRY_LEVEL_DEBUG);
		sentry_options_set_logger(options, sentry_log_handler, nullptr);

		sentry_options_set_auto_session_tracking(options, 1);

		sentry_options_add_attachmentw(options, "可以指定一个上传文件");


		int retInit = sentry_init(options);
		LOG_INFO("sentry_inti result :%d", retInit);

		sentry_value_t user = sentry_value_new_object();
		sentry_value_set_by_key(user, "id", sentry_value_new_string(deviceID.c_str()));
		sentry_value_set_by_key(user, "ip_address", sentry_value_new_string("{{auto}}"));
		sentry_set_user(user);

	}

	//在程序退出前调用
	void CloseSentry() {
		sentry_close();
	}

```

