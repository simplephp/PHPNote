> 关于全局ID生产实践

 最近朋友在咨询我全局ID如何保证全局唯一、时序性递增、单调递增、安全性问题，我在早期写过 一篇关于[高并发下ID生成方案](https://blog.csdn.net/yeshencat/article/details/72460458 "高并发下ID生成方案")的文章，算是初级的全局ID生成方案，有很多细节都是没有考虑到的，到生环境任然需要大量的细节打磨，但应对小型的应用采用数据库的方式就能解决的。在场景下咱们还可以去优化，下面来具体实践一下。


> 关于全局ID生成文章阅读推荐：
[Leaf——美团点评分布式ID生成系统](https://tech.meituan.com/2017/04/21/mt-leaf.html)、[Leaf：美团分布式ID生成服务开源](https://tech.meituan.com/2019/03/07/open-source-project-leaf.html)，美团在生产实践中的高可用方案(基于：DB、Snowflake算法等)，以下是基于 leaf 服务展开的(不重复造轮子)。

**环境要求**

> 1. 系统要求
    
    centOS/Windows
    
> 2. 编程语言

    PHP7/Java
    PHP 框架使用 hyperf，基于操作流程：PHP 使用 GRPC Client 调用 Java 提供的全局ID生成服务

> 3. PHP 代码部分，关于 grpc [代码生成可参阅](https://hyperf.wiki/#/zh-cn/grpc),主要安装 protobuf 、编写 *.proto 生成对应的编程语言代码。非框架的可以参考 [Swoole驱动的Grpc协程客户端, 底层使用高性能协程Http2-Client客户端](https://github.com/swoole/grpc)


```
咱们使用的PHP那就直接 protoc --php_out=grpc/ grpc.proto
# tree grpc
grpc
├── GPBMetadata
│   └── Grpc.php
└── Grpc
    ├── HiReply.php
    └── HiUser.php
```

```
配置 composer.json, 使用 grpc/ 下代码的自动加载. 如果 proto 
文件中使用不同的 package 设置, 或者使用了不同的目录, 
进行相应调整即可，添加之后执行 composer dump-autoload 使自动加载生效

"autoload": {
    "psr-4": {
        "App\\": "app/",
        "GPBMetadata\\": "grpc/GPBMetadata",
        "Grpc\\": "grpc/Grpc"
    },
    "files": [
    ]
},
```

```
    /**
     * 就几句代码 是不是很简单，归功于 hyperf 的高度封装，减少了代码量
     * grpc 调用， 服务端 java
     */
    public function grpc() {
        
        // 192.168.12.69:50051 java 服务对应的地址和端口
        $name = $this->request->input('name', 'Hyperf-Grpc-Client');
        $client = new \App\Grpc\HelloClient('192.168.12.69:50051', [
            'credentials' => null,
        ]);

        $request = new \Helloworld\HelloRequest();
        $request->setName($name);

        list($reply, $status, $response) = $client->sayHello($request);

        echo $response->data.PHP_EOL;

        return [
            'gid' => $response->data,
        ];
    }
```
> 3. Java 代码部分(基于 GRPC demo )

 ```
    public static void main(String[] args) throws IOException, InterruptedException {
    	
        final GreeterServer server = new GreeterServer();
        // 初始化DB
        dataSource = new DruidDataSource();
        dataSource.setUrl("jdbc:mysql://localhost:3306/leaf");
        dataSource.setUsername("root");
        dataSource.setPassword("123456");
        try {
			dataSource.init();
		} catch (SQLException e) {
			e.printStackTrace();
		}

        IDAllocDao dao = new IDAllocDaoImpl(dataSource);
        
        idGen = new SegmentIDGenImpl();
        ((SegmentIDGenImpl) idGen).setDao(dao);
        idGen.init();
        
        server.start();
        server.blockUntilShutdown();
    }
```

```
        // grpc 调用地方
        Result r = idGen.get("leaf-segment-test");
    	System.out.println(r.getId());
    	
    	HelloReply reply = null;
    
    	if(r.getStatus() == Status.SUCCESS) {
    		 reply = HelloReply.newBuilder().setMessage(("全局ID"+r.getId())).build();
    	} else {
    		 reply = HelloReply.newBuilder().setMessage(("Build Fail")).build();
    	}
```

> 图
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019122612170966.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2tpbmdzdGFydDAx,size_16,color_FFFFFF,t_70)![在这里插入图片描述](https://img-blog.csdnimg.cn/20191226121727172.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2tpbmdzdGFydDAx,size_16,color_FFFFFF,t_70)
