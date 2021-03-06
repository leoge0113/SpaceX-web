## 最初的梦想
三步走：
- version1.0为高可用（物理库，内存库）ssm web应用；
- version2.0为web服务性能提升：tomcat集群与页面分离版本；
- version3.0为分布式架构web系统：实现了分布式锁，队列等。版本与1.0,2.0区别较大，另起一个名为“SpaceX-Web-Distributed”项目。
## maven依赖
![image](https://github.com/leoge0113/elegant_ssm/blob/master/image/SpaceX_Web.jpg)

## version1.0 开发点滴
- 类构造函数特殊性造成该类对象protostuff反序列化问题

    已修复。
- 添加lambda表达式练习代码

    为了写redis操作的代码。
    函数接口是个泛型接口，lambda怎么写。
- redis 应用

    添加，
    删除（模糊/精确），查找，清空等。
    
    应用场景：为什么用内存库，什么时候添加，expiretime为什么要，什么时候清空
- service层里spring数据库事务的应用
- 自动义runtimeexception使用
- controler里使用注解@valid对form表单进行校验
- 自定义注解使用
- 采用AOP的方式判断验证结果
- Druid网络统计与监控
- 捕获全局错误

    实现GlobalExceptionResolver implements HandlerExceptionResolver 
    配置bean：
    <!--全局异常捕捉 -->
        <bean class="com.cainiao.exception.GlobalExceptionResolver" />
- redis 开启远程登录
    
    redis.conf
    1. \#bind localhost
    2. protected mode no
    3. 重启  
- spring 定时任务开发
- 集群预研之一致性hash

    分布式环境下评估hash算法好坏依据。
    1、平衡性(Balance)：平衡性是指哈希的结果能够尽可能分布到所有的缓冲中去，这样可以使得所有的缓冲空间都得到利用。很多哈希算法都能够满足这一条件。
    
    2、单调性(Monotonicity)：单调性是指如果已经有一些内容通过哈希分派到了相应的缓冲中，又有新的缓冲加入到系统中。哈希的结果应能够保证原有已分配的内容可以被映射到原有的或者新的缓冲中去，而不会被映射到旧的缓冲集合中的其他缓冲区。 
    
    3、分散性(Spread)：在分布式环境中，终端有可能看不到所有的缓冲，而是只能看到其中的一部分。当终端希望通过哈希过程将内容映射到缓冲上时，由于不同终端所见的缓冲范围有可能不同，从而导致哈希的结果不一致，最终的结果是相同的内容被不同的终端映射到不同的缓冲区中。这种情况显然是应该避免的，因为它导致相同内容被存储到不同缓冲中去，降低了系统存储的效率。分散性的定义就是上述情况发生的严重程度。好的哈希算法应能够尽量避免不一致的情况发生，也就是尽量降低分散性。 
    
    4、负载(Load)：负载问题实际上是从另一个角度看待分散性问题。既然不同的终端可能将相同的内容映射到不同的缓冲区中，那么对于一个特定的缓冲区而言，也可能被不同的用户映射为不同 的内容。与分散性一样，这种情况也是应当避免的，因此好的哈希算法应能够尽量降低缓冲的负荷。
    
    ```
    1 2 3 4 5 6 7 8 (key)
    mod 2:
    2 1 2 1 2 1 2 1
    mod 4:
    2 3 4 1 2 3 4 1
    
    
    1 2 3 4 5 6 7 8 9(key)
    mode 3：
    2 3 1 2 3 1 2 3 1
    mod 5
    2 3 4 1 2 3 4 1 2
    ```
    ==数据库分库分表时，选择2的幂次方，单调性比较好。mod 2^n 可以称为一致性hash==

-  redis m-s集群部署，代码调测
   
   遇到的问题总结见文档《redis cluster部署与开发问题总结》
- java8 optional 特性应用 

## version2.0开发点滴
### 前后端分离

#### 为什么要前后端分离
MVC是一种经典的设计模式，全名为Model-View-Controller，即模型-视图-控制器。
其中，模型是用于封装数据的载体，例如，在Java中一般通过一个简单的POJO（Plain Ordinary Java Object）来表示，其本质是一个普通的Java Bean，包含一系列的成员变量及其getter/setter方法。对于视图而言，它更加偏重于展现，也就是说，视图决定了界面到底长什么样子，在Java中可通过JSP来充当视图，或者通过纯HTML的方式进行展现，而后者才是目前的主流。模型和视图需要通过控制器来进行粘合，例如，用户发送一个HTTP请求，此时该请求首先会进入控制器，然后控制器去获取数据并将其封装为模型，最后将模型传递到视图中进行展现。
综上所述，MVC的交互过程如图下图所示。
```
graph LR
A[http request]-->B[Controller]
B -->  C[Model]
C-->D[Viewer]
B -->D

```
MVC模式的不足：
1. 每次请求必须经过“控制器->模型->视图”这个流程，用户才能看到最终的展现的界面，这个过程似乎有些复杂。
2. 实际上视图是依赖于模型的，换句话说，如果没有模型，视图也无法呈现出最终的效果。
3. 渲染视图的过程是在服务端来完成的，最终呈现给浏览器的是带有模型的视图页面，性能无法得到很好的优化。

#### 前后端分离方案

为了使数据展现过程更加直接，并且提供更好的用户体验，我们有必要对MVC模式进行改进。不妨这样来尝试，首先从浏览器发送AJAX请求，然后服务端接受该请求并返回JSON数据返回给浏览器，最后在浏览器中进行界面渲染。
改进后的MVC模式如图下图所示。

```
graph LR
A[View]-->|Ajx| B[Controller] 
B-->C[Model]
C-->|Json|A
```
也就是说，我们输入的是AJAX请求，输出的是JSON数据，市面上有这样的技术来实现这个功能吗？答案是REST。

### Nginx
#### Nginx 作为静态资源服务器
#### Nginx作为负载均衡服务器
##### Nginx 平滑加权轮询算法
普通加权轮询算法有个缺陷，就是某些情况下生成的序列是不均匀的。比如针对这样的配置：
```
http {    
    upstream cluster {    
        server a weight=5;    
        server b weight=1;    
        server c weight=1;    
    }    
    ...  
}   
```
生成的序列是这样的：{a,a, a, a, a, c, b}。会有5个连续的请求落在后端a上，分布不太均匀。
 
　　在Nginx源码中，实现了一种叫做平滑的加权轮询（smooth weighted round-robin balancing）的算法，它生成的序列更加均匀。比如前面的例子，它生成的序列为{ a, a, b, a, c, a, a}，转发给后端a的5个请求现在分散开来，不再是连续的。
 
　　该算法的原理如下：
　　每个服务器都有两个权重变量：
　　a：weight，配置文件中指定的该服务器的权重，这个值是固定不变的；
　　b：current_weight，服务器目前的权重。一开始为0，之后会动态调整。
 
　　每次当请求到来，选取服务器时，会遍历数组中所有服务器。对于每个服务器，让它的current_weight增加它的weight；同时累加所有服务器的weight，并保存为total。
　　遍历完所有服务器之后，如果该服务器的current_weight是最大的，就选择这个服务器处理本次请求。最后把该服务器的current_weight减去total。
 
　　上述描述可能不太直观，来看个例子。比如针对这样的配置：
```
http {    
    upstream cluster {    
        server a weight=4;    
        server b weight=2;    
        server c weight=1;    
    }    
    ...  
}   
```
按照这个配置，生成的序列是{a,,a,c,a,b,a}.
### 前后端分离controller开发
### Tomcat 配置实现session共享
