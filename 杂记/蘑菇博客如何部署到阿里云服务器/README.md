# 蘑菇博客如何部署到阿里云服务器

## 前言

最近很多小伙伴问我如何把蘑菇博客部署在自己的云服务器中，今天花费了一些时间整理一下博客部署的过程，希望能够对大家有所帮助，好了话不多说，下面开始进入正式的部署过程。

如果小伙伴还没有安装好对应的环境话，需要按这篇博客搭建好环境：[使用Docker快速搭建蘑菇博客](http://moguit.cn/#/info?blogUid=ab8377106a0d4b9f8d66131e4312c69e)

本文不再重复叙述 nginx、rabbitmq、mysql、solr以及redis的安装和启动，如果了解，请移步至上一篇博客~

## SpringBoot项目打包

首先我们需要将项目拉取下来，然后进入到mogu_blog_v2目录

```bash
# 拉取项目
git clone https://gitee.com/moxi159753/mogu_blog_v2.git
# 进入mogu_blog_v2目录
cd mogu_blog_v2
# 执行mvn打包命令
mvn clean install
```

完成上面操作后，能看到下面的图，说明已经成功打包了

![image-20200101130036478](images/image-20200101130036478.png)

下面我们需要进入下列的目录，把对应的jar上传到我们之前制作的docker容器中

```
# 进入 mogu_eureka目录
cd mogu_eureka\target
```

我们把下面的jar包复制

![image-20200101131008048](images/image-20200101131008048.png)

然后通过xftp工具，把jar复制到Docker容器的/home/mogu_blog/mogu_eureka目录，替换里面的jar包

```bash
cd /home/mogu_blog/mogu_eureka
```

![image-20200101131046409](images/image-20200101131046409.png)

里面的结构有：

```
./startup.sh  #启动脚本
./shutdown.sh #关闭脚本
mogu_eureka***.jar #springboot打包的可执行jar包
/config #外部配置文件
catalina.out #启动脚本后，生成的日志文件
```

然后我们执行里面的start.sh脚本启动jar包

```
./startup.sh
```

startup.sh脚本其实比较简单，代码如下

```
#!/bin/bash     
nohup java  -Xms256m -Xmx512m -jar  mogu_eureka-0.0.1-SNAPSHOT.jar  > catalina.out  2>&1 &
tail -f ./catalina.out
```

就是通过nohup命令，进行后台启动 java jar包，同时里面-Xms256m -Xmx512m 是指定最大堆和最小堆内存，这个命令就是为了防止tomcat请求太多内存，而造成其它项目无法启动。

shutdown.sh脚本如下所示：

```
#!/bin/bash
PID=$(ps -ef | grep mogu_eureka-0.0.1-SNAPSHOT.jar  | grep -v grep | awk '{ print $2 }')
if [ -z "$PID" ]
then
    echo Application is already stopped
else
    echo kill $PID
    kill $PID
fi
```

大概意思就是，我们找到jar启动的PID，然后使用kill命令杀死即可。

启动mogu_eureka后，我们重复上面的操作，分别把 mogu_picture、mogu_sms、mogu_web、mogu_admin分别执行上述操作，即可完成蘑菇博客后台项目的部署

需要注意的是：mogu_picture项目和mogu_web项目，除了替换对应的jar包外，我们还需要修改对应的配置文件，然后在启动项目

### mogu_picture修改配置

首先我们在启动startup.sh脚本前，先修改对应目录下 config文件夹的application.yml配置文件，直接修改可能会里面是乱码，可以先把它取出来，到window下修改，然后在放入

```
# 进入picture目录下
cd /home/mogu_blog/mogu_picture/config
# 编辑配置
vim application.yml
```

然后找到下面的内容

```yaml
#Data image url
data:
  image:
    url: http://your_ip:8600/
file:
  upload:
    path: /home/mogu_blog/mogu_data/
```

特别注意的是：如果有的小伙伴自己有域名的话，并且已经配置了域名解析，即可换成对应的域名

例如：我的 picture.moguit.cn域名，映射到服务器的8600端口，那么我的配置就可以改成如下所示

```yaml
#Data image url
data:
  image:
    url: http://picture.moguit.cn/
file:
  upload:
    path: /home/mogu_blog/mogu_data/
```

修改完成后，我们启动picture项目

### 修改mogu_web配置文件

同理，找到mogu_web下的config文件，我们需要将下面的域名，改成自己的IP地址

```yaml
#Data image url
data:
  # 门户页面
  webSite:
    url: http://101.132.122.175:9527/#/
  # mogu_web网址，用于第三方登录回调
  web:
    url: http://101.132.122.175:8603
  # 静态资源映射，通过nginx
  image:
    url: http://101.132.122.175:8600/
```

如果拥有域名的小伙伴，可以将配置改成如下形式

```yaml
#Data image url
data:
  # 门户页面
  webSite:
    url: http://www.moguit.cn:9527/#/
  # mogu_web网址，用于第三方登录回调
  web:
    url: http://101.132.122.175:8603
  # 静态资源映射，通过nginx
  image:
    url: http://picture.moguit.cn/
```

同时在配置文件的最下面，还需要修改第三方注册需要的 clientId 和 ClientSecret：如果不清楚如何获取的小伙伴，可以查看我的这篇博客，在后面部分对ID的获取有相关介绍：[SpringBoot+Vue如何集成第三方登录JustAuth](http://moguit.cn/#/info?blogUid=8cbadb54967257f12d6cc7eb1a58a361)

```yaml

# 第三方登录
justAuth:
  clientId:
    gitee: XXXXXXXXXXXXXXXXXXXXXX
    github: XXXXXXXXXXXXXXXXXXXXXX
  clientSecret:
    gitee: XXXXXXXXXXXXXXXXXXXXXX
    github: XXXXXXXXXXXXXXXXXXXXXX
```

### 查看项目启动

在我们把配置文件修改完成后，然后启动5个服务后，我们使用下列命令查看启动的端口号

```
netstat -tunlp
```

能够发现5个服务已经成功启动了

![image-20200101131754872](images/image-20200101131754872.png)

到目前为止已经启动了对应的端口了

tip：有些小伙伴反馈rabbitmq端口经常断开，所以在启动项目的时候，注意查看是否出现rabbitmq的报错信息，如果有的话，那就是因为rabbitmq没有正确启动而造成的，使用下面的命令启动即可

```bash
# 启动rabbitmq
systemctl start rabbitmq-server
```

### 查看Eureka注册中心

服务器都启动完成后，下面我们验证一下后台是否正常启动，访问下列的地址：

```
http://your_ip:8761
```

首次登录会出现登录框：用户名：user，密码：password123

这个用户名密码是可以修改的，我们如果要修改的话，可以到mogu_eureka 下的 config文件夹修改配置

```yaml
spring:
  jmx:
    default-domain: mogu_eureka
  security:
    user:
      name: user
      password: password123

eureka:
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://user:password123@localhost:8761/eureka
```

我们需要修改里面的对应name  和 password即可，同时修改后，对应的defaultZone也需要修改

如果出现下面的页面，说明已经成功启动

![image-20200103100004362](images/image-20200103100004362.png)



### 查看swagger-ui页面

后台项目启动后，都能够看到对应的swagger-ui页面

```
# admin接口
http://your_ip:8601/swagger-ui.html
# picture接口
http://your_ip:8601/swagger-ui.html
# web接口
http://your_ip:8603/swagger-ui.html
```

进入 admin接口是这样的

![image-20200103101522521](images/image-20200103101522521.png)

选择LoginRestApi，验证登录，输入默认用户名和密码：admin  mogu2018

![image-20200103101627025](images/image-20200103101627025.png)

登录功能正常使用，我们把token复制到来，然后在swagger页面的右上角，有一个authorize的按钮，点击后，将token粘贴进去，即可操作全部接口进行测试了~

![image-20200103101654027](images/image-20200103101654027.png)

## Vue项目打包

前端部分，我们现在需要修改两个地方的配置，分别是：vue_mogu_admin 和 vue_mogu_web

### vue_mogu_web项目打包

下面我们到 vue_mogu_web/config/目录下，修改prod.env.js文件

```
  //配置线上环境
  WEB_API: '"http://yourIp:8603"',
  PICTURE_HOST: '"http://youip:8600"',
```

如果有域名的小伙伴，可以改成下面的配置，例如我这边有 picture.moguit.cn，用来显示图片，即可改成

```
  //配置线上环境
  WEB_API: '"http://yourIp:8603"',
  PICTURE_HOST: '"http://picture.moguit.cn"',
```

然后执行下列命令，进行依赖安装和打包

```
# 安装依赖
npm install --registry=https://registry.npm.taobao.org

# 打包
npm run build
```

打包完成后，会生成一个dist目录，我们将整个dist目录，压缩成 zip格式

![img](images/1574822138267.png)

然后使用xftp工具，丢入到我们的前端目录下，目录在 /home/mogu_blog/vue_mogu_web

![image-20200103103853899](images/image-20200103103853899.png)

注意：如果该文件夹下存在 dist文件夹，我们需要将其删除，然后在解压，然后使用下面命令进行解压

```
unzip dist.zip
```

### vue_mogu_admin项目打包

前台admin项目，修改配置的方式基本类似，我们到 vue_mogu_admin/config/目录下，修改prod.env.js文件，然后把里面的ip地址，改成你对应服务器的即可

```
// 生产环境
ADMIN_API: '"http://youIp:8601"',
PICTURE_API: '"http://youIp:8602"',
WEB_API: '"http://youIp:8603"',
SOLR_API: '"http://youIp:8080/solr"',
BASE_IMAGE_URL: '"http://youIp:8600"',
BLOG_WEB_URL: '"http://youIp:9527"',
```

如有拥有域名的小伙伴，可以将配置改成下面的形式

```
//生产环境
ADMIN_API: '"http://101.132.194.128:8601"',
PICTURE_API: '"http://101.132.194.128:8602"',
WEB_API: '"http://101.132.194.128:8603"',
SOLR_API: '"http://101.132.194.128:8080/solr"',
BASE_IMAGE_URL: '"http://picture.moguit.cn"',
BLOG_WEB_URL: '"http://www.moguit.cn"',
```

修改完成后prod.js后，我们还需要修改ckeditor图片上传的接口，vue_mogu_admin/static/ckeditor/config.js，修改下面的内容，这个只需要在admin项目才需要修改，web中没有使用ckeditor

```
//开启工具栏“图像”中文件上传功能，后面的url为待会要上传action要指向的的action或servlet
config.filebrowserImageUploadUrl = "http://your_ip:8602/ckeditor/imgUpload?";

//开启插入\编辑超链接中文件上传功能，后面的url为待会要上传action要指向的的action或servlet                                                                                                   
config.filebrowserUploadUrl = 'http://your_ip:8602/ckeditor/fileUpload';
```

修改完成后，需要进行重新编译~ 打包~ 部署

```
# 安装依赖
npm install --registry=https://registry.npm.taobao.org

# 打包
npm run build
```

我们在按上述操作，把dist压缩，然后放到/home/mogu_blog/vue_mogu_admin文件夹下

![img](images/1574822290047.png)

然后解压

```
unzip dist.zip
```

## 访问前端项目

执行完上述操作后，即可重新访问页面了

```
# 前台项目
http://yourIp:9527
# 后台项目 admin  mogu2018
http://yourIp:9528
```

如果配置了域名的话，输入对应的域名即可

```yaml
# 前台项目
http://www.moguit.cn
# 后台项目 admin  mogu2018
http://admin.moguit.cn
```

