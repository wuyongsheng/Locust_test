

> 这段时间接触了一款新的性能测试工具：Locust，和Loadrunner及Jmeter一样，也是一款很好的性能测试工具。

### 简介


在Locust测试框架中，测试场景是采用纯Python脚本进行描述的。对于最常见的HTTP(S)协议的系统，Locust采用Python的requests库作为客户端，使得脚本编写大大简化，富有表现力的同时且极具美感。而对于其它协议类型的系统，Locust也提供了接口，只要我们能采用Python编写对应的请求客户端，就能方便地采用Locust实现压力测试。从这个角度来说，Locust可以用于压测任意类型的系统。

在模拟有效并发方面，Locust的优势在于其摒弃了进程和线程，完全基于事件驱动，使用gevent提供的非阻塞IO和coroutine来实现网络层的并发请求，因此即使是单台压力机也能产生数千并发请求数；再加上对分布式运行的支持，理论上来说，Locust能在使用较少压力机的前提下支持极高并发数的测试。

###  安装

使用pip命令安装Locust

pip install locustio

安装完成后，检测是否安装成功

```

C:\Users\Administrator>locust  -help
Usage: locust [options] [LocustClass [LocustClass2 ... ]]

Options:
  -h, --help            show this help message and exit
  -H HOST, --host=HOST  Host to load test in the following format:
                        http://10.21.32.33
  --web-host=WEB_HOST   Host to bind the web interface to. Defaults to
 '' (all
                        interfaces)
  -P PORT, --port=PORT, --web-port=PORT
                        Port on which to run web host
  -f LOCUSTFILE, --locustfile=LOCUSTFILE
                        Python module file to import, e.g. '../other.p
y'.
                        Default: locustfile
  --csv=CSVFILEBASE, --csv-base-name=CSVFILEBASE
                        Store current request stats to files in CSV fo
rmat.
  --master              Set locust to run in distributed mode with thi
s
```
### 测试案例

####  测试场景

对Django rest api 进行测试

Django 的安装，菜鸟教程有介绍

http://www.runoob.com/django/django-install.html

安装完成后，使用如下命令启动，

``` 

D:\django_restful>python3 manage.py runserver 127.0.0.1:9000 

```

启动完成之后，在浏览器中输入如下地址，就可以访问api接口：

http://127.0.0.1:9000/

如下图所示：

![Django.jpg](https://upload-images.jianshu.io/upload_images/12273007-7c81bf49d7d77c15.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

点击其中的 users 和 groups 链接，会分别显示相应的用户和组的信息，如下图：

![users.jpg](https://upload-images.jianshu.io/upload_images/12273007-b6ae63aff585b937.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![groups.jpg](https://upload-images.jianshu.io/upload_images/12273007-125041ec48a13dbb.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


本次针对如下两个接口进行测试：

```
    "users": "http://127.0.0.1:9000/users/",
    "groups": "http://127.0.0.1:9000/groups/"
```
#### 编写简单的测试脚本

创建 locust_test.py 文件，通过 Python 编写性能测试脚本。

```
from locust import HttpLocust,TaskSet,task

class UserBehavior(TaskSet):
    def on_start(self):
        self.users_index=0
        self.groups_index=0

    @task(2)
    def test_users(self):
        users_id=self.locust.id[self.users_index]
        url='/users/'+str(users_id)+'/'
        self.client.get(url,auth=('wysh','123456'))
        self.users_index=(self.users_index+1)%len(self.locust.id)

    @task(1)
    def test_groups(self):
        groups_id=self.locust.id[self.groups_index]
        url='/groups/'+str(groups_id)+'/'
        self.client.get(url,auth=('wysh','123456'))
        self.groups_index=(self.groups_index+1)%len(self.locust.id)

class WebsiteUser(HttpLocust):
    task_set = UserBehavior
    id=[1,2]
    min_wait = 3000
    max_wait = 6000
    host = 'http://127.0.0.1:9000'
```
那么，如上Python脚本是如何表达出以上测试场景的呢？

- 从脚本中可以看出，脚本主要包含两个类，一个是WebsiteUser（继承自HttpLocust，而HttpLocust继承自Locust），另一个是UserBehavior（继承自TaskSet）。事实上，在Locust的测试脚本中，所有业务测试场景都是在Locust和TaskSet两个类的继承子类中进行描述的。

-  task:装饰该方法为一个事务后面的数字表示请求比例,上面的比例为2:1默认都是1:1
-  test_ users()方法表示个用户行为,这里是请求user接口。
-  test_ groups()方法表示请求 group接口
-  client.get()用于指定请求的路径
-  Websiteuser类用于设置性能测试。
-  task_set:指向一个定义的用户行为类。
-  min wait:执行事务之间用户等待时间的下界(单位:亳秒)
-  max wait:执行事务之间用户等待时间的上界(单位:亳秒)

####  执行性能测试

Locust脚本调试通过后，就算是完成了所有准备工作，可以开始进行压力测试了。

使用如下命令，开始执行性能测试：

```
locust -f D:\PycharmProjects\locust\locust_test.py --host
=http://127.0.0.1:9000
```
![启动locust.jpg](https://upload-images.jianshu.io/upload_images/12273007-052fa96eba4d3414.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

参数说明：

Locust是通过在Terminal中执行命令进行启动的，通用的参数有如下两个：

-  -H, --host：被测系统的host，若在Terminal中不进行指定，就需要在Locust子类中通过host参数进行指定
-  -f, --locustfile：指定执行的Locust脚本文件

####  设置测试
通过浏览器访问：http://localhost:8089（Locust启动网络监控器，默认为端口号为: 8089），如果要使用其它端口，就可以在上面的启动命令中使用如下参数进行指定：-P, --port：

![设置locust.jpg](https://upload-images.jianshu.io/upload_images/12273007-9bb0cdbf1a1becff.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在Locust的Web管理页面中，需要配置的参数只有两个：

-  Number of users to simulate: 设置并发用户数，对应中no_web模式的-c, --clients参数；
-  Hatch rate (users spawned/second): 启动虚拟用户的速率，对应着no_web模式的-r, --hatch-rate参数。
- 参数配置完毕后，点击【Start swarming】即可开始测试。

运行之后，可以看到主界面如下：

![运行的界面.jpg](https://upload-images.jianshu.io/upload_images/12273007-5c8e088a8f726315.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

性能测试参数

-  Type： 请求的类型，例如GET/POST。

-  Name：请求的路径。

-  request：当前请求的数量。

-  fails：当前请求失败的数量。

-  Median：中间值，单位毫秒，一半的服务器响应时间低于该值，而另一半高于该值。

-  Average：平均值，单位毫秒，所有请求的平均响应时间。

-  Min：请求的最小服务器响应时间，单位毫秒。

-  Max：请求的最大服务器响应时间，单位毫秒。

-  Content Size：单个请求的大小，单位字节。

-  reqs/sec：是每秒钟请求的个数。

点击Chart菜单，可以查看性能图表

![饼图.jpg](https://upload-images.jianshu.io/upload_images/12273007-7328063025899c76.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
