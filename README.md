# java-backend project 集成jenkins multijob插件完成串并行自动化部署


## 自行集成multijob、参数化build插件搭建jenkins步骤
1. clone本项目
2. 运行本项目中的jenkins.war(版本号: 2.176.1)包或者自行下载最新(注意: 使用root用户和普通user用户run的jenkins的workspace位置不一致)
   具体不一致可[参考](https://github.com/EugeneHuang9638/treadpit/wiki/summary#323-linux%E4%B8%8D%E5%90%8C%E7%94%A8%E6%88%B7%E8%BF%90%E8%A1%8Cjenkinswar), 运行命令 java -jar jenkins.war --httpPort=8050
3. 访问localhost:8050按照页面指导设置用户名和密码, 选择`Install suggested plugins`安装jenkins建议的插件。
4. 安装`multijob`插件
   登录`jenkins首页 -> Manage Jenkins -> Manage Plugins -> Availiable -> 搜索multijob -> 勾选并选择Download without start`。 确认有无下载成功  
   可查看jenkins控制台的log信息(使用nohup的方式启动或者无后台方式启动可查看到控制台信息)或者查看后台启动时指定重定向的log文件。
5. 安装`Extended Choice Parameter`插件(支持job参数化构建)，步骤同第四步
6. 创建`multijob`, 配置各job之间的依赖关系（串并行）, 其中若有需要拉取git代码, 则还需要在job配置git的用户名和密码, 操作详情可自己百度。
7. 运行job

－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－

## mock一个case
* 背景如下
  ```
    当clone java-backend项目之后会发现, 项目中的dynamic-proxy-adapter模块项目是会依赖与lib-common project的,
    所以打包前必须要先将lib-common install至本地仓库, 再打dynamic-proxy-adapter模块jar包, 这样才能在本地仓库中
    找到lib-common包完成依赖关系
  ```

* 解决方案
  ```
    multijob插件的作用就是解决上述情况, 它可以设置job之间的执行方式(串并行)和触发机制
  ```
## 使用已搭建好的环境运行上述case步骤
* 前置环境条件
  1. linux  
  2. jdk1.8  
  3. maven  
  4. 只提供job的folder, 需要自己将job拷贝至jenkins workspace目录下, 具体目录可[参考](https://github.com/EugeneHuang9638/treadpit/wiki/summary#323-linux%E4%B8%8D%E5%90%8C%E7%94%A8%E6%88%B7%E8%BF%90%E8%A1%8Cjenkinswar)
     随后执行 `Manage Jenkins -> Reload Configuration from Disk`
  
* 执行步骤
  1. 需要先走[下述注意点的第一个步骤](https://github.com/EugeneHuang9638/jenkins-java-backend#%E6%B3%A8%E6%84%8F%E7%82%B9)
  2. 输入自己定义的用户名和密码进入jenkins, 选择start-dynamic-proxy-adapter job, 点击build with paremeter, 设置参数BRANCH=develop 点击build
  3. build成功后, 访问: localhost:8081/v1/test/sync-user


## 注意点
1. 因为java-backend project中有存在依赖第三方jar包(在maven仓库中找不到)的模块, 所以在运行mvn install时会报错
   解决方案: 找任一位置拉取该[project](https://github.com/EugeneHuang9638/dynamic-proxy-adapter)代码, 并进入thirdparty文件夹,
   执行mvn-install.sh脚本
2. 若项目时采取微服务架构, 所以可能存在多job同时运行的情况, 为了不增加机器的压力, 或者以后项目docker化时需要push镜像至远程
　 的网络延迟问题时, 可以设置multijob最大的运行数.
   `jenkins首页 -> Manage Jenkins -> Configure System -> Maven Project Configuration -> of executors -> 自己配置自定义数字`


## 若想自定义jenkins的主题颜色, 可查看以下链接
* https://github.com/afonsof/jenkins-material-theme#installation  
  步骤很详细, 但注意替换颜色后的css的url路径为:   
  `http://afonsof.com/jenkins-material-theme/dist/material-{{your-color-name}}.css`  
  而不是上述url中内容所说的:  
  `https://cdn.rawgit.com/afonsof/jenkins-material-theme/gh-pages/dist/material-{{your-color-name}}.css`

