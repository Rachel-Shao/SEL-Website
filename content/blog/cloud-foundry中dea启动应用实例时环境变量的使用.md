+++

id= "188"

title = "Cloud Foundry中DEA启动应用实例时环境变量的使用"
describtion = "在Cloud Foundry v2中，当应用用户需要启动应用的实例时，用户通过cf CLI向cloud controller发送请求，而cloud controller通过NATS向DEA转发启动请求。真正执行启动事宜的是DEA，DEA主要做的工作为启动一个warden container, 并将droplet等内容拷贝进入container内部，最后配置完指定的环境变量，在这些环境变量下启动应用的启动脚本。 本文将从阐述Cloud Foundry中DEA如何为应用实例的启动配置环境变量。 "
tags= [ "dea" , "cloudfoundry" ]
date= "2014-11-20 13:03:30"
author = "孙宏亮"
banner= "img/blogs/188/cloud-foundry.jpg"
categories = [ "cloudfoundry" ]
+++


在Cloud Foundry v2中，当应用用户需要启动应用的实例时，用户通过cf CLI向cloud controller发送请求，而cloud controller通过NATS向DEA转发启动请求。真正执行启动事宜的是DEA，DEA主要做的工作为启动一个warden container, 并将droplet等内容拷贝进入container内部，最后配置完指定的环境变量，在这些环境变量下启动应用的启动脚本。 本文将从阐述Cloud Foundry中DEA如何为应用实例的启动配置环境变量。


**DEA接收应用启动请求及其执行流程**
=====================

在这部分，通过代码的形式来说明DEA对于应用启动请求的执行流程。 
1.首先DEA订阅相应主题的消息，主题为“dea.#{bootstrap.uuid}.start”，含义为“自身DEA的应用启动消息”：

    subscribe("dea.#{bootstrap.uuid}.start") do |message|  
        bootstrap.handle\_dea\_directed\_start(message)  
    end  
    

2.当收到订阅主题之后，执行bootstrap.handle\_dea\_directed\_start(message)，含义为“通过bootstrap类实例来处理应用的启动请求”：

    def handle\_dea\_directed\_start(message)  
        start\_app(message.data)  
    end  
    

3.可以认为处理的入口，即为以上代码中的start\_app方法：

    def start\_app(data)  
        instance = instance\_manager.create\_instance(data)  
        return unless instance
    
        instance.start  
    end  
    

4.在start\_app方法中，首先通过instance\_manager类实例来创建一个instance对象，通过执行instance实例的类方法start，可以看到自始至终，传递的参数的原始来源都是通过NATS消息传递来的message，也就是1中的message：

    def start(&callback)  
        p = Promise.new do  
        ……  
        \[  
            promise\_droplet,  
            promise\_container  
        \].each(&:run).each(&:resolve)  
        \[  
            promise\_extract\_droplet,  
            promise\_exec\_hook\_script('before\_start'),  
            promise\_start  
        \].each(&:resolve)  
        ……  
        p.deliver  
    end  
    

5.其中真正关于应用启动的执行在promise\_start方法中实现：

    def promise\_start  
      Promise.new do |p|  
        env = Env.new(StartMessage.new(@raw\_attributes), self)  
        if staged\_info  
           command = start\_command || staged\_info\['start\_command'\] 
           unless command  
             p.fail(MissingStartCommand.new)  
             next  
           end  
           start\_script = 
             Dea::StartupScriptGenerator.new(  
                command,  
                env.exported\_user\_environment\_variables,  
                env.exported\_system\_environment\_variables 
             ).generate 
        else  
         start\_script = env.exported\_environment\_variables + "./startup;\\nexit"  
        end  
        response = container.spawn(start\_script, container.resource\_limits(self.file\_descriptor\_limit, NPROC\_LIMIT))  
        attributes\['warden\_job\_id'\] = response.job\_id  
        container.update\_path\_and\_ip  
        bootstrap.snapshot.save 
        p.deliver 
      end  
    end  
    

可以看到在第5步中，DEA涉及到了应用的ENV环境变量等信息，最后通过container.spawn方法实现了应用的启动。

**DEA环境变量的配置**
==============

在以上步骤的第5步，首先创建了环境变量env = Env.new(StartMessage.new(@raw\_attributes), self)，Env类的初始化方法如下：

    def initialize(message, instance\_or\_staging\_task=nil)  
       @strategy\_env = if message.is\_a? StagingMessage  
         StagingEnv.new(message, instance\_or\_staging\_task)  
       else  
         RunningEnv.new(message, instance\_or\_staging\_task)  
       end  
    end  
    

可见，实际是创建了一个RunningEnv类的实例，参数主要为cloud controller发送来的信息。 在promise\_start方法中，创建env变量之后，通过判断staged\_info来选择start\_script变量的构建。 现在分析staged\_info的代码实现：

    def staged\_info  
      @staged\_info ||= begin  
        Dir.mktmpdir do |destination\_dir|  
           staging\_file\_name = 'staging\_info.yml'  
           copied\_file\_name = "#{destination\_dir}/#{staging\_file\_name}" 
           copy\_out\_request("/home/vcap/#{staging\_file\_name}", destination\_dir)  
           YAML.load\_file(copied\_file\_name) if File.exists?(copied\_file\_name)  
        end  
      end  
    end  
    

主要是通过指定路径，然后从路径中提取出变量。下面以一个ruby的应用为例，其staging\_info.yml文件的内容为：

    ---  
    detected\_buildpack: Ruby/Rack  
    start\_command: bundle exec rackup config.ru -p $PORT  
    

因此，最终@staged\_info的内容如上，在instance.start方法中，command为bundle exec rackup config.ru -p $PORT。有了command变量之后，接着是构建start\_script变量：

    start\_script =  
        Dea::StartupScriptGenerator.new(  
            command,  
            env.exported\_user\_environment\_variables,  
            env.exported\_system\_environment\_variables  
        ).generate  
    

可以看到，DEA通过StartupScriptGenerator类来创建start\_script，其中参数为三个，第一个为刚才涉及到的command，后两个均需要通过env变量来生成。 现在看exported\_user\_environment\_variables方法的实现：

    def exported\_user\_environment\_variables  
        to\_export(translate\_env(message.env))  
    end  
    

该方法实现从message中提取中属性为env的信息，作为用户的环境变量。 进入env.exported\_system\_environment\_variables的方法实现：

    def exported\_system\_environment\_variables  
        env = \[  
            \["VCAP\_APPLICATION",  Yajl::Encoder.encode(vcap\_application)\],  
            \["VCAP\_SERVICES", Yajl::Encoder.encode(vcap\_services)\],  
            \["MEMORY\_LIMIT", "#{message.mem\_limit}m"\]  
        \]  
        env << \["DATABASE\_URL", DatabaseUriGenerator.new(message.services).database\_uri\] if message.services.any?  
    
        to\_export(env + strategy\_env.exported\_system\_environment\_variables)  
    end  
    

可见在生成系统的环境变量的时候，首先创建一个env数组变量，其中有信息VCAP\_APPLICATION, VCAP\_SERVICES, MEMORY\_LIMIT三种，其中VCAP\_APPLICATION的信息如下：

    def vcap\_application  
        @vcap\_application ||=  
          begin  
            hash = strategy\_env.vcap\_application  
    
            hash\["limits"\] = message.limits  
            hash\["application\_version"\] = message.version  
            hash\["application\_name"\] = message.name  
            hash\["application\_uris"\] = message.uris  
            # Translate keys for backwards compatibility  
            hash\["version"\] = hash\["application\_version"\]  
            hash\["name"\] = hash\["application\_name"\]  
            hash\["uris"\] = hash\["application\_uris"\]  
            hash\["users"\] = hash\["application\_users"\]
    
            hash  
          end  
    end  
    

这部分信息中包含了应用的name，uris，users，version等一系列内容。其中需要特别注意的是代码hash = strategy\_env.vcap\_application，该代码的作用为调用了RunningEnv类中vcap\_application方法，如下：

    def vcap\_application  
        hash = {}  
        hash\["instance\_id"\] = instance.attributes\["instance\_id"\]  
        hash\["instance\_index"\] = message.index  
        hash\["host"\] = HOSTStartupScriptGenerator  
        hash\["port"\] = instance.instance\_container\_port  
        started\_at = Time.at(instance.state\_starting\_timestamp)  
        hash\["started\_at"\] = started\_at  
        hash\["started\_at\_timestamp"\] = started\_at.to\_i  
        hash\["start"\] = hash\["started\_at"\]  
        hash\["state\_timestamp"\] = hash\["started\_at\_timestamp"\]  
        hash  
    end  
    

可见在以上代码中，vcap\_application信息中记录了很多关于应用实例的信息，包括instance\_id, instance\_index, host, port, started\_at, started\_at\_timestamp, start, state\_timestamp等。 VCAP\_SERVICES的信息如下：

      WHITELIST\_SERVICE\_KEYS = %W\[name label tags plan plan\_option credentials syslog\_drain\_url\].freeze  
    def vcap\_services  
       @vcap\_services ||=  
          begin  
             services\_hash = Hash.new { |h, k| h\[k\] = \[\] }  
    
             message.services.each do |service|  
                service\_hash = {}  
                WHITELIST\_SERVICE\_KEYS.each do |key|  
                   service\_hash\[key\] = service\[key\] if service\[key\]  
                end  
    
                services\_hash\[service\["label"\]\] << service\_hash  
             end  
    
             services\_hash
          end
    end  
    

这部分内容主要从message中寻找是否存在和WHITELIST\_SERVICES\_KEYS中值作为键的值，若存在的话，加入services\_hash该变量。 随后，DEA在运行代码env << \["DATABASE\_URL", DatabaseUriGenerator.new(message.services).database\_uri\] if message.services.any?，该部分代码的作用主要要理解DatabaseUriGenerator的含义，这部分笔者仍不是很清楚。 再后来，DEA执行代码to\_export(env + strategy\_env.exported\_system\_environment\_variables)，这部分的内容非常重要，主要要进入strategy\_env对象所在的类中查看exported\_system\_environment\_variables方法：

    def exported\_system\_environment\_variables  
      env = \[  
        \["HOME", "$PWD/app"\],  
        \["TMPDIR", "$PWD/tmp"\],  
        \["VCAP\_APP\_HOST", HOST\],  
        \["VCAP\_APP\_PORT", instance.instance\_container\_port\],  
      \]  
      env << \["PORT", "$VCAP\_APP\_PORT"\]  
      env  
    end  
    

可以看到，这里主要包含运行信息的环境变量，比如说HOME目录，TMP临时目录，应用实例运行的host地址，应用实例运行的端口号。其中最为重要的就是应用实例运行的端口号，在我的另一篇博文Cloud Foundry中DEA与warden通信完成应用端口监听 中，有涉及到如何通过warden server来开辟端口，最终为DEA所用，并通过环境变量的形式传递给应用实例的启动脚本。其中VCAP\_APP\_PORT和PORT都是warden container开启那个端口号。 分析完了StartupScriptGenerator的三个参数，需要进入StartupScriptGenerator类的generate方法：

    def generate  
        script = \[\]  
        script << "umask 077"  
        script << @system\_envs  
        script << EXPORT\_BUILDPACK\_ENV\_VARIABLES\_SCRIPT  
        script << @user\_envs
        script << "env > logs/env.log"  
        script << START\_SCRIPT % @start\_command  
        script.join("\\n")  
    end  
    

以上的代码即为创建启动脚本的过程。其中@system\_envs为之前分析的env.exported\_system\_environment\_variables, @user\_envs为env.exported\_user\_environment\_variables。还有两个脚本是EXPORT\_BUILDPACK\_ENV\_VARIABLES\_SCRIPT和START\_SCRIPT，EXPORT\_BUILDPACK\_ENV\_VARIABLES\_SCRIPT脚本的代码如下，其意为执行某路径下的所有sh脚本：

    EXPORT\_BUILDPACK\_ENV\_VARIABLES\_SCRIPT = strip\_heredoc(<<-BASH).freeze  
       unset GEM\_PATH  
       if \[ -d app/.profile.d \]; then  
         for i in app/.profile.d/\*.sh; do  
            if \[ -r $i \]; then  
               . $i  
            fi  
         done  
         unset i  
       fi  
    BASH  
    

而START\_SCRIPT代码如下：

    START\_SCRIPT = strip\_heredoc(<<-BASH).freeze  
       DROPLET\_BASE\_DIR=$PWD
       cd app  
       (%s) > >(tee $DROPLET\_BASE\_DIR/logs/stdout.log) 2> >(tee $DROPLET\_BASE\_DIR/logs/stderr.log >&2) &  
       STARTED=$!  
       echo "$STARTED" >> $DROPLET\_BASE\_DIR/run.pid  
    
       wait $STARTED  
    BASH  
    

以上即创建完了start\_script，回到instance.rb中的promise\_start方法中，即执

    response = container.spawn(start\_script, container.resource\_limits(self.file\_descriptor\_limit, NPROC\_LIMIT))  
    

即完成应用实例的启动工作。详情可以进入container类中的spawn方法实现。

**转载请注明出处。**
------------

这篇文档更多出于我本人的理解，肯定在一些地方存在不足和错误。希望本文能够对接触Cloud Foundry v2中dea\_ng如何启动应用实例的人有些帮助，如果你对这方面感兴趣，并有更好的想法和建议，也请联系我。