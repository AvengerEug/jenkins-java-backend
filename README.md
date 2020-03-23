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

  
