When a mail is sent from a mail client like Foxmail to a mail server, they communicate with SMTP protocol.
In the chapter 02, we have found SMTPServerFactory. This is the start point of SMTP server.
```
@ProvidesIntoSet
InitializationOperation configureSmtp(ConfigurationProvider configurationProvider,
                                    SMTPServerFactory smtpServerFactory,
                                    SendMailHandler sendMailHandler) {
    return InitilizationOperationBuilder
        .forClass(SMTPServerFactory.class)
        .init(() -> {
            smtpServerFactory.configure(configurationProvider.getConfiguration("smtpserver"));
            smtpServerFactory.init();
            sendMailHandler.init(null);
        });
}

@PostConstruct
public void init() throws Exception {
    servers = createServers(config);
    for (AbstractConfigurableAsyncServer server: servers) {
        server.init();
    }
}

protected SMTPServer createServer() {
   return new SMTPServer(smtpMetrics);
}

@Override
protected List<AbstractConfigurableAsyncServer> createServers(HierarchicalConfiguration<ImmutableNode> config) throws Exception {
    
    List<AbstractConfigurableAsyncServer> servers = new ArrayList<>();
    List<HierarchicalConfiguration<ImmutableNode>> configs = config.configurationsAt("smtpserver");
    
    for (HierarchicalConfiguration<ImmutableNode> serverConfig: configs) {
        SMTPServer server = createServer();
        server.setDnsService(dns);
        server.setProtocolHandlerLoader(loader);
        server.setFileSystem(fileSystem);
        server.configure(serverConfig);
        servers.add(server);
    }

    return servers;
}
```
The configureSmtp function defines the initial code block of SMTPServerFactory. Guice container will automatically create 
a instance of SMTPServerFactory, then execute the initial code block. In the initial code block, init function will be invoked.
Going into the init function, in there createServers which is implemented by subclass will be invoked. Going into the createServers 
function of SMTPServerFactory, in there SMTPServer will be created. There are many SMTPServers according to smtpserver.xml, including 
No SSL/TSL server, SSL server, TSL server. When all servers are created, the init function of every SMTPServer instance is invoked. 
Taking a look at the init function of SMTPServer:
```
@PostConstruct
public final void init() throws Exception {

    if (isEnabled()) {

        buildSSLContext();
        preInit();
        frameHandlerFactory = createFrameHandlerFactory();
        bind();
        port = retrieveFirstBindedPort();

        mbeanServer = ManagementFactory.getPlatformMBeanServer();
        registerMBean();
        
        LOGGER.info("Init {} done", getServiceType());

    }

}
```
Going into the bind function directly.
```
@Override
public synchronized void bind() throws Exception {
    if (started) {
        throw new IllegalStateException("Server running already");
    }

    if (addresses.isEmpty()) {
        throw new RuntimeException("Please specify at least on socketaddress to which the server should get bound!");
    }

    ServerBootstrap bootstrap = new ServerBootstrap();
    bootstrap.channel(NioServerSocketChannel.class);

    bossGroup = bossWorker.map(count -> new NioEventLoopGroup(count, NamedThreadFactory.withName(jmxName + "-boss")));
    workerGroup = new NioEventLoopGroup(ioWorker, NamedThreadFactory.withName(jmxName + "-io"));

    bossGroup.<Runnable>map(boss -> () -> bootstrap.group(boss, workerGroup))
        .orElse(() -> bootstrap.group(workerGroup))
        .run();

    ChannelInitializer<SocketChannel> factory = createPipelineFactory();

    // Configure the pipeline factory.
    bootstrap.childHandler(factory);

    configureBootstrap(bootstrap);

    for (InetSocketAddress address : addresses) {
        Channel channel = bootstrap.bind(address).sync().channel();
        channels.add(channel);
    }

    started = true;
}
```
If you are familiar with Netty, you will be aware what is doing there. Netty is an asynchronous event-driving network application 
framework for rapid development of maintainable high performance protocol servers & clients. ChannelInboundHandlerAdapter is a base 
class which handles requests
