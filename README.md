# About Ysm
关于ysm逆向的一些研究信息!!

## 免责声明
本人并非ysm开发者之一，仅抱好奇态度对ysm进行逆 所有由本文档造成的法律/非法律层面的问题本人均不承担责任

## 开始
相比看这份文档的都知道ysm是一个自定义玩家模型的模组,但是关于ysm的二次/第三方开发项目却少之又少,那么,为什么呢?
在下面的内容里,我会一一赘述

## 架构
很早很早以前(2024/2前),ysm一直采用的纯java架构,即所有实现都是由jvm语言实现,并不会直接关联到C/C++上,
但是在ysm中一直有一层<del>神秘</del>混淆,其实也并不是非常神秘,<del>只是对密集恐惧症患者有些不太友好,</del>
下图便是ysm旧版本混淆的图示了</br>
<img src=/resources/ysm_legacy_decompile.png>
从这张图中可以反映出ysm加密的一部分疏漏,比如字符串没有加密,混淆只是rename了class和field等等(<del>根据我本人猜测,ysm可能是用Progurad做的混淆</del>)</br>
那么新的架构呢?在2024/2左右,沉默了接近半年的ysm终于有了点动静,但同时也带来了一个炸裂的更新,那就是Java x JNI架构,
那么这又有什么新的变化呢?</br>
在分析了新的java部分的decompile后我推测ysm仍旧用相同的办法对ysm的java部分的代码进行了混淆,
所以我按照老方法开始了第一步逆向,但这中间却出了不少插曲</br>
在我熟悉地打开那一堆o0o0oOOOo0后尝试搜索通信相关的类时,我并没有直接找到这些class而且在搜索过程中,
很多class也是缺失的,我便怀疑起了ysm加密了class的可能,于是我便试探着通过attach将那部分class给dump出来,当然这里非常顺利(在这里我使用了arthas将ysm的class
给dump了出来(dump com/elfmcys/yesstevemodel/* --limit 1000000))(PS:在新版本ysm似乎没了class加密)
在dump之后,那些类也出现了,但是随之而来的却是一个更炸裂的问题——ysm把整个缓存加载甚至是模型同步的
逻辑,写进了JNI?!(<del>不能理解搞这种逆天架构</del>)
<img src=/resources/ysm_modern_decompile.png>
<img src=/resources/ysm_modern_decompile_1.png>
以及其他触发缓存同步相关的:
<img src=/resources/ysm_modern_decompile_2.png>
<img src=/resources/ysm_modern_decompile_4.png>
<img src=/resources/ysm_modern_decompile_5.png>
<img src=/resources/ysm_modern_decompile_6.png>
<img src=/resources/ysm_modern_decompile_7.png>
可以看出,他的native不只是生成数据的活,他甚至把整个处理缓存数据包数据的都写进了native,当然更逆天的还远不仅此

## 通信协议(Layer)
PS:由于ysm的混淆,这里新版本的数据包信息可能并不是很全
Ysm的通信使用的是modloaderAPI自带的networking,即ysm的通信走的mc的custompayload数据包

### 旧版本
PS:旧版本更加详细的结构可以参考我的reobf了一半的旧版本[传送门](https://github.com/MiskaDaeve/YsmDeobfNonCompleted/blob/main/ysm_1.1.5-hotfix-2-deobf-classes.jar)
#### Forge
 Channel Name: yes_steve_model:network </br>
 Data: 
      Idx: 0 Type: Byte Desc: PacketId</br>
      Idx: 1 Type: Bytes Desc: Packet Body(在数据包类中可以看到每个数据包的具体结构)
#### Fabric
 Channel Names: yes_steve_model:[packetId] </br>
 Data: Packet Body(在数据包类中可以看到每个数据包的具体结构)

### 新版本
新版本的ysm做了跨modloader上ysm的协议兼容,所以fabric和forge版本的数据包框架结构变为了一个,当然他们也打乱了数据包id</br>
PS:在Velocity兼容性修复前,他们的channelName的path一直使用的协议版本号(1.2.1),这也是为何那些版本不兼容Velocity的原因(Velocity不允许.这些字符出现在channelName中,在实际开发中也不应当出现),
在新版本中，他们仅仅只是简单地将旧的channelName来了个replace(".","_").(<del>我感觉真的很无语,毕竟都用数据包检测版本了就不能不用版本号当channelName吗</del>)</br>
 Channel Name: yes_steve_model:1.2.1(Before Velocity compatibility fixes)</br>
 Data:
      Idx: 0 Type: Byte Desc: PacketId</br>
      Idx: 1 Type: Bytes Desc: Packet Body(在数据包类中可以看到每个数据包的具体结构)</br>
PS:他们似乎为了实现跨modloader兼容性,将forge的network的一部分给硬塞进了fabric(<del>迷惑操作+1</del>)

## 缓存同步
>**由于新版本将缓存文件生成乃至模型加载的逻辑写入了native,故该段所涉及到的均为旧版ysm的逆向内容**

PS:关于缓存同步,在ysm中一直是谜一样的存在,由于ysm使用了一些奇奇怪怪的序列化方式导致了跨版本的缓存生成
十分困难(他们使用了Gson将模型中的json一些json文件反序列化回了被混淆的javaClass后又使用了ObjectOutputStream序列化回去才写入的缓存,同时这也是所有修复BleedingPipe的mod与ysm旧版本冲突的原因)

缓存同步的逻辑如下:
1. S -> C : Model reload(pktId: 2)
2. C -> S : Owned cache list(pktId: 0)
3. S -> C : Cache hit(If md5 contained, pktId: 3) ...
4. S -> C : Cache data(Including password data(48 bytes), pktId:1) ...

在同步缓存开始时,客户端会在它的ysm worker线程里提交一些任务,大致如下: </br>
```java
    AsyncExectuor.INSTANCE.execute(() -> {
        while(ClientCacheManager.passwordData == null){
            Thread.sleep(100);
        }

        ClientCacheManager.continueSync(cacheData);
    });
````
但是!,他们并没有注意线程安全的问题,这个passwordData字段没有volatile修饰,这意味着这里可能会存在线程可见性问题.
同时,在我在结构中提到的那个md5utils中的MessageDigest,也是在这多个线程间共享的,这意味着在检查md5时,必定会因为线程安全导致一些问题,就比如logs里那一堆密钥错误问题,这是因为ysm的一些缓存加密的密钥是根据文件的md5 digest生成的,而线程不安全会使得MessageDigest无法生成正确的digest</br>
同时,ysm并未对缓存数据进行切片发送,这也导致部分缓存数据超出mc数据包编码/解码器的长度限制从而导致玩家被踢出服务器,详见:[#14](https://github.com/TartaricAcid/ysm/issues/14)

关于新版本部分内容:
新版本的缓存同步是在ysm "握手"阶段后进行的,但是由于整个数据包乃至发包调用都是在native中完成的,所以这里只能追溯到"握手"阶段</br>
大致如下:
1. S -> C : Handshake(pktId: 51)
2. C -> S : Handshake Reply(pkt: 52)
3. S -> C && C -> S Cache sync logics

PS: 数据包是否拆分仍旧不详, 缓存文件格式仍旧不详, 同时, 如果Handshake reply的版本号对不上也不会允许客户端发包并且客户端会在几秒内做出提示
<img src=/resources/ysm_handshake_failed.png>
<img src=/resources/ysm_handshake_failed_lang.png>
<img src=/resources/ysm_handshake_failed_dec.png>
同时,我们不难看出一个bug,如果服务器在玩家进入后3秒内无应答,那么这句话还是会弹出,但是模型仍能正常同步(本人已测试)
此外,这里完全没有必要去创建一个新的线程也没有必要非得硬sleep 3s,ysm完全可以将其递归式地schedule到主线程并在超过一定模糊时常后取消掉下一次schedule,比如下面的:</br>
```java
    private static volatile long lastFirstScheduled = -1; //Because it may be called from another threads according the logic of ysm
    private static volatile boolean finishedHandshake = false;
    
    public static void notifySchedule(){
        if(lastFirstScheduled == -1){
           lastFirstScheduled = System.nanoTime();
        }
        
        if((System.nanoTime() - lastFirstScheduled) > 3000000000L){
            if(!finishedHandshake){
                //Send msg
            }
            finishedHandshake = false;
            lastFirstScheduled = -1;
            return; //Cancel next schedule
        }
        
        Minecraft.getInstance().execute(ThisClass::notifySchedule);
    }
```
同时,在ysm的NetworkRegistry中也可能存在问题,它的奇怪的CAS操作很有可能会导致跨服问题,因为在跨服端中,客户端的Channel并不会因为跨服而重新创建因为客户端到跨服端的连接是不会断开的,这也就导致已经握手过的客户端的channelVer的预期值永远不会是null
<img src=/resources/ysm_weird_cas_operation.png>
最后,ysm1.2.0以及后面的一些版本并不直接采用的channel attr进行标记,在forge上他们依赖了forgehandshake,这也就导致那个时期的ysm不兼容跨modloader

## 杂项
![图片](https://github.com/user-attachments/assets/153d9b33-82e7-4b75-b54f-a1e1ede59e4d)

![图片](https://github.com/user-attachments/assets/d035cd68-819b-4a40-9f6c-6f8532d2dc1e)

https://web.archive.org/web/20240730025323/https://github.com/TartaricAcid/ysm/issues/79
