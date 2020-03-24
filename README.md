1. # java-backend project 集成jenkins multijob插件完成串并行自动化部署


  ## 一、自行集成multijob、参数化build插件搭建jenkins步骤

  1. clone本项目
  2. 运行本项目中的jenkins.war(版本号: 2.176.1)包或者自行下载最新(注意: 使用root用户和普通user用户run的jenkins的workspace位置不一致)
     具体不一致可[参考](https://github.com/EugeneHuang9638/treadpit/wiki/summary#323-linux%E4%B8%8D%E5%90%8C%E7%94%A8%E6%88%B7%E8%BF%90%E8%A1%8Cjenkinswar), 运行命令 java -jar jenkins.war --httpPort=8050
  3. 访问localhost:8050按照页面指导设置用户名和密码, 选择`Install suggested plugins`安装jenkins建议的插件。
  4. 安装`multijob`插件
     登录`jenkins首页 -> Manage Jenkins -> Manage Plugins -> Availiable -> 搜索multijob -> 勾选并选择Download without start`。 确认有无下载成功  
     可查看jenkins控制台的log信息(使用nohup的方式启动或者无后台方式启动可查看到控制台信息)或者查看后台启动时指定重定向的log文件。
  5. 安装`Extended Choice Parameter和Github`插件(支持job参数化构建)，步骤同第四步
  6. 创建`multijob`, 配置各job之间的依赖关系（串并行）, 其中若有需要拉取git代码, 则还需要在job配置git的用户名和密码, 操作详情可自己百度。
  7. 运行job

  －－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－

  ## 二、mock一个case

  * 背景如下

    ```txt
    当clone java-backend项目之后会发现, 项目中的dynamic-proxy-adapter模块项目是会依赖与lib-common project的,
      所以打包前必须要先将lib-common install至本地仓库, 再打dynamic-proxy-adapter模块jar包, 这样才能在本地仓库中
      找到lib-common包完成依赖关系
    ```

  * 解决方案

    ```txt
      multijob插件的作用就是解决上述情况, 它可以设置job之间的执行方式(串并行)和触发机制
    ```

  ## 三、使用已搭建好的环境运行上述case步骤

  * 前置环境条件
    1. linux  
    2. jdk1.8  
    3. maven  
    4. 只提供job的folder, 需要自己将job拷贝至jenkins workspace目录下, 具体目录可[参考](https://github.com/EugeneHuang9638/treadpit/wiki/summary#323-linux%E4%B8%8D%E5%90%8C%E7%94%A8%E6%88%B7%E8%BF%90%E8%A1%8Cjenkinswar)
       随后执行 `Manage Jenkins -> Reload Configuration from Disk`

  * 执行步骤
    2. 输入自己定义的用户名和密码进入jenkins, 选择start-dynamic-proxy-adapter job, 点击build with paremeter, 设置参数`BRANCH=jenkins-for-dynamic-proxy` 点击build
    3. build成功后, 访问: localhost:8082/v1/test/sync-user


  ## 四、注意点

  2. 若项目时采取微服务架构, 所以可能存在多job同时运行的情况, 为了不增加机器的压力, 或者以后项目docker化时需要push镜像至远程
     的网络延迟问题时, 可以设置multijob最大的运行数.
     `jenkins首页 -> Manage Jenkins -> Configure System -> Maven Project Configuration -> of executors -> 自己配置自定义数字`

  3. 若jenkins打包maven项目时，提示 mvn命令找不到，则可以将mvn命令软连接到 `/usr/bin`目录下，此目录下的命令是所有用户都可以使用的。但在jenkins中执行`mvn -version`时，会报如下错

     ```java
     Error: Could not find or load main class org.codehaus.plexus.classworlds.launcher.Launcher
     ```

     解决方案是: 直接把mvn命令的全路径加上, eg: `/usr/local/maven/bin/mvn -version`  。或者在执行命令前执行`source /etc/profile`命令，保证当前终端能使用上环境变量 


  ## 五、若想自定义jenkins的主题颜色, 可查看以下链接

  * https://github.com/afonsof/jenkins-material-theme#installation  
    步骤很详细, 但注意替换颜色后的css的url路径为:   
    `http://afonsof.com/jenkins-material-theme/dist/material-{{your-color-name}}.css`  

    eg: 使用蓝色的样式: `http://afonsof.com/jenkins-material-theme/dist/material-light-blue.css`

  * 而不是上述url中内容所说的:  
    `https://cdn.rawgit.com/afonsof/jenkins-material-theme/gh-pages/dist/material-{{your-color-name}}.css`

  ## 六、Jenkins镜像化

  * 此镜像包含`multijob、Extended Choice Parameter`插件，可使用项目中提供的`jobs.tar.gz`压缩包来实现上述说的case，包含了**传递参数、按顺序**build的特性。内部安装了`git, jdk, maven ssh`等软件，支持日常工作部署需要。若jenkins需要将项目虚拟化(docker), 后续可以为镜像中安装docker环境。

  * 如何使用案例

    ```
    1. 将工作目录下的jobs.tar.gz解压到/root下, 即 tar -zxvf jobs.tar.gz -c /root
    此时root目录下会多出一个jobs的文件夹,
    2. 然后拉取镜像 docker pull registry.cn-hangzhou.aliyuncs.com/avengereug/jenkins:v2
    3. 启动镜像: docker run --name jenkins -it -d -v /root/jobs:/root/.jenkins/jobs --network host registry.cn-hangzhou.aliyuncs.com/avengereug/jenkins:v2
       (若需要将本地的maven仓库挂载可以添加参数:  -v  宿主机maven仓库repository目录:/root/.m2/repository)
    4. 访问jenkins:  http://ip:8050 
    5. 使用root/root登录
    6. 点击start-dynamic-proxy-adapter
    7. 点击左侧build with parameters, 分支名改成: jenkins-for-dynamic-proxy
    8. 点击启动
    
    注意: 
    1. 当再次重新走CI流程时，名为local-dynamic-proxy-adapter的job会报错，原因是端口被占用。此时可以在此job的配置的shell脚本的第一行添加如下脚本再重新run即可: 
    ps -ef | grep dynamic-proxy-adapter | grep -v grep | awk '{print $2}' | xargs kill -9
    
    2. 若jenkins需要在本机上部署项目，则启动容器时需要指定`network`类型为`host`，这样的话，容器中暴露出来的端口和宿主机的端口一致了，容器不需要再额外暴露了
    
    ```

* 如何查看jenkins镜像的工作目录，有如下两个路径

  1. 镜像内部的`/root/.jenkins/`

  2. 构建镜像时的`VOLUME`指令绑定了镜像内部的`/root/.jenkins/`路径，那究竟是宿主机的哪个目录与之挂在的呢？使用`docker inspect + 容器id或容器名称`命令来确认, eg: `docker inspect jenkins` 执行此命令后，会输出结构如下的信息:

     ```json
     [
         {
             "Id": "5d955a41b56325b04e04657a16c12743abc5505a76e6fe85dbc399e5b796753e",
             "Created": "2020-03-23T14:14:41.293507324Z",
             "Path": "java",
             "Args": [
                 "-jar",
                 "jenkins.war",
                 "--httpPort=8050"
             ],
             "State": {
                 "Status": "running",
                 "Running": true,
                 "Paused": false,
                 "Restarting": false,
                 "OOMKilled": false,
                 "Dead": false,
                 "Pid": 2430,
                 "ExitCode": 0,
                 "Error": "",
                 "StartedAt": "2020-03-23T14:14:46.480503585Z",
                 "FinishedAt": "0001-01-01T00:00:00Z"
             },
             "Image": "sha256:fce6824dd89f87eedc5e43e356e1b78a6143d54290c65c8c587c58507ac6b5a1",
             "ResolvConfPath": "/var/lib/docker/containers/5d955a41b56325b04e04657a16c12743abc5505a76e6fe85dbc399e5b796753e/resolv.conf",
             "HostnamePath": "/var/lib/docker/containers/5d955a41b56325b04e04657a16c12743abc5505a76e6fe85dbc399e5b796753e/hostname",
             "HostsPath": "/var/lib/docker/containers/5d955a41b56325b04e04657a16c12743abc5505a76e6fe85dbc399e5b796753e/hosts",
             "LogPath": "/var/lib/docker/containers/5d955a41b56325b04e04657a16c12743abc5505a76e6fe85dbc399e5b796753e/5d955a41b56325b04e04657a16c12743abc5505a76e6fe85dbc399e5b796753e-json.log",
             "Name": "/jenkins",
             "RestartCount": 0,
             "Driver": "overlay2",
             "Platform": "linux",
             "MountLabel": "",
             "ProcessLabel": "",
             "AppArmorProfile": "docker-default",
             "ExecIDs": null,
             "HostConfig": {
                 "Binds": [
                     "/home/eug/workspace/jenkins-java-backend/jobs:/root/.jenkins/jobs"
                 ],
                 "ContainerIDFile": "",
                 "LogConfig": {
                     "Type": "json-file",
                     "Config": {}
                 },
                 "NetworkMode": "host",
                 "PortBindings": {
                     "8050/tcp": [
                         {
                             "HostIp": "",
                             "HostPort": "8050"
                         }
                     ]
                 },
                 "RestartPolicy": {
                     "Name": "no",
                     "MaximumRetryCount": 0
                 },
                 "AutoRemove": false,
                 "VolumeDriver": "",
                 "VolumesFrom": null,
                 "CapAdd": null,
                 "CapDrop": null,
                 "Capabilities": null,
                 "Dns": [],
                 "DnsOptions": [],
                 "DnsSearch": [],
                 "ExtraHosts": null,
                 "GroupAdd": null,
                 "IpcMode": "private",
                 "Cgroup": "",
                 "Links": null,
                 "OomScoreAdj": 0,
                 "PidMode": "",
                 "Privileged": false,
                 "PublishAllPorts": false,
                 "ReadonlyRootfs": false,
                 "SecurityOpt": null,
                 "UTSMode": "",
                 "UsernsMode": "",
                 "ShmSize": 67108864,
                 "Runtime": "runc",
                 "ConsoleSize": [
                     0,
                     0
                 ],
                 "Isolation": "",
                 "CpuShares": 0,
                 "Memory": 0,
                 "NanoCpus": 0,
                 "CgroupParent": "",
                 "BlkioWeight": 0,
                 "BlkioWeightDevice": [],
                 "BlkioDeviceReadBps": null,
                 "BlkioDeviceWriteBps": null,
                 "BlkioDeviceReadIOps": null,
                 "BlkioDeviceWriteIOps": null,
                 "CpuPeriod": 0,
                 "CpuQuota": 0,
                 "CpuRealtimePeriod": 0,
                 "CpuRealtimeRuntime": 0,
                 "CpusetCpus": "",
                 "CpusetMems": "",
                 "Devices": [],
                 "DeviceCgroupRules": null,
                 "DeviceRequests": null,
                 "KernelMemory": 0,
                 "KernelMemoryTCP": 0,
                 "MemoryReservation": 0,
                 "MemorySwap": 0,
                 "MemorySwappiness": null,
                 "OomKillDisable": false,
                 "PidsLimit": null,
                 "Ulimits": null,
                 "CpuCount": 0,
                 "CpuPercent": 0,
                 "IOMaximumIOps": 0,
                 "IOMaximumBandwidth": 0,
                 "MaskedPaths": [
                     "/proc/asound",
                     "/proc/acpi",
                     "/proc/kcore",
                     "/proc/keys",
                     "/proc/latency_stats",
                     "/proc/timer_list",
                     "/proc/timer_stats",
                     "/proc/sched_debug",
                     "/proc/scsi",
                     "/sys/firmware"
                 ],
                 "ReadonlyPaths": [
                     "/proc/bus",
                     "/proc/fs",
                     "/proc/irq",
                     "/proc/sys",
                     "/proc/sysrq-trigger"
                 ]
             },
             "GraphDriver": {
                 "Data": {
                     "LowerDir": "/var/lib/docker/overlay2/ee1a97c8f344a32c01871c0eaec54d6d556141997776ffacef8758d19cf79a83-init/diff:/var/lib/docker/overlay2/b18b12027ec179983112befb6854c14be4940706591d58ecbbba12459da48c7d/diff:/var/lib/docker/overlay2/517a2ba8b65492a8924d0c918c312652cc9d51cb3305a3a0a96fcbf3b292c7c8/diff:/var/lib/docker/overlay2/187926888df517955161b17d3d8186f8819b98de05d9ca28b97732844c265b33/diff:/var/lib/docker/overlay2/634feff375df5b2c04cad0ae627c301d683e42451145b1d922527116f2bd298b/diff:/var/lib/docker/overlay2/06b35c7aac0be64296c3055059f664061963dfff8b3b6e23327909a1b644b797/diff:/var/lib/docker/overlay2/34221b4a597dfe5117cc042a638c13a3bf15a7ca3efead221522260208ac4ebc/diff:/var/lib/docker/overlay2/b9e017f2836b4e6f738d84351a24ffb24093a9d7d8b78a434919ff3e0e141287/diff:/var/lib/docker/overlay2/e8537efad3b1af2ebfe83ecf0c255152a8a2ff82e9bd54b76b478e215ffe28e5/diff:/var/lib/docker/overlay2/3bacca5cc70365acc8d2e6329dd6f0b873de0cb946a5929672d18199e57c5f4a/diff:/var/lib/docker/overlay2/66fd1ac299fb387ae0743a938452be4a6b6612bfad0a85065faf6dde0c2fb78d/diff",
                     "MergedDir": "/var/lib/docker/overlay2/ee1a97c8f344a32c01871c0eaec54d6d556141997776ffacef8758d19cf79a83/merged",
                     "UpperDir": "/var/lib/docker/overlay2/ee1a97c8f344a32c01871c0eaec54d6d556141997776ffacef8758d19cf79a83/diff",
                     "WorkDir": "/var/lib/docker/overlay2/ee1a97c8f344a32c01871c0eaec54d6d556141997776ffacef8758d19cf79a83/work"
                 },
                 "Name": "overlay2"
             },
             "Mounts": [
                 {
                     "Type": "bind",
                     "Source": "/home/eug/workspace/jenkins-java-backend/jobs",
                     "Destination": "/root/.jenkins/jobs",
                     "Mode": "",
                     "RW": true,
                     "Propagation": "rprivate"
                 },
                 {
                     "Type": "volume",
                     "Name": "daf861dedd5ff437d1db318c329cc36258598c7fbec9a62111b1dd0be4252227",
                     "Source": "/var/lib/docker/volumes/daf861dedd5ff437d1db318c329cc36258598c7fbec9a62111b1dd0be4252227/_data",
                     "Destination": "/root/.jenkins",
                     "Driver": "local",
                     "Mode": "",
                     "RW": true,
                     "Propagation": ""
                 }
             ],
             "Config": {
                 "Hostname": "ubuntu",
                 "Domainname": "",
                 "User": "",
                 "AttachStdin": false,
                 "AttachStdout": false,
                 "AttachStderr": false,
                 "ExposedPorts": {
                     "8050/tcp": {}
                 },
                 "Tty": true,
                 "OpenStdin": true,
                 "StdinOnce": false,
                 "Env": [
                     "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/local/jdk/bin:/usr/local/maven/bin",
                     "JAVA_HOME=/usr/local/jdk",
                     "MAVEN_HOME=/usr/local/maven"
                 ],
                 "Cmd": null,
                 "Image": "registry.cn-hangzhou.aliyuncs.com/avengereug/jenkins:v2",
                 "Volumes": {
                     "/root/.jenkins": {}
                 },
                 "WorkingDir": "/root",
                 "Entrypoint": [
                     "java",
                     "-jar",
                     "jenkins.war",
                     "--httpPort=8050"
                 ],
                 "OnBuild": null,
                 "Labels": {
                     "MAINTAINER": "avengerEug",
                     "org.label-schema.build-date": "20200114",
                     "org.label-schema.license": "GPLv2",
                     "org.label-schema.name": "CentOS Base Image",
                     "org.label-schema.schema-version": "1.0",
                     "org.label-schema.vendor": "CentOS",
                     "org.opencontainers.image.created": "2020-01-14 00:00:00-08:00",
                     "org.opencontainers.image.licenses": "GPL-2.0-only",
                     "org.opencontainers.image.title": "CentOS Base Image",
                     "org.opencontainers.image.vendor": "CentOS"
                 }
             },
             "NetworkSettings": {
                 "Bridge": "",
                 "SandboxID": "1c419efa93aead7739f5ff9a062c12572109badcb41cce3eb9e1187ac4075c49",
                 "HairpinMode": false,
                 "LinkLocalIPv6Address": "",
                 "LinkLocalIPv6PrefixLen": 0,
                 "Ports": {},
                 "SandboxKey": "/var/run/docker/netns/default",
                 "SecondaryIPAddresses": null,
                 "SecondaryIPv6Addresses": null,
                 "EndpointID": "",
                 "Gateway": "",
                 "GlobalIPv6Address": "",
                 "GlobalIPv6PrefixLen": 0,
                 "IPAddress": "",
                 "IPPrefixLen": 0,
                 "IPv6Gateway": "",
                 "MacAddress": "",
                 "Networks": {
                     "host": {
                         "IPAMConfig": null,
                         "Links": null,
                         "Aliases": null,
                         "NetworkID": "648c0afff000e653fad1a5eac5d32dffdd6e6f1808a6db0eed2c999a3978e2c2",
                         "EndpointID": "71e5f186ea3a6f7eb5ecf6e4c394fdeb48b97db5bcfb628e39a15cf474606ea4",
                         "Gateway": "",
                         "IPAddress": "",
                         "IPPrefixLen": 0,
                         "IPv6Gateway": "",
                         "GlobalIPv6Address": "",
                         "GlobalIPv6PrefixLen": 0,
                         "MacAddress": "",
                         "DriverOpts": null
                     }
                 }
             }
         }
     ]
     ```

     我们找到`Mounts`节点，如下

     ```json
     "Mounts": [
                 {
                     "Type": "volume",
                     "Name": "daf861dedd5ff437d1db318c329cc36258598c7fbec9a62111b1dd0be4252227",
                     "Source": "/var/lib/docker/volumes/daf861dedd5ff437d1db318c329cc36258598c7fbec9a62111b1dd0be4252227/_data",
                     "Destination": "/root/.jenkins",
                     "Driver": "local",
                     "Mode": "",
                     "RW": true,
                     "Propagation": ""
                 }
        ......
     ```

     由如上可知，宿主机的`"/var/lib/docker/volumes/daf861dedd5ff437d1db318c329cc36258598c7fbec9a62111b1dd0be4252227/_data"`目录与容器的`/root/.jenkins`目录相互挂载，而`/root/.jenkins`就是jenkins的工作目录

  ## 七、Dockerfile
  ```Dockerfile
    FROM centos
    LABEL MAINTAINER=avengerEug
    ADD apache-maven-3.3.9-bin.tar.gz /usr/local/
    RUN mv /usr/local/apache-maven-3.3.9 /usr/local/maven

    ADD jdk-8u221-linux-x64.tar.gz  /usr/local/
    RUN mv /usr/local/jdk1.8.0_221 /usr/local/jdk

    ENV JAVA_HOME=/usr/local/jdk
    ENV MAVEN_HOME=/usr/local/maven
    ENV PATH=$PATH:$JAVA_HOME/bin:$MAVEN_HOME/bin

    COPY .jenkins /root/.jenkins/
    COPY ./jenkins/jenkins.war /root/

    RUN yum install -y git

    VOLUME "/root/.jenkins"

    RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
    RUN echo 'Asia/Shanghai' >/etc/timezone

    EXPOSE 8050

    WORKDIR /root

    ENTRYPOINT ["java", "-jar", "jenkins.war", "--httpPort=8050"]
  ```
