---
title: VasSonic 源码解析
date: 2017-09-15 19:30:09
categories:
- Foo
- Bar
- Baz
tags:
-Android
---
[0]: https://segmentfault.com/img/bVUPwi?w=1146&h=210
[1]: https://segmentfault.com/img/bVUPwl?w=950&h=156
[2]: https://segmentfault.com/img/bVUPww?w=1592&h=342
[3]: https://segmentfault.com/img/bVUPwE?w=1336&h=244
[4]: https://segmentfault.com/img/bVUPwG?w=2016&h=619
[5]: https://segmentfault.com/img/bVUPxd?w=1608&h=644
[6]: https://segmentfault.com/img/bVUPxl?w=1578&h=524
[7]: https://segmentfault.com/img/bVUPxs?w=1985&h=1803
[8]: https://segmentfault.com/img/bVUPxv?w=2271&h=851
[9]: https://segmentfault.com/img/bVUPxw?w=1905&h=851
[10]: https://segmentfault.com/img/bVUPxF?w=1156&h=72
[11]: https://segmentfault.com/img/bVUPxI?w=968&h=130
[12]: https://segmentfault.com/img/bVUPxK?w=688&h=122
[13]: https://segmentfault.com/img/bVUPxO?w=1092&h=46
[14]: https://segmentfault.com/img/bVUPxw?w=1905&h=851
[15]: https://segmentfault.com/img/bVUPxV?w=736&h=154
[16]: https://segmentfault.com/img/bVUPya?w=1790&h=216
[17]: https://segmentfault.com/img/bVUPye?w=1270&h=600
#VasSonic源码解析
>VasSonic是腾讯推出的为了提高H5页面首屏加载速度而推出的高性能Hybrid框架，目前广泛应用在QQ商城等Hybrid界面中，以提高用户体验。</BR>
GitHub:<https://github.com/Tencent/VasSonic>

#一.实现原理
几乎所有的Hybrid界面都以WebView界面为载体，H5界面加载的时间主要消耗在在WebView初始化、网络请求、WebView渲染三个部分。WebView初始化与WebView渲染均是100ms的时间量级，其中最主要的时间瓶颈在网络请求上，尤其是弱网情况下，其消耗时间将达到s级，导致H5界面白屏时间较长或无法正常打开。
为了解决这个问题，WebView中提供了</br>

	public WebResourceResponse shouldInterceptRequest(WebView view, String url)
函数负责拦截并加载html中用到的资源文件，以此为基础，引出VasSonic的核心思想并行加载，本地缓存，模板变更。
整体的思路便是利用WebView初始化的时间并行的进行网络请求，并利用缓存进行预加载，网络请求完成后再把变化的部分返回给浏览器利用JS进行数据变更。
</br></br>
#二.核心流程SonicSession
整个源码我们应以SonicSession文件为切入点，它管理整个并行流程，负责获取缓存，网络请求，并以向外暴露抽象方法的方式交给子类处理错误处理，预加载，模板变更，并提供与JS的唯一交互出口，下面我们具体走一遍它的流程。
当WebView启动初始化前，便会调用SonicSession的start方法，然后会在子线程调用runSonicFlow方法，开始并行处理。
![0][0]
在runSonicFlow中，我们会尝试获取缓存，并交由子类的handleLocalHtml进行处理，预加载就可以在这里处理。
![1][1]
而后会运行handleFlow_connection方法，首先他会获取SessionConnect，之后会添加本地缓存的一些信息从而获取变化后的网页，进行connect操作后，获取到responseCode，会交给SonicSession的子类进行hanldeFlow_304,handleFlow_HttpError,handleFlow_ServiceUnavailable一系列错误处理。
之后会判断缓存是否为空，缓存为空的回话会交由子类的handleFlow_FirstLoad处理，不为空的话回判断是Template变化的话会交由子类的handleFlow_TemplateChange处理，Data变化的话会交由子类的handleFlow_DataChange处理。
![2][2]
可见，SonicSession主要向子类在几个关键的节点上暴露方法，获取缓存后子类调用handleLocalHtml处理，网络连接结束后会根据错误码等信息交由子类的hanldeFlow_304, andleFlow_HttpError, handleFlow_ServiceUnavailable 进行一系列错误处理，获取网络请求成功后，如果缓存为空，会调用子类的handleFlowFirstLoad处理，缓存不为空的话，会根据结果的header信息判断是模板变化还是数据变化，分别交由handleFlow_TemplateChange，handleFlow_DataChange处理。

在SonicSession中，同样会提供一些方法用于各种事件的回调。
SessionClient创建完成后调用OnClientReady方法，这个方法标识着WebView已经初始化完成。
当WebView开始加载资源后，会被shouldInterceptRequest方法拦截，会调用我们的OnClientRequestResource方法，标识着WebView开始加载数据。
![3][3]
当WebView渲染完成，会调用OnWebReady方法，其中会携带着JS的回调方法，最后会通过setResult调用H5的回调方法。
![4][4]
</br></br>
#三.两种SonicSesion的具体实现
由上面对SonicSession的分析可知，子类SonicSession所需要关注的点有
调用loadUrl或loadUrlWithBaseUrl启动浏览器加载流程。
赋值pengdingWebResourceStream，并返回给浏览器解析。
模板和Data变化通知给浏览器做相应处理。
拦截资源实现加载。
调用setResult进行浏览器回调。
###1.StandardSonicSession
基于安全性上的考量，StandardSonicSession模式下仅支持loadUrl。
在浏览器准备完成后，我们就可以直接调用loadUrl，所以在OnClientReady中，我们直接就可以调用loadUrl。

关于pendingWebResourceStream的赋值，在上面一条线的流程中完成。 
WebView支持边加载边渲染的特性，我们可以将流传进去后，继续进行写操作，于是定义了SonicSessionStream，之后我们会介绍到。
所以需要判断数据是否接受完成，对赋值操作做不同的处理，然后再做保存数据的操作。
![5][5]
![6][6]
下面我们看一下StandardSession的流程究竟是怎么处理的，哪些地方对pendingWebResourceStream进行了操作。
![7][7]
在上面的流程图中可以看到会根据模板与Data的变化进行不同的处理。
由上面的分析可知WebView在loadUrl后，可在onClientRequestResource方法中对浏览器资源的加载进行拦截，注意！这个方法内只会拦截我们的html文件，进行文件流的传入，资源文件并不会拦截。这个方法做了对网络流的等待，等待pengdingWebResourceStream有值之后，就会将流返回给浏览器进行加载渲染。
setResul在onWebReady中需吊用一次，在这里我们确定回调方法。
以后每次流程发生转变的时候需调用一次，可以帮我们明确每一次请求走的流程，更主要的是当Data变化时，我们可以调用浏览器的回调方法。
###2.QuickSonicSession
与Standard核心的不同点便在于可以通过loadBaseUrl实现加在缓存的html字符串，速度上有一定优势，但安全性上有一定问题，经实测速度优势不明显。
在实现上，和Standard模式我们需要关注变化的点主要就是不能在OnClientReady中不能直接调用loadUrl，需要在有缓存的时候loadBaseUrl，然后自己构造header，其他变化不大。
</br></br>
#四.具体流程
下面我们梳理下具体的流程，首先我们展现的是初始化的流程。
![8][8]
下面展现的是SonicSession start后的流程。
![9][9]
</br></br>
#五.具体问题处理
+ 不同线程间等待如何调度与通信
网络请求所在的业务在Sonic子线程中，主要通过主线程的handler与WebView所在的主线程通信。
![10][10]
![11][11]
![12][12]
WebView所在的主线程主要通过控制Atomic变量对子线程进行控制。
![13][13]
![14][14]
当WebView加载完成，网络请求的结果还未赋值时，将通过同步锁的方式等待网络请求的结果。
![15][15]
其实WebView是支持边加载边渲染的特性的，只要将数据流传递给WebView即可，于是提供了SonicSessionStream支持这种特性，下面会介绍到。

+ 如何做网络流和内存流的桥接
可以看到网络数据完成时，直接将数据放入Stream中，而未完成时，则创建了SonicSessionStream。
![16][16]
其中上面的outputStream代表着已读取到内存中的memStream，responseStream则代表着仍未读取的netStream。
![17][17]
通过重写SonicSeesionStream的read方法即可实现桥接。

+ 如何做局部刷新
当缓存数据网页加载完成，即调用onWebReady后，会通过javascriptInterface将js的回调方法返回给App。
当服务器想通知客户端局部刷新时，会通过头部的template-tag通知，并返回data的json。
通过比较本地的data与新data返回差异data，并将结果赋值给pendingDiffData。
然后会通过setResult调用js的回调方法，完成局部刷新过程。

+ 缓存文件储存
进行缓存时，会计算返回结果的sha1值，并存入sp中。
获取缓存时，会从sp中读取文件的sha1值进行比对。

+ head管理（跨域问题）

