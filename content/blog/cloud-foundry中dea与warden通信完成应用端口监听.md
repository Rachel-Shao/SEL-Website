+++

id= "209"

title = "Cloud Foundry中DEA与warden通信完成应用端口监听"
describtion = "在Cloud Foundry v2中，当应用用户需要启动应用的实例时，用户通过cf CLI向cloud controller发送请求，而cloud controller通过NATS向DEA转发启动请求。真正执行启动事宜的是DEA，DEA主要做的工作为启动一个warden container, 并将droplet等内容拷贝进入container内部，最后配置完指定的环境变量，在这些环境变量下启动应用的启动脚本。 本文将从阐述Cloud Foundry中DEA如何为应用实例的启动配置环境变量。 "
tags= [ "dea" , "cloudfoundry" ]
date= "2014-12-02 16:55:44"
author = "孙宏亮"
banner= "img/blogs/209/cloud-foundry.jpg"
categories = [ "cloudfoundry" ]

+++


在Cloud Foundry v2版本中，DEA为一个用户应用运行的控制模块，而应用的真正运行都是依附于warden。更具体的来说，是DEA接收到Cloud Controller的请求；DEA发送请求给warden server；warden server创建warden container并将用户应用droplet等环境配置好；DEA发送应用启动请求至warden serve；最后warden container执行启动脚本启动应用。 本文主要具体描述，DEA如何与warden交互，以保证最终用户的应用可以成功绑定某一个端口，实现用户应用对外提供服务。 


DEA在执行启动一个应用的时候，主要做到以下这些部分：promise\_droplet, promise\_container, 其中这两个部分并发完成；promise\_extract\_droplet, promise\_exec\_hook\_script(“before\_start”), promise\_start等。代码如下：

    \[  
      promise\_droplet,  
      promise\_container  
    \].each(&:run).each(&:resolve)  
    
    \[  
      promise\_extract\_droplet,  
      promise\_exec\_hook\_script('before\_start'),  
      promise\_start  
    \].each(&:resolve)  

**promise\_droplet:**
=====================

在这一个环节，DEA主要做的工作是将droplet下载本机，通过droplet\_uri,其中基本的路径在/config/dea.yml中，为base\_dir: /tmp/dea\_ng, 因此最终DEA下载到的droplet存放于DEA组件所在的宿主机上。

**promise\_container:**
=======================

该环节的工作主要完成创建一个warden container，随后可以为应用的运行提供一个合适的环境。promise\_container的源码实现如下：

    def promise\_container  
        Promise.new do |p|
        bind\_mounts = \[{'src\_path' => droplet.droplet\_dirname, 'dst\_path' => droplet.droplet\_dirname}\]  
        with\_network = true  
        container.create\_container(  
            bind\_mounts: bind\_mounts + config\['bind\_mounts'\],  
            limit\_cpu: config\['instance'\]\['cpu\_limit\_shares'\],  
            byte: disk\_limit\_in\_bytes,  
            inode: config.instance\_disk\_inode\_limit,  
            limit\_memory: memory\_limit\_in\_bytes,  
            setup\_network: with\_network)  
        attributes\['warden\_handle'\] = container.handle  
        promise\_setup\_def create\_container(params)  
        \[:bind\_mounts, :limit\_cpu, :byte, :inode, :limit\_memory, :setup\_network\].each do |param|  
        raise ArgumentError, "expecting #{param.to\_s} parameter to create container" if params\[param\].nil?  
    end  
    
    with\_em do  
        new\_container\_with\_bind\_mounts(params\[:bind\_mounts\])  
        limit\_cpu(params\[:limit\_cpu\])  
        limit\_disk(byte: params\[:byte\], inode: params\[:inode\])  
        limit\_memory(params\[:limit\_memory\])  
        setup\_network if params\[:setup\_network\]  
    end  
    endenvironment.resolve  
        p.deliver  
        end  
    end  

可以看到传入的参数主要有：

*   bind\_mounts：完成宿主机文件目录的路径mount到container内部；
*   limit\_cpu：用于限制container的CPU资源分配；
*   byte：磁盘限额；
*   innode：磁盘的innode的限制
*   limit\_memory：内存限额；
*   setup\_network：网络的配置项。

其中setup\_network一直设置为true。 在container.create\_container的方法实现中，有以下的方法，如下：

    def create\_container(params)  
        \[:bind\_mounts, :limit\_cpu, :byte, :inode, :limit\_memory, :setup\_network\].each do |param|  
          raise ArgumentError, "expecting #{param.to\_s} parameter to createdef create\_container(params)  
        \[:bind\_mounts, :limit\_cpu, :byte, :inode, :limit\_memory, :setup\_network\].each do |param|  
          raise ArgumentError, "expecting #{param.to\_s} parameter to create container" if params\[param\].nil?  
        end  
    
      with\_em do  
          new\_container\_with\_bind\_mounts(params\[:bind\_mounts\])  
          limit\_cpu(params\[:limit\_cpu\])  
          limit\_disk(byte: params\[:byte\], inode: params\[:inode\])  
          limit\_memory(params\[:limit\_memory\])  
          setup\_network if params\[:setup\_network\]  
      end  
    end container" if params\[param\].nil?  
      end  
    
      with\_em do  
        new\_container\_with\_bind\_mounts(params\[:bind\_mounts\])  
        limit\_cpu(params\[:limit\_cpu\])  
        limit\_disk(byte: params\[:byte\], inode: params\[:inode\])  
        limit\_memory(params\[:limit\_memory\])  
        setup\_network if params\[:setup\_network\]  
      end  
    end  

主要关注一下setup\_network方法，如下：

    def setup\_network  
        request = ::Warden::Protocol::NetInRequest.new(handle: handle)  
        response = call(:app, request)  
        network\_ports\['host\_port'\] = response.host\_port  
        network\_ports\['container\_port'\] = response.container\_port  
    
        request = ::Warden::Protocol::NetInRequest.new(handle: handle)  
        response = call(:app, request)  
        network\_ports\['console\_host\_port'\] = response.host\_port  
        network\_ports\['console\_container\_port'\] = response.container\_port 
    
    end  

从代码中可以看到，在setup\_network中，主要完成了两次NetIn操作。对于NetIn操作，需要说明的是，完成的工作是将host主机上的端口映射到container内部的端口。换言之，将host\_ip:port1映射到container\_ip:port2，也就是说如果container在container\_ip上监听的是端口port2，则host机器外部的请求访问host机器，并且端口为port1的时候，host的内核网络栈，会将请求转发给container的port2端口，其中使用的协议为DNAT协议。 因此，在以上的代码中实现了两次NetIn操作，也就是说将container的两个端口映射到了host宿主机，第一个端口用于container内应用的正常占用端口，第二个端口是用来为应用的console功能做服务。虽然container也分配了第二个端口，但是在而后的应用启动等中，该console\_port都没有使用过，可见Cloud Foundry在这里只是预留了接口，但是没有真正利用起来。 以上主要描述了NetIn的功能，以下进入NetIn操作的源码实现。NetIn的源码实现，主要为warden server的部分。其中，是由DEA进程通过warden.sock和warden server建立通信，随后DEA进程发送NetIn请求给warden server，warden server最终处理该请求。 现在进入warden范畴，研究warden如何接收请求，并实现端口的映射。 在warden/lib/warden/server.rb中，大部分代码都是为了完成warden server的运行，在run！方法中，可以看到warden server另外还启动了一个unix domain server，代码如下：

    server = ::EM.start\_unix\_domain\_server(unix\_domain\_path, ClientConnection)  

也就是说，warden server会通过整个unix domain server接收从DEA进程发送来的关于warden container的一系列请求，在ClientConnection类中定义，关于这部分请求如何处理的方法。 当unix domain server中ClientConnection类通过receive\_data（data）方法来实现接收请求，代码如下：

    def receive\_data(data)  
      @buffer << data  
      @buffer.each\_request do |request|  
        begin  
            receive\_request(request)  
        rescue => e  
            close\_connection\_after\_writing  
            logger.warn("Disconnected client after error")  
            logger.log\_exception(e)  
        end  
      end  
    end  

从代码中可以看到，当buffer中有请求的时候，通过receive\_request（request）方法来进一步提取请求。再进入receive\_request(request)方法中，可以看到通过process（request）来处理请求。 接着进入真正请求处理的部分，也就时process（request）的实现：

    def process(request)  
        case request  
        when Protocol::PingRequest  
          response = request.create\_response  
          send\_response(response)  
    
        when Protocol::ListRequest  
          response = request.create\_response  
          response.handles = Server.container\_klass.registry.keys.map(&:to\_s)  
          send\_response(response)  
    
        when Protocol::EchoRequest  
          response = request.create\_response  
          response.message = request.message  
          send\_response(response)  
    
        when Protocol::CreateRequest  
          container = Server.container\_klass.new  
          container.register\_connection(self)  
          response = container.dispatch(request)  
          send\_response(response)  
    
        else  
          if request.respond\_to?(:handle)  
            container = find\_container(request.handle)  
            process\_container\_request(request, container)  
          else  
            raise WardenError.new("Unknown request: #{request.class.name.split("::").last}")  
          end  
        end  
    rescue WardenError => e  
        send\_error(e)  
    rescue => e  
        logger.log\_exception(e)  
        send\_error(e)  
    end  

可见，在warden server中，请求类型可以简单分为5种：PingRequest, ListRequest, EchoRequest, CreateRequest和其他请求，像NetIn请求则属于其他请求中的一种，程序执行进入case语句块的else分支，也就是：

    if request.respond\_to?(:handle)  
      container = find\_container(request.handle)  
      process\_container\_request(request, container)  
    else  
      raise WardenError.new("Unknown request: #{request.class.name.split("::").last}")  
    end

代码清晰可见，warden server首先通过handle找到具体是给哪一个warden container发送请求，然后调用process\_container\_request(request, container)方法。进入process\_container\_request(request, container)方法可以看到：加入请求类型不为StopRequest以及StreamRequest，则进入case语句块的else分支，执行代码：

    response = container.dispatch(request)  
         send\_response(response)  

可以看到，是调用了container.dispatch（request）方法才返回了response。 以下进入warden/lib/warden/container/base.rb文件中，该文件的dispatch方法主要实现了warden server接收到请求并预处理之后，如何分发执行具体的请求，代码如下：

    def dispatch(request, &blk)  
        klass\_name = request.class.name.split("::").last  
        klass\_name = klass\_name.gsub(/Request$/, "")  
        klass\_name = klass\_name.gsub(/(.)(\[A-Z\])/) { |m| "#{m\[0\]}\_#{m\[1\]}" }  
        klass\_name = klass\_name.downcase  
    
        response = request.create\_response  
    
        t1 = Time.now  
    
        before\_method = "before\_%s" % klass\_name  
        hook(before\_method, request, response)  
        emit(before\_method.to\_sym)  
    
        around\_method = "around\_%s" % klass\_name  
        hook(around\_method, request, response) do  
            do\_method = "do\_%s" % klass\_name  
        send(do\_method, request, response, &blk)  
        end  
    
        after\_method = "after\_%s" % klass\_name  
        emit(after\_method.to\_sym)  
        hook(after\_method, request, response)  
    
        t2 = Time.ndef dispatch(request, &blk)  
        klass\_name = request.class.name.split("::").last  
        klass\_name = klass\_name.gsub(/Request$/, "")  
        klass\_name = klass\_name.gsub(/(.)(\[A-Z\])/) { |m| "#{m\[0\]}\_#{m\[1\]}" }  
        klass\_name = klass\_name.downcase  
    
        response = request.create\_response  
    
        t1 = Time.now  
    
        before\_method = "before\_%s" % klass\_name  
        hook(before\_method, request, response)  
        emit(before\_method.to\_sym)  
    
        around\_method = "around\_%s" % klass\_name  
        hook(around\_method, request, response) do  
            do\_method = "do\_%s" % klass\_name  
            send(do\_method, request, response, &blk)  
        end  
    
        after\_method = "after\_%s" % klass\_name  
        emit(after\_method.to\_sym)  
        hook(after\_method, request, response)  
    
        t2 = Time.now  
    
        logger.info("%s (took %.6f)" % \[klass\_name, t2 - t1\],  
            :request => request.to\_hash,  
            :response => response.to\_hash)  
    
        response  
     endow  
    
        logger.info("%s (took %.6f)" % \[klass\_name, t2 - t1\],  
            :request => request.to\_hash,  
            :response => response.to\_hash)  
    
        response  
    end  

首先提取出请求的类型名，如果是NetIn请求的话，提取出来的请求的类型名称为net\_in，随后构建出方法明do\_method，也就是do\_net\_in，接着就通过send(do\_method, request, response, &blk)方法，将参数发送给do\_net\_in方法。 现在就是进入warden/lib/warden/container/features/net.rb文件中, do\_net\_in方法实现如下：

    def do\_net\_in(request, response)  
        if request.host\_port.nil?  
          host\_port = self.class.port\_pool.acquire  
    
          #Use same port on the container side as the host side if unspecified  
          container\_port = request.container\_port || host\_port  
    
          # Port may be re-used after this container has been destroyed  
          @resources\["ports"\] << host\_port  
          @acquired\["ports"\] << host\_port  
        else  
          host\_port = request.host\_port  
          container\_port = request.container\_port || host\_port  
        end  
    
        \_net\_in(host\_port, container\_port)  
    
        @resources\["net\_in"\] ||= \[\]  
        @resources\["net\_in"\] << \[host\_port, container\_port\]  
    
        response.host\_port      = host\_port  
        response.container\_port = container\_port  
    rescue WardenError  
      self.class.port\_pool.release(host\_port) unless request.host\_port  
      raise  
    end  

可见，如果请求端口没有指定的话，那么就使用代码host\_port = self.class.port\_pool.acquire来获取端口号，默认情况下，container端口号与host端口号保持一致，有了这两个端口号之后，执行代码\_net\_in(host\_port, container\_port)， 真正实现端口映射，如下：

    def \_net\_in(host\_port, container\_port)  
      sh File.join(container\_path, "net.sh"), "in", :env => {  
        "HOST\_PORT"      => host\_port,  
        "CONTAINER\_PORT" => container\_port,  
      }  
    end  

可以清晰的看到是，使用了容器内部的net.sh脚本来实现端口映射。现在进入warden/root/linux/skeleton/net.sh脚本，进入参数为in的执行部分：

    "in")  
        if \[ -z "${HOST\_PORT:-}" \]; then  
            echo "Please specify HOST\_PORT..." 1>&2  
            exit 1  
        fi  
        if \[ -z "${CONTAINER\_PORT:-}" \]; then  
          echo "Please specify CONTAINER\_PORT..." 1>&2  
          exit 1  
        fi  
        iptables -t nat -A ${nat\_instance\_chain} \\  
          --protocol tcp \\  
          --destination "${external\_ip}" \\  
          --destination-port "${HOST\_PORT}" \\  
          --jump DNAT \\  
          --to-destination "${network\_container\_ip}:${CONTAINER\_PORT}"  
        ;;  

可见，该脚本这部分的功能是在host主机创建一条DNAT规则，使得host主机上所有HOST\_PORT端口上的网络请求，都转发至network\_container\_ip:CONTAINER\_PORT上，也就是完成了目标地址IP转变。

**promise\_extract\_droplet:**
==============================

该环节主要完成的是让container运行脚本，使得container容器将位于host主机的droplet文件，解压至container内部，代码内容如下：

    def promise\_extract\_droplet  
        Promise.new do |p|  
          script = "cd /home/vcap/ && tar zxf #{droplet.droplet\_path}"  
          container.run\_script(:app, script)  
          p.deliver  
        end  
    end  
    

**promise\_exec\_hook\_script('before\_start')：**
=================================================

这部分主要完成的功能是让容器运行名为before\_start的脚本，在老的版本中，该部分的设置默认为空。

**promise\_start:**
===================

这部分主要完成的是应用的启动。其中创建容器的时候，返回的端口号，会被DEA保存。最终，当DEA启动应用的时候，由于DEA会将端口号作为参数传递给应用程序的启动脚本，因此当应用启动时会自动去监听已经为它设置的端口，即完成端口监听。 

**转载请注明出处。** 
----------

这篇文档更多出于我本人的理解，肯定在一些地方存在不足和错误。希望本文能够对接触Cloud Foundry v2中warden模块以及应用端口映射的人有些帮助，如果你对这方面感兴趣，并有更好的想法和建议，也请联系我。 