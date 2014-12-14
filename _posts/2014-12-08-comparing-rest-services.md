---
layout: post
title: "Comparing Rest Services"
modified:
categories: 
excerpt: A persormance comparision of rest services and using plain netty 
tags: [rest-express, netty, spray, spray-can, spray.io]
date: 2014-12-08T13:58:34+05:30
---

Here is a comparision of three restfull services in java and scala the reason for comparing spray in this comparision is because you can use it along with your java program.

### A RestExpress Endpoint

{% highlight java%}
public class RestExpressMain {
    private static final String SERVICE_NAME = "Test Service";
    private static final String RESPONSECODE = "\"status\":\"ok\"";

    private static final int DEFAULT_EXECUTOR_THREAD_POOL_SIZE = 8;
    private static final int SERVER_PORT = 9008;
    private static final Logger LOG = LoggerFactory.getLogger(RestExpressMain.class);
    private static final RestExpressMain INSTANCE = new RestExpressMain();

    RestExpressMain() {
    }

    public static void main(String[] args) {
        RestExpress server = null;
        try {
            server = initializeServer(args);
            server.awaitShutdown();
        } catch (IOException e) {
            LOG.info(e.getMessage());
        }
    }

    public static RestExpress initializeServer(String[] args) throws IOException {
        RestExpress server = new RestExpress()
                .setName(SERVICE_NAME)
                .setBaseUrl("http://localhost:" + SERVER_PORT)
                .setExecutorThreadCount(DEFAULT_EXECUTOR_THREAD_POOL_SIZE);
        server.uri("/ping", INSTANCE).action("ping", HttpMethod.GET).noSerialization();
        server.bind(SERVER_PORT);
        return server;
    }

    public void ping(Request request, Response response) {
        response.setBody(RESPONSECODE);
    }
}
{% endhighlight %}

###A Simple Spray-can Endpoint

{% highlight scala %}
class SprayBenckMarkService extends Actor {
  import spray.http.Uri.Path._
  import spray.http.Uri._

  def jsonResponseEntity = HttpEntity(
    contentType = ContentTypes.`application/json`,
    string = "{\"status\":\"ok\"}")

  def fastPath: Http.FastPath = {
    case HttpRequest(GET, Uri(_, _, Slash(Segment("fast-ping", Path.Empty)), _, _), _, _, _) =>
      HttpResponse(entity = "FAST-PONG!")

    case HttpRequest(GET, Uri(_, _, Slash(Segment("fast-json", Path.Empty)), _, _), _, _, _) =>
      HttpResponse(entity = jsonResponseEntity)
  }

  def receive = {
    case _: Http.Connected => sender ! Http.Register(self, fastPath = fastPath)
    case HttpRequest(GET, Uri.Path("/ping"), _, _, _) => sender ! HttpResponse(entity = "\"status\":\"ok\"")
    case HttpRequest(GET, Uri.Path("/"), _, _, _) => sender ! HttpResponse(
      entity = HttpEntity(MediaTypes.`text/html`,
        <html>
          <body>
            <h1>Tiny <i>spray-can</i> benchmark server</h1>
            <p>Defined resources:</p>
            <ul>
              <li><a href="/ping">/ping</a></li>
            </ul>
          </body>
        </html>.toString()
      )
    )
    case _: HttpRequest => sender ! HttpResponse(NotFound, entity = "Unknown resource!")
  }
}
{% endhighlight %}

### Netty Endpoint

For netty we have the server the initializer and the handler Code

####Server
{% highlight java %}
public final class NettyMain {

    static final boolean SSL = System.getProperty("ssl") != null;
    static final int PORT = Integer.parseInt(System.getProperty("port", SSL? "8443" : "8080"));

    public static void main(String[] args) throws Exception {
        // Configure SSL.
        final SslContext sslCtx;
        if (SSL) {
            SelfSignedCertificate ssc = new SelfSignedCertificate();
            sslCtx = SslContext.newServerContext(ssc.certificate(), ssc.privateKey());
        } else {
            sslCtx = null;
        }
        //server Configuration
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup);
            b.channel(NioServerSocketChannel.class);

                b.handler(new LoggingHandler(LogLevel.INFO));


            b.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
            b.option(ChannelOption.SO_REUSEADDR, true);
            b.option(ChannelOption.SO_KEEPALIVE, true);
            b.option(ChannelOption.SO_RCVBUF, 52428800);
            b.option(ChannelOption.SO_LINGER, 0);

            b.childHandler(new HttpHelloWorldServerInitializer(sslCtx));
            b.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
            b.childOption(ChannelOption.TCP_NODELAY, true);

            Channel ch = b.bind(PORT).sync().channel();

            System.err.println("Open your web browser and navigate to " +
                    (SSL? "https" : "http") + "://127.0.0.1:" + PORT + '/');

            ch.closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
{% endhighlight %}

####Handler
{% highlight java %}
public class HttpHelloWorldServerHandler extends ChannelInboundHandlerAdapter {
    private static final byte[] CONTENT = { 'H', 'e', 'l', 'l', 'o', ' ', 'W', 'o', 'r', 'l', 'd' };

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        ctx.flush();
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        if (msg instanceof HttpRequest) {
            HttpRequest req = (HttpRequest) msg;

            if (HttpHeaders.is100ContinueExpected(req)) {
                ctx.write(new DefaultFullHttpResponse(HTTP_1_1, CONTINUE));
            }
            boolean keepAlive = HttpHeaders.isKeepAlive(req);
            FullHttpResponse response = new DefaultFullHttpResponse(HTTP_1_1, OK, Unpooled.wrappedBuffer(CONTENT));
            response.headers().set(CONTENT_TYPE, "text/plain");
            response.headers().set(CONTENT_LENGTH, response.content().readableBytes());

            if (!keepAlive) {
                ctx.write(response).addListener(ChannelFutureListener.CLOSE);
            } else {
                response.headers().set(CONNECTION, Values.KEEP_ALIVE);
                ctx.write(response);
            }
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
{% endhighlight %}

####Initializer
{% highlight java %}
public class HttpHelloWorldServerInitializer extends ChannelInitializer<SocketChannel> {

    private final SslContext sslCtx;

    public HttpHelloWorldServerInitializer(SslContext sslCtx) {
        this.sslCtx = sslCtx;
    }

    @Override
    public void initChannel(SocketChannel ch) {
        ChannelPipeline p = ch.pipeline();
        if (sslCtx != null) {
            p.addLast(sslCtx.newHandler(ch.alloc()));
        }
        p.addLast(new HttpServerCodec());
        p.addLast(new HttpHelloWorldServerHandler());
    }
}
{% endhighlight %}

