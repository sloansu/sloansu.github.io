---
title: Android_HAL
tags:
  - Android
  - HAL
  - Hardware
categories:
  - Android
  - HAL层
date: 2018-06-11 19:45:48
---


> Any problem in computer science can be solved by anther layer of indirection.
> 在计算机科学领域，没有什么问题是加一个中间层所解决不了的。

## Android HAL定义

Android HAL[**Android Hard Abstract Layer**]，它向下屏蔽硬件驱动模块的实现细节，向上提供硬件访问服务。

Android 系统分为两层来支持硬件设备，其中一层在内核空间中实现（硬件驱动），另外一层在用户空间中实现（HAL）。而传统的Linux系统把对硬件的支持完全实现在内核空间中，即把对硬件的支持完全在硬件驱动模块实现。

<!-- more -->

## 1. Android HAL层存在的原因 ##

Android为什么不像传统的Linux一样，只采用硬件驱动层来实现对硬件设备的访问服务呢？

+ <strong>Linux内核源码是开源。</strong>因为硬件驱动层实现在Linux内核代码中，而Linux内核源码遵循GPL开源协议，如果我们在Android系统所使用的Linux内核中添加或修改了代码，那么必须将将其公开。这将损害移动设备厂商的利益。
+ <strong>尽管Android系统源码也是开源的，但它遵循Apache License协议。</strong> Apache License是一种对商业应用友好的许可，它允许移动设备厂商添加或修改Android系统源码，而不必公开这些代码。

而对于硬件设备的操作，只有在内核空间才能实现。所以折中的方案是，分别使用硬件驱动层和硬件抽象层来实现，其中硬件驱动层仅实现最简单的硬件访问方法，而在硬件抽象层中，实现具体的细节和参数设定。

## 2. Android HAL层的实现原理 ##

硬件抽象层的实现，依托了C/C++中的动态链接库的动态加载技术和指针类型强制转换技术。

动态链接库的动态加载技术，使得framework层和HAL实现了解耦，并能够分开进行编译。

### 2.1 动态链接库动态加载技术 ###

[动态链接库](http://tldp.org/HOWTO/Program-Library-HOWTO/dl-libraries.html#DLOPEN)专为软件系统的模块化而生。

动态链接库不能直接运行，也不能接受消息。它们是一些独立的文件，其中包含被可执行程序或其它动态库调用来完成某项工作的函数或是数据。只有在其它模块调用动态链接库中的函数，它才发挥作用。

动态链接库的加载方式有两种：

+ <strong>载入时动态链接</strong>，即编译的时候链接动态库，这样loader可以在程序启动的时候，将所有的动态链接映射到内存中；
+ <strong>运行时动态链接</strong>，在运行过程中，通过dlopen和dlfree的方式加载动态链接库，动态的将动态链接库加载到内存中。

两种方式的优劣：

1. 从编程角度来讲，第一种方式最方便，效率上影响也不大，在内存使用上有区别；第一种方式，一个库的代码，只要运行一次，便会占用物理内存，之后及时再也不使用，也会占用物理内存，直到进程的终止。
2. 第二种方式，库代码占用的内存可以使用<strong>dlfree的方式释放掉，返回给物理内存</strong>。

Android HAL层各设备模块的加载，使用的是运行时动态链接技术，

需要注意的是动态链接库，C实现API`<dlfcn.h>`，打开库，查找符号，处理错误，关闭库
头文件

	#inlcude <dlfcn.h>
	// 使用dlopen/dlsym/dlclose函数，动态地将动态库装在到当前进程实体中。
	int main(){
		void *handler = dlopen("libcount.so"， RTLD_LAZY);
		if(handler == NULL){
			cout << "ERROR: " << dlerror() << endl;
			return -1;
		}
		void (*inc)() = (void(*)())dlsym(handler, "inc");
		if(inc == NULL){
			cout << "ERROR: " << dlerror() << endl;
			return -1;
		}
		inc();
		int (*get)() = (int (*)())dlsym(handler, "get");
		if(get == NULL){
			cout << "ERROR: " << dlerror() << endl;
			return -1;
		}
		dlclose(handler);
	}
	
相关动态链接技术的详细文章，可参照[C/C++动态链接库]()（待完成...）。

### 2.2 指针类型的强制转换 ###

指针类型的强制转换，转换过程：子类指针->父类指针->子类指针（C）


## 3. Android HAL层具体的实现 ##

Android源码路径：

`android/hardware/libhardware/hardware.c`和`android/hardware/libhardware/include/hardware.h`

其中，hardware.h中定义了如下结构体和方法：

+ <strong>struct hw\_module\_t</strong>，定义了HAL层的硬件模块框架，所有硬件模块的父类。
+ <strong>struct hw\_module\_methods\_t</strong>，定义了模块结构体的方法框架，目前只定义了一个函数指针。
+ <strong>struct hw\_device\_t</strong>，定义了HAL层的硬件设备框架，所有硬件设备的父类。
+ <strong>int hw\_get\_module</strong>，对应模块库的动态加载函数。

hardware.c中，对hw\_get\_module方法进行了实现。

### 3.1 HAL接口的标准化方案 ###

所有的HAL模块必须继承hw\_module\_t框架，这里使用C的继承方法实现,即HAL模块框架定义的第一个成员变量，必须为`struct hw_module_t common`,后面可以拓展此模块的专有属性和方法。

HAL模块下的HAL设备框架必须继承hw\_device\_t，HAL设备框架的第一个属性值必须为`struct hw_device_t common`。

对于hw\_module\_methods\_t定义的方法:
`int (*open)(const struct hw_module_t* module, const char* id, struct hw_device_t** device);`,必须实现。

HAL模块的动态库中，必须初始化变量名为HAL_MODULE_INFO_SYM（"HMI"）的结构体。它作为动态库被加载后的导出符号。

一般来说，Android HAL层硬件模块的框架接口是由谷歌定义的，而硬件厂商只要实现相对应的接口文件即可。

### 3.2 模块Module接口 ###

	struct hw_module_t{
		uint32_t tag;
		uint16_t module_api_version;
		uint16_t hal_api_version
		const char* id;
		const char* name;
		const char* author;
		struct hw_module_methods_t* methods;
		void* dso;// 存handle，方便后面close。
	}

	// xxx硬件模块
	struct xxx_module_t{
		struct hw_module_t common;
		// 其他的特有模块属性和方法指针
		...
	}

### 3.2 设备Device接口 ###

	struct hw_device_t{
		uint32_t tag;
		uint32_t version;
		struct hw_module_t *module;
		int (*close)(struct hw_device_t* device);
	}
	
	// xxx硬件设备
	struct xxx_device_t{
		struct hw_device_t common;
		
		// 设备其他的属性及方法指针
	}

## 4. 案例 ##

camera HAL3实现。

定义接口文件：（android/hardware/libhardware/include/hardware/）camera3.h,camera_common.h。

其中，camera3.h中，定义了
	
	// 定义了camera流的类型模式
	typedef enum camera3_stream_type camera3_stream_type_t;
	// 定义了camera流的旋转方向
	typedef enum camera3_stream_rotation camera3_stream_rotation_t;
	// 定义了camera流的结构
	typedef struct camera3_stream camera3_stream_t;
	// 定义了camera流的配置信息,configure_streams()
	typedef struct camera3_stream_configuration camera3_stream_configuration_t;

	// 定义了camera buffer的状态
	typedef enum camera3_buffer_status camera3_buffer_status_t;
	// 定义了camera buffer的结构
	typedef struct camera3_stream_buffer camera3_stream_buffer_t；

	// 定义了由HAL到framework的callback的信息的类型：notify()回调
	typedef enum camera3_msg_type camera3_msg_type_t;
	// 定义camera_msg_error的错误码
	typedef enum camera3_error_msg_code camera3_error_msg_code_t;
	// 定义了拍摄指令信息的结构
	typedef struct camera3_shutter_msg camera3_shutter_msg_t;
	// 定义了拍摄指令信息的结构
	typedef struct camera3_notify_msg camera3_notify_msg_t;

	// 定义了request的模板
	typedef enum camera3_request_template camera3_request_template_t;
	// camera request的模板
	typedef struct camera3_capture_request camera3_capture_request_t;
	// camera result的模板
	typedef struct camera3_capture_result camera3_capture_result_t;
	// camera HAL到framework的回调

	typedef struct camera3_callback_ops camera3_callback_ops_t;
	// 定义了camera设备的执行动作
	typedef struct camera3_device_ops camera3_device_ops_t;
	// 定义了camera设备的结构
	ypedef struct camera3_device camera3_device_t;

camera_common.h定义了：

	// 定义了camera的元数据
	typedef struct camera_metadata camera_metadata_t;
	// 定义了描述camera的信息结构体
	typedef struct camera_info camera_info_t;
	// 定义了camera设备的状态模式
	typedef enum camera_device_status camera_device_status_t;
	// 定义闪光灯的状态模式
	typedef enum torch_mode_status torch_mode_status_t;
	// 定义了HAL到framework的状态改变回调
	typedef struct camera_module_callbacks camera_module_callbacks_t;
	typedef struct camera_module camera_module_t;

以上camera_module_t和camera_device_t中定义的方法，都需要在HAL中实现。

## 5. Android8.0后的HAL层 ##

Treble Project.

<hr>
## 参考文献 ##
   [1] [C++ dlopen mini-HOWTO](http://www.tldp.org/HOWTO/html_single/C++-dlopen/)
   
   [2] [Dynamically Loaded (DL) Libraries](http://tldp.org/HOWTO/Program-Library-HOWTO/dl-libraries.html#DLOPEN)

   [3] [老罗（罗升阳）博客](https://blog.csdn.net/Luoshengyang/article/details/6567257)

   [4] [Android源码](http://androidxref.com/7.1.1_r6/xref/hardware/)