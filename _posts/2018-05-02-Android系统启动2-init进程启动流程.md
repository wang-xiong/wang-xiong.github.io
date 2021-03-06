---
layout: post
title: "「Android系统启动」二、init进程启动流程"
subtitle: 'Android系统启动过程学习'
date:       2018-05-02
author: "Wangxiong"
header-style: text
tags:
  - Android
  - Android系统启动
---

## 1.init进程简介

init进程是Android系统中用户空间的第一个进程，主要负责创建Zygote(孵化器)和属性服务等功能。源码位于system/core/init下。

## 2.Android系统启动的流程

1. 启动电源以及系统启动

   按下电源，引导芯片代码从预定义的地方开始执行（固化在ROM），加载引导程序到Bootloader到RAM，然后执行。

2. 引导程序Bootloader

   引导程序是在Android系统开始运行前的一个小程序，主要作用就是把系统OS拉起来并运行。

3. Linux内核启动

   内核启动，设置缓存，被包含存储器，计划列表，加载驱动。当内核完成系统设置，它首先会在系统中寻找init文件，然后启动root进程或者系统的第一个进程init。

4. init进程的启动

## 3.init入口函数

init的入口函数为main，代码在system/core/init.cpp文件中。

```c
int main(int argc, char** argv) {
    if (!strcmp(basename(argv[0]), "ueventd")) {
        return ueventd_main(argc, argv);
    }
    if (!strcmp(basename(argv[0]), "watchdogd")) {
        return watchdogd_main(argc, argv);
    }
    umask(0);
    add_environment("PATH", _PATH_DEFPATH);
    bool is_first_stage = (argc == 1) || (strcmp(argv[1], "--second-stage") != 0);
    //创建文件并挂载
    if (is_first_stage) {
        mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
        mkdir("/dev/pts", 0755);
        mkdir("/dev/socket", 0755);
        mount("devpts", "/dev/pts", "devpts", 0, NULL);
        #define MAKE_STR(x) __STRING(x)
        mount("proc", "/proc", "proc", 0, "hidepid=2,gid=" MAKE_STR(AID_READPROC));
        mount("sysfs", "/sys", "sysfs", 0, NULL);
    }
    open_devnull_stdio();
    klog_init();
    klog_set_level(KLOG_NOTICE_LEVEL);
    NOTICE("init %s started!\n", is_first_stage ? "first stage" : "second stage");
    if (!is_first_stage) {
        // Indicate that booting is in progress to background fw loaders, etc.
        close(open("/dev/.booting", O_WRONLY | O_CREAT | O_CLOEXEC, 0000));
        //1.初始化属性相关资源
        property_init();
        process_kernel_dt();
        process_kernel_cmdline();
        export_kernel_boot_props();
    }
 ...
    //2.启动属性服务
    start_property_service();
    const BuiltinFunctionMap function_map;
    Action::set_function_map(&function_map);
    Parser& parser = Parser::GetInstance();
    parser.AddSectionParser("service",std::make_unique<ServiceParser>());
    parser.AddSectionParser("on", std::make_unique<ActionParser>());
    parser.AddSectionParser("import", std::make_unique<ImportParser>());
    //解析init.rc配置文件
    parser.ParseConfig("/init.rc");//3
   ...   
       while (true) {
        if (!waiting_for_exec) {
            am.ExecuteOneCommand();
            restart_processes();
        }
        int timeout = -1;
        if (process_needs_restart) {
            timeout = (process_needs_restart - gettime()) * 1000;
            if (timeout < 0)
                timeout = 0;
        }
        if (am.HasMoreCommands()) {
            timeout = 0;
        }
        bootchart_sample(&timeout);
        epoll_event ev;
        int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, timeout));
        if (nr == -1) {
            ERROR("epoll_wait failed: %s\n", strerror(errno));
        } else if (nr == 1) {
            ((void (*)()) ev.data.ptr)();
        }
    }
    return 0;
}
```

核心地方：调用property_init初始化属性，调用start_property_service启动属性服务，调用parser.ParseConfig(“/init.rc”)解析init.rc。

## 4.init.rc

init.rc是一个配置文件，文件位置:system/core/rootdir/init.rc。是由Android初始化语言编写的脚本，主要包含五种类型语句：Action、Commands、Services、Options和Import。

```c
on init
    sysclktz 0
    # Mix device-specific information into the entropy pool
    copy /proc/cmdline /dev/urandom
    copy /default.prop /dev/urandom
...

on boot
    # basic network init
    ifup lo
    hostname localhost
    domainname localdomain
    # set RLIMIT_NICE to allow priorities from 19 to -20
    setrlimit 13 40 40
...
```

部分代码如上所示，其中#是注释符号。on init和on boot是Action类型语句，它的格式为：

```c
on <trigger> [&& <trigger>]*     //设置触发器      
<command>      
<command>      //动作触发之后要执行的命令`
```

Services类型语句

```c
service <name> <pathname> [ <argument> ]*   //<service的名字><执行程序路径><传递参数>  
   <option>       //option是service的修饰词，影响什么时候、如何启动services  
   <option>  
   ...
```

Android7.0中对init.rc进行了拆分，每个服务对应一个rc文件。其中zygote服务的启动脚本在init.zygoteXX.rc文件中，64位处理器对应init.zygote64.rc。文件位于system/core/rootdir/init.zygote64.rc。

```c
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    writepid /dev/cpuset/foreground/tasks /dev/stune/foreground/tasks
```

根据service语句可知，进程名为zygote，执行程序路径为/system/bin/app_process64，传递参数-Xzygote /system/bin --zygote --start-system-server。clas main指明zygote的class name为main。

## 5.解析service过程

如何解析service的rc文件，需要用到两个函数，ParseSection和ParseLinSection，ParseSection函数主要用来搭建service的架子，ParseLinSec用来解析子项。代码位置：*system/core/init/service.cpp*

```c
bool ServiceParser::ParseSection(const std::vector<std::string>& args,
                                 std::string* err) {
    if (args.size() < 3) {
        *err = "services must have a name and a program";
        return false;
    }
    const std::string& name = args[1];
    if (!IsValidName(name)) {
        *err = StringPrintf("invalid service name '%s'", name.c_str());
        return false;
    }
    std::vector<std::string> str_args(args.begin() + 2, args.end());
    //1.根据参数，构造出一个service对象，它的classname为”default”
    service_ = std::make_unique<Service>(name, "default", str_args);
    return true;
}

bool ServiceParser::ParseLineSection(const std::vector<std::string>& args,
                                     const std::string& filename, int line,
                                     std::string* err) const {
    return service_ ? service_->HandleLine(args, err) : false;
}
```

当解析完毕时会调用EndSection：

```c
void ServiceParser::EndSection() {
    if (service_) {
        ServiceManager::GetInstance().AddService(std::move(service_));
    }
}
```

```c
void ServiceManager::AddService(std::unique_ptr<Service> service) {
    Service* old_service = FindServiceByName(service->name());
    if (old_service) {
        ERROR("ignored duplicate definition of service '%s'",
              service->name().c_str());
        return;
    }
    //1.将service对象添加到services链表中
    services_.emplace_back(std::move(service));
}
```

解析service的过程就是根据参数创建出service对象，然后根据选项域的内容填充service对象，最后将service对象加到vector类型的services链表中。

## 6.init启动zygote

如何启动zygote service，zygote启动脚本中zygote的class name为main，在init.rc文件中

```c
...
on nonencrypted    
    # A/B update verifier that marks a successful boot.  
    exec - root -- /system/bin/update_verifier nonencrypted  
    class_start main         
    class_start late_start 
...
```

class_start使一个COMMAND语句，对应函数为do_class_start，main就是zygote，do_class_start函数代码在builtins.cpp中，位置*system/core/init/builtins.cpp*

```c
static int do_class_start(const std::vector<std::string>& args) {
        /* Starting a class does not start services
         * which are explicitly disabled.  They must
         * be started individually.
         */
    ServiceManager::GetInstance().
        ForEachServiceInClass(args[1], [] (Service* s) { s->StartIfNotDisabled(); });
    return 0;
}
```

StartIfNotDisabled在*system/core/init/service.cpp*

```c
bool Service::StartIfNotDisabled() {
    if (!(flags_ & SVC_DISABLED)) {
        return Start();
    } else {
        flags_ |= SVC_DISABLED_START;
    }
    return true;
}
```

```c
bool Service::Start() {     
    flags_ &= (~(SVC_DISABLED|SVC_RESTARTING|SVC_RESET|SVC_RESTART|SVC_DISABLED_START));     time_started_ = 0;     
    if (flags_ & SVC_RUNNING) {
        //如果Service已经运行，则不启动         
        return false;     
    }     
    bool needs_console = (flags_ & SVC_CONSOLE);     
    if (needs_console && !have_console) {         
        ERROR("service '%s' requires console\n", name_.c_str());         
        flags_ |= SVC_DISABLED;         
        return false;     
    }   
    //判断需要启动的Service的对应的执行文件是否存在，不存在则不启动该Service     
    struct stat sb;     
    if (stat(args_[0].c_str(), &sb) == -1) {         
        ERROR("cannot find '%s' (%s), disabling '%s'\n", args_[0].c_str(), strerror(errno), name_.c_str());         
        flags_ |= SVC_DISABLED;         
        return false;     
    }  
    ...     
    pid_t pid = fork();
    //1.fork函数创建子进程     
    if (pid == 0) {//运行在子进程中         
        umask(077);         
        for (const auto& ei : envvars_) {             
            add_environment(ei.name.c_str(), ei.value.c_str());         
        }         
        for (const auto& si : sockets_) {             
            int socket_type = ((si.type == "stream" ? SOCK_STREAM :                           (si.type == "dgram" ? SOCK_DGRAM : SOCK_SEQPACKET)));             
            const char* socketcon = !si.socketcon.empty() ? si.socketcon.c_str() : scon.c_str();              
            int s = create_socket(si.name.c_str(), socket_type, si.perm,                                  si.uid, si.gid, socketcon);             
            if (s >= 0) {                 
                PublishSocket(si.name, s);             
            }         
        } 
        ...         
        //2.通过execve执行程序         
        if (execve(args_[0].c_str(), (char**) &strs[0], (char**) ENV) < 0) {            
            ERROR("cannot execve('%s'): %s\n", args_[0].c_str(), strerror(errno));         
        }          
        _exit(127);     
    } 
    ...     
    return true; 
}
```

调用fork函数创建子进程，并在子进程中调用execve执行system/bin/app_process，进入app_main.cpp的main函数，位置*frameworks/base/cmds/app_process/app_main.cpp*

```c
int main(int argc, char* const argv[])
{
    ...
    if (zygote) {
        //1.调用runtime的start方法启动zygote。
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
        return 10;
    }
}
```

## 7.属性服务

类似于Windows平台的注册表管理器，Android提供了属性服务，在init.cpp代码中，*system/core/init/init.cpp*，

```c
//初始化属性服务
property_init();
...
//启动属性服务
start_property_service();
```

property_init函数在*system/core/init/property_service.cpp*，__system_property_area_init用于初始化属性内存区域。

```c
void property_init() {
    if (__system_property_area_init()) {
        ERROR("Failed to initialize property area\n");
        exit(1);
    }
}
```

start_property_service函数

```c
void start_property_service() {
    //1.创建非阻塞的socket
    property_set_fd = create_socket(PROP_SERVICE_NAME, SOCK_STREAM | SOCK_CLOEXEC | SOCK_NONBLOCK, 0666, 0, 0, NULL);
    if (property_set_fd == -1) {
        ERROR("start_property_service socket creation failed: %s\n", strerror(errno));
        exit(1);
    }
    //2.调用listen函数对property_set_fd进行监听
    listen(property_set_fd, 8);
    //3.将property_set_fd放入了epoll句柄中，用epoll来监听property_set_fd：当property_set_fd中有数据到来时，init进程将用handle_property_set_fd函数进行处理。
    register_epoll_handler(property_set_fd, handle_property_set_fd);
}
```

属性服务处理请求

从上文得知，属性服务收到客户端的请求时，调用handle_property_set_fd函数进行处理，代码在*system/core/init/property_service.cpp*

```c
static void handle_property_set_fd() {  
    ...
    if(memcmp(msg.name,"ctl.",4) == 0) {
        close(s);
        if (check_control_mac_perms(msg.value, source_ctx, &cr)) {
            handle_control_message((char*) msg.name + 4, (char*) msg.value);
        } else {
            ERROR("sys_prop: Unable to %s service ctl [%s] uid:%d gid:%d pid:%d\n",
                        msg.name + 4, msg.value, cr.uid, cr.gid, cr.pid);
        }
    } else {
        //1.检查客户端进程权限
        if (check_mac_perms(msg.name, source_ctx, &cr)) {
            //2.调用property_set函数对属性进行修改。
            property_set((char*) msg.name, (char*) msg.value);
        } else {
            ERROR("sys_prop: permission denied uid:%d  name:%s\n",
                      cr.uid, msg.name);
        }
        close(s);
     }
     freecon(source_ctx);
     break;
  default:
     close(s);
     break;
  }
}
```

```c
int property_set(const char* name, const char* value) {
    int rc = property_set_impl(name, value);
    if (rc == -1) {
        ERROR("property_set(\"%s\", \"%s\") failed\n", name, value);
    }
    return rc;
}
```

property_set_impl函数主要用来对属性进行修改，并对以ro、net和persist开头的属性进行相应的处理。

```c
static int property_set_impl(const char* name, const char* value) {
    size_t namelen = strlen(name);
    size_t valuelen = strlen(value);
    if (!is_legal_property_name(name, namelen)) return -1;
    if (valuelen >= PROP_VALUE_MAX) return -1;
    if (strcmp("selinux.reload_policy", name) == 0 && strcmp("1", value) == 0) {
        if (selinux_reload_policy() != 0) {
            ERROR("Failed to reload policy\n");
        }
    } else if (strcmp("selinux.restorecon_recursive", name) == 0 && valuelen > 0) {
        if (restorecon_recursive(value) != 0) {
            ERROR("Failed to restorecon_recursive %s\n", value);
        }
    }
    //从属性存储空间查找该属性
    prop_info* pi = (prop_info*) __system_property_find(name);
    //如果属性存在
    if(pi != 0) {
        //如果属性以"ro."开头，则表示是只读，不能修改，直接返回
        if(!strncmp(name, "ro.", 3)) return -1;
        //更新属性值
        __system_property_update(pi, value, valuelen);
    } else {
        //如果属性不存在则添加该属性
        int rc = __system_property_add(name, namelen, value, valuelen);
        if (rc < 0) {
            return rc;
        }
    }
    /* If name starts with "net." treat as a DNS property. */
    if (strncmp("net.", name, strlen("net.")) == 0)  {
        if (strcmp("net.change", name) == 0) {
            return 0;
        }
        //以net.开头的属性名称更新后，需要将属性名称写入net.change中  
        property_set("net.change", name);
    } else if (persistent_properties_loaded &&
            strncmp("persist.", name, strlen("persist.")) == 0) {
        /*
         * Don't write properties to disk until after we have read all default properties
         * to prevent them from being overwritten by default values.
         */
        write_persistent_property(name, value);
    }
    property_changed(name, value);
    return 0;
}
```

## 8.init进程总结

init进程主要做了三件事：

1.创建一些文件夹并挂载设备。

2.初始化和启动属性服务。

3.解析init.rc配置并启动zygote进程。