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