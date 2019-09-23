---
title: "Grpc授权挖坑"
categories:
  - Tech
tags:
  - Grpc
  - SSL
---

###跳坑Grpc
最新项目本来选择继承Thrift的坑。   
然后IOS强制要求https。这乃坑一。        
然后Thrift有巨坑。（Java框架的实现。Python2.3的Lib生成的库既然还有关键字问题。）。这乃坑二。    
第三个是Protobuf+Grpc写的贼厉害（Http2+Spdy+SSL+Android/IOS等各种成熟生态系统）。


然后就是常说的技术选型不动脑。填坑就得填到老

###FirstDay：PerfectSSL
完美授权方案。
Client端和IOS端都提供了基于SSL授权加密的比较完美的方案。在官方文档都已经写出来。但是没有Android的。当时也没有想太多。就直接拿来用

服务端
```

Server server = ServerBuilder.forPort(8443)
    // Enable TLS
    .useTransportSecurity(certChainFile, privateKeyFile)
    .addService(TestServiceGrpc.bindService(serviceImplementation))
    .build();
server.start();

```

IOS端
```
#import <GRPCClient/GRPCCall.h>

#import <GRPCClient/GRPCCall+ChannelArg.h>

#import <GRPCClient/GRPCCall+ChannelCredentials.h>

+(NSString*)host{


    NSString *kHostAddress = @"grpc.friddle.me:443"; //test

    return kHostAddress;

} 
0x01  在程序启动的时候开始连接服务器
+(void)connectServer{


    NSString *certsPath = [[NSBundle mainBundle] pathForResource:@"server" ofType:@"pem"];

    NSError *error;

    NSString *contentInUTF8 = [NSString stringWithContentsOfFile:certsPath

                                                        encoding:NSASCIIStringEncoding

                                                           error:&error];



    NSError *err = nil;

    BOOL ss = [GRPCCall setTLSPEMRootCerts:contentInUTF8

                            withPrivateKey:nil

                             withCertChain:nil

                                   forHost:[[self class] host]

                                     error:&err];



    NSLog(@"%d:%@", ss, err);
}


0x10  初始化grpc服务，调用接口
_service = [[SNIndexService alloc] initWithHost:gRPCHost];  //gRPCHost为0x00中的host

-(void)fetchFeedComplete:(void(^)(BOOL isSuccess))complete{

    SNFeedRequest *req = [[SNFeedRequest alloc] init];

    req.header = [GRPCHelper grpcHeader];

    req.page = self.feedPage;



    req.productPage = [[SNPageMessage alloc] init];

    req.productPage.showCount = 3;

    req.productPage.currentPage = 0;



    [_service feedWithRequest:req handler:^(SNFeedResponse * _Nullable response, NSError * _Nullable error) {

        ZLLog(@"%@", error);

        ZLLog(@"%@", response);

        complete(error==nil?YES:NO);

    }];

}
```

Java端
```

ManagedChannel channel = NettyChannelBuilder.forAddress("overridehost", 50052)  .sslContext(GrpcSslContexts.forClient().trustManager(new File("grpc.crt")).overrideAuthority("overridehost").build()).build();
        
```

然后？
怎么没有Android的SSL授权文档。好奇怪。不是说支持Android吗。
然后在别人的博客上找到了https的授权方式。一般是用Okhttp的方式。但是调用方法并不是Grpc的那套方法。
而是okhttp固有的ssl调用方法。
但是好说歹说弄上去了
```
    public void test_Https() throws Exception {
        ManagedChannel channel= OkHttpChannelBuilder.forAddress("grpc.friddle.me",443).overrideAuthority("grpc.friddle.me").sslSocketFactory(getSslSocketFactory( new FileInputStream("src/main/resources/server.pem"))).build();
    }


    private static SSLSocketFactory getSslSocketFactory(InputStream testCa)
    throws Exception {
        if (testCa == null) {
            return (SSLSocketFactory) SSLSocketFactory.getDefault();
        }
        SSLContext context = SSLContext.getInstance("TLS");
        context.init(null, getTrustManagers(testCa), null);
        return context.getSocketFactory();
    }

    private static TrustManager[] getTrustManagers(InputStream testCa) throws Exception {
        KeyStore ks = KeyStore.getInstance(KeyStore.getDefaultType());
        ks.load(null);##整体拆分图
![image](/content/images/2016/12/DeepinScreenshot20161207092259.png)

        CertificateFactory cf = CertificateFactory.getInstance("X.509");
        X509Certificate cert = (X509Certificate) cf.generateCertificate(testCa);
        X500Principal principal = cert.getSubjectX500Principal();
        ks.setCertificateEntry(principal.getName("RFC2253"), cert);
        TrustManagerFactory trustManagerFactory =
                TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
        trustManagerFactory.init(ks);
        return trustManagerFactory.getTrustManagers();
    }
```

一切看上去很完美。但是直到测试拉了一个Android4.4以下的手机来说4.4以下访问不了。然后发现是巨坑Android4.4以下。没实现专门针对Http2+TLS的Cipher算法

Android支持的Cipher列表:"https://developer.android.com/reference/javax/crypto/Cipher.html"
更坑的是。
官方表示：4.4下的手机大家也不要心急。我们用Google Play提供了系统的补丁。你们只要强制用户用Google Play安装这个补丁就行了。。。[官方文档传送门](https://github.com/grpc/grpc-java/blob/master/SECURITY.md#transport-security-tls)中国呢？(黑人问好脸？？？？)

但是整个底层架构已经搭建完成了。箭在弦上不得不发。。真的不能脱离Grpc了。。。

###另一种取巧加密方案。修改压缩
在GoogleAuth授权不能用。AndroidSSL有问题基础上。只能挖坑了。
然后果断打了压缩(Compress)的主义。我们搞不了SSL加密。我们自己搞对称加密吧。（期待Android的混淆管用。管用。管用）

Grpc默认压缩是Zip.那么我们修改其默认压缩。自己添加压缩方案。系统调用很简单

调用
```
                   File encrypt_file = GetFileFromPath(authConfig.getEncryptionFile());
            DES3Compressor compressor = new DES3Compressor(encrypt_file);
            OptionUtils.registryCompressor(compressor);
            OptionUtils.registryDeCompressor(compressor);
            builder = builder.compressorRegistry(OptionUtils.getCompressorRegistry())
                    .decompressorRegistry(OptionUtils.getDecompressorRegistry());

```
OptionUtils
```
public class OptionUtils {

    private static CompressorRegistry compressorRegistry = CompressorRegistry.getDefaultInstance();
    private static DecompressorRegistry decompressorRegistry = DecompressorRegistry.getDefaultInstance();

    public static void registryCompressor(Codec codec) {
        compressorRegistry.register(codec);
    }

    public static CompressorRegistry getCompressorRegistry() {
        return compressorRegistry;
    }

    public static void registryDeCompressor(Codec codec) {
        decompressorRegistry = decompressorRegistry.with(codec, true);
    }

    public static DecompressorRegistry getDecompressorRegistry() {
        return decompressorRegistry;
    }
}

```

Encryption
```
省略的请自己解决
class DES3Compressor 
    @Override
    public OutputStream compress(OutputStream os) throws IOException {
        return encryption.encryption(os);
    }

    @Override
    public InputStream decompress(InputStream is) throws IOException {
        return encryption.decryption(is);
    }
```

Android端调用
```
   File encrypt_file = xxxx
    ES3Compressor compressor = new DES3Compressor(encrypt_file);
    OptionUtils.registryCompressor(compressor);
    OptionUtils.registryDeCompressor(compressor);
    BookV2.BookDetailRequest request = BookV2.BookDetailRequest.newBuilder().setBookId(20000).build();
    BookV2.BookDetailResponse response = null;

    ManagedChannel channel = NettyChannelBuilder.forAddress("localhost", 50051).usePlaintext(true)  .compressorRegistry(getCompressorRegistry()).decompressorRegistry(getDecompressorRegistry()).build();
    BookServiceGrpc.BookServiceBlockingStub blockingStub = BookServiceGrpc.newBlockingStub(channel)
            .withCompression(new DES3Compressor().getMessageEncoding());


```
一定要记得除了注册压缩算法以外。每次调用的时候还需要选择压缩名字。否则默认会走系统默认的压缩。即Zip。


####可选择压缩的坑。加返回加密的坑
    这本来感觉很完美。然后发现双方是可以默认选择压缩类型的。即客户端假如发现他不支持我们定义的压缩类型。可以选择Zip不走你的加密压缩。那么这又是个巨坑啊。
    然后看看。打主义到拦截器拉。假如选择压缩是Gzip。那我默认的把你的连接服务Reset掉。不让你调用。那拦截器怎么做了。

拒绝服务的调用代码
```     
interceptCall {
....
        Metadata.Key<String> headerKey=Metadata.Key.of("grpc-encoding",Metadata.ASCII_STRING_MARSHALLER);
        String encryption=headers.get(headerKey);
        if(encryption==null||!encryption.equals(DES3Compressor.getEncoding()))
        {
            throw Status.UNAUTHENTICATED
                    .withDescription(String.format("unsupported auth type %s", encryption))
                    .asRuntimeException();
        }
...
}
```

暂时这样做告一段落。然后等4.4慢慢的退出市场。我们就可以重新选择ssl证书加密的形式。    

