> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [oacia.dev](https://oacia.dev/douyin-6shen-init/)

> # 前言 抖音六神算法，一听这名字就超 cooool 的，正好最近也没有什么别的事情，于是就趁着周末来逆一逆抖音 APP, 在分析六神算法的过程中，它的 java 层的字符串加密算法我感觉很有意思，于是......

抖音六神算法，一听这名字就超 cooool 的，正好最近也没有什么别的事情，于是就趁着周末来逆一逆抖音 APP, 在分析六神算法的过程中，它的 java 层的字符串加密算法我感觉很有意思，于是就写了这篇文章来记录一下～

本文所分析的抖音版本为 `300600`

打开 app 之后用 repable 抓包，抖音直接显示无网络，所以想都不用想，肯定是有 ssl pinning，但是常规的过 ssl pinning 的代码没有用，因为抖音和其他 app 不一样，它的 ssl 校验是在 native 层的

hook apk 打开的 so 库，一般网络库的加载都是在最前面的，在 `libsscronet.so` 找到 ssl 相关的函数

![](https://oacia.dev/douyin-6shen-init/image-20240801003621820.png)

这一块应该是在 so 中用自定义的函数去进行 ssl 的验证

搜索字符串发现有很多 `certificate` , 那大概率就是在这个 so 里面进行 `ssl pinning` 的

![](https://oacia.dev/douyin-6shen-init/image-20240801004717078.png)

看了一下导入函数，感觉 `SSL_CTX_set_custom_verify` 有点像校验的样子

> SSL_CTX_set_custom_verify 证书校验异步操作，**相比 OpenSSL，BoringSSL 单独将认证证书的流程拿出来从而提供更详细的认证控制**

![](https://oacia.dev/douyin-6shen-init/image-20240801004045357.png)

再细看一下这个函数

<table><tbody><tr><td data-num="1"></td><td><pre>void SSL_CTX_set_custom_verify(
</pre></td></tr><tr><td data-num="2"></td><td><pre>    SSL_CTX *ctx, int mode,
</pre></td></tr><tr><td data-num="3"></td><td><pre>    enum ssl_verify_result_t (*callback)(SSL *ssl, uint8_t *out_alert)) {
</pre></td></tr><tr><td data-num="4"></td><td><pre>  ctx-&gt;verify_mode = mode;
</pre></td></tr><tr><td data-num="5"></td><td><pre>  ctx-&gt;custom_verify_callback = callback;
</pre></td></tr><tr><td data-num="6"></td><td><pre>}
</pre></td></tr></tbody></table>

其中的 mode 用来验证客户端，mode 的值为 1 进行验证，但是如果取 0 就是不验证

<table><tbody><tr><td data-num="1"></td><td></td></tr><tr><td data-num="2"></td><td></td></tr><tr><td data-num="3"></td><td></td></tr><tr><td data-num="4"></td><td><pre>#define SSL_VERIFY_NONE 0x00
</pre></td></tr><tr><td data-num="5"></td><td></td></tr><tr><td data-num="6"></td><td></td></tr><tr><td data-num="7"></td><td></td></tr><tr><td data-num="8"></td><td></td></tr><tr><td data-num="9"></td><td></td></tr><tr><td data-num="10"></td><td><pre>#define SSL_VERIFY_PEER 0x01
</pre></td></tr><tr><td data-num="11"></td><td></td></tr><tr><td data-num="12"></td><td></td></tr><tr><td data-num="13"></td><td></td></tr><tr><td data-num="14"></td><td></td></tr><tr><td data-num="15"></td><td><pre>#define SSL_VERIFY_FAIL_IF_NO_PEER_CERT 0x02
</pre></td></tr><tr><td data-num="16"></td><td></td></tr><tr><td data-num="17"></td><td></td></tr><tr><td data-num="18"></td><td></td></tr><tr><td data-num="19"></td><td><pre>#define SSL_VERIFY_PEER_IF_NO_OBC 0x04
</pre></td></tr></tbody></table>

所以我们可以直接用 frida patch 第二个参数的值为 0

<table><tbody><tr><td data-num="1"></td><td><pre>function hook_dlopen(soName = '') {
</pre></td></tr><tr><td data-num="2"></td><td><pre>    Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"),
</pre></td></tr><tr><td data-num="3"></td><td><pre>        {
</pre></td></tr><tr><td data-num="4"></td><td><pre>            onEnter: function (args) {
</pre></td></tr><tr><td data-num="5"></td><td><pre>                var pathptr = args[0];
</pre></td></tr><tr><td data-num="6"></td><td><pre>                if (pathptr !== undefined &amp;&amp; pathptr != null) {
</pre></td></tr><tr><td data-num="7"></td><td><pre>                    var path = ptr(pathptr).readCString();
</pre></td></tr><tr><td data-num="8"></td><td><pre>                    
</pre></td></tr><tr><td data-num="9"></td><td><pre>                    if (path.indexOf(soName) &gt;= 0) {
</pre></td></tr><tr><td data-num="10"></td><td><pre>                        this.is_can_hook = true;
</pre></td></tr><tr><td data-num="11"></td><td><pre>                    }
</pre></td></tr><tr><td data-num="12"></td><td><pre>                }
</pre></td></tr><tr><td data-num="13"></td><td><pre>            },
</pre></td></tr><tr><td data-num="14"></td><td><pre>            onLeave: function (retval) {
</pre></td></tr><tr><td data-num="15"></td><td><pre>                if (this.is_can_hook) {
</pre></td></tr><tr><td data-num="16"></td><td><pre>                    ssl_bypass();
</pre></td></tr><tr><td data-num="17"></td><td><pre>                    
</pre></td></tr><tr><td data-num="18"></td><td><pre>                }
</pre></td></tr><tr><td data-num="19"></td><td><pre>            }
</pre></td></tr><tr><td data-num="20"></td><td><pre>        }
</pre></td></tr><tr><td data-num="21"></td><td><pre>    );
</pre></td></tr><tr><td data-num="22"></td><td><pre>}
</pre></td></tr><tr><td data-num="23"></td><td></td></tr><tr><td data-num="24"></td><td><pre>function ssl_bypass(){
</pre></td></tr><tr><td data-num="25"></td><td><pre>    let SSL_CTX_set_custom_verify = Module.getExportByName('libsscronet.so', 'SSL_CTX_set_custom_verify');
</pre></td></tr><tr><td data-num="26"></td><td><pre>    console.log('start hook SSL_CTX_set_custom_verify')
</pre></td></tr><tr><td data-num="27"></td><td><pre>    if (SSL_CTX_set_custom_verify != null) {
</pre></td></tr><tr><td data-num="28"></td><td><pre>        Interceptor.attach(SSL_CTX_set_custom_verify, {
</pre></td></tr><tr><td data-num="29"></td><td><pre>            onEnter: function (args) {
</pre></td></tr><tr><td data-num="30"></td><td><pre>                console.log('SSL_CTX_set_custom_verify mode: ',args[1]);
</pre></td></tr><tr><td data-num="31"></td><td><pre>                args[1] = ptr(0);
</pre></td></tr><tr><td data-num="32"></td><td><pre>                console.log('patch mode from 1 to 0 to bypass ssl')
</pre></td></tr><tr><td data-num="33"></td><td><pre>            },
</pre></td></tr><tr><td data-num="34"></td><td><pre>            onLeave: function (retval) {
</pre></td></tr><tr><td data-num="35"></td><td></td></tr><tr><td data-num="36"></td><td><pre>            }
</pre></td></tr><tr><td data-num="37"></td><td><pre>        });
</pre></td></tr><tr><td data-num="38"></td><td><pre>    }
</pre></td></tr><tr><td data-num="39"></td><td><pre>}
</pre></td></tr><tr><td data-num="40"></td><td></td></tr><tr><td data-num="41"></td><td></td></tr><tr><td data-num="42"></td><td><pre>setImmediate(hook_dlopen, "libsscronet.so")
</pre></td></tr></tbody></table>

抓到包啦

![](https://oacia.dev/douyin-6shen-init/image-20240801010106216.png)

在请求头中出现的六个 `x-` 带头的参数就是六神算法

![](https://oacia.dev/douyin-6shen-init/image-20240818142610556.png)

接下来就是去定位这些参数的位置咯，该怎么定位呢？

其实很简单，按照正向安全开发的经验来说，对于大型的 APP, 这种关键参数所涉及的算法基本上是不可能在 java 层中出现的，他们通常极大概率是把加密算法下沉到 native 层中去调用，所以说，我们就只需要找 so 就可以啦，而这种含有关键算法的 so 一般都有一个共性，就是 so 中不会出现静态注册的 java 方法，并且 so 被混淆的非常滴厉害，各种花指令寄存器跳转啥的通通来一遍让我们一打开 IDA 反编译连伪代码都看不了，找到这类 so，那就离关键算法不远了，而其他无关紧要的 so 基本不会做防护，要是也加上高强度的混淆，可能会出 bug 不说，还会拖慢 APP 启动速度影响用户体验

而小型的 APP, 由于很少会在安全上花功夫，所以说我们在 java 层中搜索参数名称，一般就可以找到参数所在的位置了

而在之前我们已经知道进行 `ssl pinning` 的 so 库的名称是 `libsscronet.so` , 那对参数进行加密的 so 库一般就在网络库的附近加载，感觉 `libmetasec_ml.so` 这个 so 挺像的

![](https://oacia.dev/douyin-6shen-init/image-20240818142932419.png)

一用 IDA 反编译 so 之后引入眼帘的就是让人感到亲切的 `BR` 寄存器跳转！

![](https://oacia.dev/douyin-6shen-init/image-20240818143106335.png)

接下来看看这个 so 动态注册了什么方法咯

<table><tbody><tr><td data-num="1"></td><td><pre>function getModuleInfoByPtr(fnPtr) {
</pre></td></tr><tr><td data-num="2"></td><td><pre>  var modules = Process.enumerateModules();
</pre></td></tr><tr><td data-num="3"></td><td><pre>  var modname = null,
</pre></td></tr><tr><td data-num="4"></td><td><pre>      base = null;
</pre></td></tr><tr><td data-num="5"></td><td><pre>  modules.forEach(function (mod) {
</pre></td></tr><tr><td data-num="6"></td><td><pre>    if (mod.base &lt;= fnPtr &amp;&amp; fnPtr.toInt32() &lt;= mod.base.toInt32() + mod.size) {
</pre></td></tr><tr><td data-num="7"></td><td><pre>      modname = mod.name;
</pre></td></tr><tr><td data-num="8"></td><td><pre>      base = mod.base;
</pre></td></tr><tr><td data-num="9"></td><td><pre>      return false;
</pre></td></tr><tr><td data-num="10"></td><td><pre>    }
</pre></td></tr><tr><td data-num="11"></td><td><pre>  });
</pre></td></tr><tr><td data-num="12"></td><td><pre>  return [modname, base];
</pre></td></tr><tr><td data-num="13"></td><td><pre>}
</pre></td></tr><tr><td data-num="14"></td><td><pre>function hook_registNatives() {
</pre></td></tr><tr><td data-num="15"></td><td><pre>  var env = Java.vm.getEnv();
</pre></td></tr><tr><td data-num="16"></td><td><pre>  var handlePointer = env.handle.readPointer();
</pre></td></tr><tr><td data-num="17"></td><td><pre>  console.log("handle: " + handlePointer);
</pre></td></tr><tr><td data-num="18"></td><td><pre>  var nativePointer = handlePointer.add(215 * Process.pointerSize).readPointer();
</pre></td></tr><tr><td data-num="19"></td><td><pre>  console.log("register: " + nativePointer);
</pre></td></tr><tr><td data-num="20"></td><td><pre>  
</pre></td></tr><tr><td data-num="21"></td><td><pre>   typedef struct {
</pre></td></tr><tr><td data-num="22"></td><td><pre>      const char* name;
</pre></td></tr><tr><td data-num="23"></td><td><pre>      const char* signature;
</pre></td></tr><tr><td data-num="24"></td><td><pre>      void* fnPtr;
</pre></td></tr><tr><td data-num="25"></td><td><pre>   } JNINativeMethod;
</pre></td></tr><tr><td data-num="26"></td><td><pre>   jint RegisterNatives(JNIEnv* env, jclass clazz, const JNINativeMethod* methods, jint nMethods)
</pre></td></tr><tr><td data-num="27"></td><td><pre>   */
</pre></td></tr><tr><td data-num="28"></td><td></td></tr><tr><td data-num="29"></td><td><pre>  Interceptor.attach(nativePointer, {
</pre></td></tr><tr><td data-num="30"></td><td><pre>    onEnter: function onEnter(args) {
</pre></td></tr><tr><td data-num="31"></td><td><pre>      var env = Java.vm.getEnv();
</pre></td></tr><tr><td data-num="32"></td><td><pre>      var p_size = Process.pointerSize;
</pre></td></tr><tr><td data-num="33"></td><td><pre>      var methods = args[2];
</pre></td></tr><tr><td data-num="34"></td><td><pre>      var methodcount = args[3].toInt32();
</pre></td></tr><tr><td data-num="35"></td><td><pre>      var name = env.getClassName(args[1]);
</pre></td></tr><tr><td data-num="36"></td><td><pre>      console.log("==== class: " + name + " ====");
</pre></td></tr><tr><td data-num="37"></td><td><pre>      console.log("==== methods: " + methods + " nMethods: " + methodcount + " ====");
</pre></td></tr><tr><td data-num="38"></td><td></td></tr><tr><td data-num="39"></td><td><pre>      for (var i = 0; i &lt; methodcount; i++) {
</pre></td></tr><tr><td data-num="40"></td><td><pre>        var idx = i * p_size * 3;
</pre></td></tr><tr><td data-num="41"></td><td><pre>        var fnPtr = methods.add(idx + p_size * 2).readPointer();
</pre></td></tr><tr><td data-num="42"></td><td><pre>        var infoArr = getModuleInfoByPtr(fnPtr);
</pre></td></tr><tr><td data-num="43"></td><td><pre>        var modulename = infoArr[0];
</pre></td></tr><tr><td data-num="44"></td><td><pre>        var modulebase = infoArr[1];
</pre></td></tr><tr><td data-num="45"></td><td><pre>        var logstr = "name: " + methods.add(idx).readPointer().readCString() + ", signature: " + methods.add(idx + p_size).readPointer().readCString() + ", fnPtr: " + fnPtr + ", modulename: " + modulename + " -&gt; base: " + modulebase;
</pre></td></tr><tr><td data-num="46"></td><td></td></tr><tr><td data-num="47"></td><td><pre>        if (null != modulebase) {
</pre></td></tr><tr><td data-num="48"></td><td><pre>          logstr += ", offset: " + fnPtr.sub(modulebase);
</pre></td></tr><tr><td data-num="49"></td><td><pre>        }
</pre></td></tr><tr><td data-num="50"></td><td></td></tr><tr><td data-num="51"></td><td><pre>        console.log(logstr);
</pre></td></tr><tr><td data-num="52"></td><td><pre>      }
</pre></td></tr><tr><td data-num="53"></td><td><pre>    }
</pre></td></tr><tr><td data-num="54"></td><td><pre>  });
</pre></td></tr><tr><td data-num="55"></td><td><pre>}
</pre></td></tr></tbody></table>

这里注册了 `ms.bd.c.l.a` , 它的方法签名是 `(IIJLjava/lang/String;Ljava/lang/Object;)Ljava/lang/Object;` , 并且在 so 中的偏移是 `0x126e30`

![](https://oacia.dev/douyin-6shen-init/image-20240818143547405.png)

直接对偏移 `0x126e30` 的传入传出参数进行 hook, 发现就是六神算法的生成位置～

![](https://oacia.dev/douyin-6shen-init/image-20240818144352514.png)

而在对 `ms.bd.c.l.a` 进行交叉引用后发现，这个 native 函数还有一个字符串加密的功能如图所示，通过调用 `l.a` 函数，并传入 code 为 `0x1000001` 来实现 java 层的字符串加密，接下来我们来对这个字符串加密算法进行更加深入的研究

![](https://oacia.dev/douyin-6shen-init/image-20240818162150793.png)

这个 so 的花指令主要有五类 (其实还有一类是 `CSEL-BR` 寄存器跳转混淆，但是由于字符串解密算法不涉及 `CSEL-BR` 混淆，所以此处没有给出相关的示例)

1.  `BL-BR` 寄存器跳转  
    这个寄存器跳转简单读一下汇编就可以很清楚的知道是怎么实现的啦，主要是通过一个函数获取到了当前 BL 指令下一行的内存地址，然后通过加上一个偏移使用 BR 寄存器跳转的形式跳过去，所以我们只要把中间越过的指令直接 NOP 掉就好啦
    
    ![](https://oacia.dev/douyin-6shen-init/image-20240817210246836.png)
    
2.  `B.GT-CBNZ` 垃圾指令  
    这里出现了一大段 IDA 无法解析的指令
    

![](https://oacia.dev/douyin-6shen-init/image-20240805011945451.png)

其实对于此处的汇编来说，它执行后最终的值是固定的

<table><tbody><tr><td data-num="1"></td><td><pre>MOV             W8, 
</pre></td></tr><tr><td data-num="2"></td><td><pre>MOV             W9, 
</pre></td></tr><tr><td data-num="3"></td><td><pre>STR             W8, [SP,
</pre></td></tr><tr><td data-num="4"></td><td><pre>STR             W9, [SP,
</pre></td></tr><tr><td data-num="5"></td><td><pre>LDR             W8, [SP,
</pre></td></tr><tr><td data-num="6"></td><td><pre>LDR             W9, [SP,
</pre></td></tr><tr><td data-num="7"></td><td><pre>CMP             W9, 
</pre></td></tr><tr><td data-num="8"></td><td><pre>B.GT          
</pre></td></tr><tr><td data-num="9"></td><td><pre>MOV             W9, 
</pre></td></tr><tr><td data-num="10"></td><td><pre>MADD            W8, W8, W8, W9
</pre></td></tr><tr><td data-num="11"></td><td><pre>MOV             W9, 
</pre></td></tr><tr><td data-num="12"></td><td><pre>UDIV            W9, W8, W9
</pre></td></tr><tr><td data-num="13"></td><td><pre>SUB             W9, W9, W9,LSL
</pre></td></tr><tr><td data-num="14"></td><td><pre>ADD             W8, W8, W9
</pre></td></tr><tr><td data-num="15"></td><td><pre>CBNZ            W8, loc_55318
</pre></td></tr></tbody></table>

我们可以使用 unicorn 将这段汇编运行一下看看具体的值

<table><tbody><tr><td data-num="1"></td><td></td></tr><tr><td data-num="2"></td><td><pre>from unicorn import *
</pre></td></tr><tr><td data-num="3"></td><td><pre>from unicorn.arm64_const import *
</pre></td></tr><tr><td data-num="4"></td><td><pre>from keystone import *  
</pre></td></tr><tr><td data-num="5"></td><td><pre>from capstone import *  
</pre></td></tr><tr><td data-num="6"></td><td></td></tr><tr><td data-num="7"></td><td><pre>BASE = 0x0
</pre></td></tr><tr><td data-num="8"></td><td><pre>CODE = BASE + 0x0
</pre></td></tr><tr><td data-num="9"></td><td><pre>CODE_SIZE = 0x100000
</pre></td></tr><tr><td data-num="10"></td><td><pre>STACK = 0x7F00000000
</pre></td></tr><tr><td data-num="11"></td><td><pre>STACK_SIZE = 0x100000
</pre></td></tr><tr><td data-num="12"></td><td><pre>FS = 0x7FF0000000
</pre></td></tr><tr><td data-num="13"></td><td><pre>FS_SIZE = 0x100000
</pre></td></tr><tr><td data-num="14"></td><td><pre>ks = Ks(KS_ARCH_ARM64, KS_MODE_LITTLE_ENDIAN)  
</pre></td></tr><tr><td data-num="15"></td><td><pre>uc = Uc(UC_ARCH_ARM64, UC_MODE_LITTLE_ENDIAN)  
</pre></td></tr><tr><td data-num="16"></td><td><pre>cs = Cs(CS_ARCH_ARM64, CS_MODE_LITTLE_ENDIAN)  
</pre></td></tr><tr><td data-num="17"></td><td></td></tr><tr><td data-num="18"></td><td></td></tr><tr><td data-num="19"></td><td><pre>def hook_code(uc: unicorn.Uc, address, size, user_data):
</pre></td></tr><tr><td data-num="20"></td><td><pre>    
</pre></td></tr><tr><td data-num="21"></td><td><pre>    for i in cs.disasm(CODE_DATA[address - BASE:address - BASE + size], address):
</pre></td></tr><tr><td data-num="22"></td><td><pre>        
</pre></td></tr><tr><td data-num="23"></td><td><pre>        ZF_flag = (uc.reg_read(UC_ARM64_REG_NZCV) &gt;&gt;30)&amp;1
</pre></td></tr><tr><td data-num="24"></td><td></td></tr><tr><td data-num="25"></td><td><pre>        if i.mnemonic == "b.gt":
</pre></td></tr><tr><td data-num="26"></td><td><pre>            print(f"0x{i.address}: {i.mnemonic} {i.op_str} ; ZF = {ZF_flag}")
</pre></td></tr><tr><td data-num="27"></td><td><pre>            uc.reg_write(UC_ARM64_REG_PC, address + size)
</pre></td></tr><tr><td data-num="28"></td><td><pre>        elif i.mnemonic == "cmp":
</pre></td></tr><tr><td data-num="29"></td><td><pre>            w9 = uc.reg_read(UC_ARM64_REG_W9)
</pre></td></tr><tr><td data-num="30"></td><td><pre>            print(f"0x{i.address}: {i.mnemonic} {i.op_str} ; ZF = {ZF_flag}, w9 = {hex(w9)}")
</pre></td></tr><tr><td data-num="31"></td><td><pre>        elif i.mnemonic == "cbnz":
</pre></td></tr><tr><td data-num="32"></td><td><pre>            w8 = uc.reg_read(UC_ARM64_REG_W8)
</pre></td></tr><tr><td data-num="33"></td><td><pre>            print(f"0x{i.address}: {i.mnemonic} {i.op_str} ; ZF = {ZF_flag}, w8 = {hex(w8)}")
</pre></td></tr><tr><td data-num="34"></td><td><pre>            uc.reg_write(UC_ARM64_REG_PC, address + size)
</pre></td></tr><tr><td data-num="35"></td><td><pre>        else:
</pre></td></tr><tr><td data-num="36"></td><td><pre>            print(f"0x{i.address}: {i.mnemonic} {i.op_str} ; ZF = {ZF_flag}")
</pre></td></tr><tr><td data-num="37"></td><td></td></tr><tr><td data-num="38"></td><td></td></tr><tr><td data-num="39"></td><td><pre>def hook_mem_access(uc: unicorn.Uc, type, address, size, value, userdata):
</pre></td></tr><tr><td data-num="40"></td><td><pre>    pc = uc.reg_read(UC_ARM64_REG_PC)  
</pre></td></tr><tr><td data-num="41"></td><td></td></tr><tr><td data-num="42"></td><td><pre>    print('pc:%x type:%d addr:%x size:%x' % (pc, type, address, size))
</pre></td></tr><tr><td data-num="43"></td><td><pre>    
</pre></td></tr><tr><td data-num="44"></td><td><pre>    return True
</pre></td></tr><tr><td data-num="45"></td><td></td></tr><tr><td data-num="46"></td><td></td></tr><tr><td data-num="47"></td><td><pre>def inituc(uc):
</pre></td></tr><tr><td data-num="48"></td><td><pre>    uc.mem_map(CODE, CODE_SIZE, UC_PROT_ALL)
</pre></td></tr><tr><td data-num="49"></td><td><pre>    uc.mem_map(STACK, STACK_SIZE, UC_PROT_ALL)
</pre></td></tr><tr><td data-num="50"></td><td></td></tr><tr><td data-num="51"></td><td><pre>    uc.mem_write(CODE, CODE_DATA)
</pre></td></tr><tr><td data-num="52"></td><td><pre>    uc.reg_write(UC_ARM64_REG_SP, STACK + 0x1000)
</pre></td></tr><tr><td data-num="53"></td><td></td></tr><tr><td data-num="54"></td><td><pre>    uc.hook_add(UC_HOOK_CODE, hook_code)
</pre></td></tr><tr><td data-num="55"></td><td><pre>    uc.hook_add(UC_HOOK_MEM_UNMAPPED | UC_HOOK_INTR, hook_mem_access)
</pre></td></tr><tr><td data-num="56"></td><td></td></tr><tr><td data-num="57"></td><td></td></tr><tr><td data-num="58"></td><td><pre>shellcode = '''
</pre></td></tr><tr><td data-num="59"></td><td><pre>MOV W8, #0x1B5
</pre></td></tr><tr><td data-num="60"></td><td><pre>MOV W9, #0x16F
</pre></td></tr><tr><td data-num="61"></td><td><pre>STR W8, [SP,#0x4C]
</pre></td></tr><tr><td data-num="62"></td><td><pre>STR W9, [SP,#0x48]
</pre></td></tr><tr><td data-num="63"></td><td><pre>LDR W8, [SP,#0x4C]
</pre></td></tr><tr><td data-num="64"></td><td><pre>LDR W9, [SP,#0x48]
</pre></td></tr><tr><td data-num="65"></td><td><pre>CMP W9, #0x4C
</pre></td></tr><tr><td data-num="66"></td><td><pre>B.GT #0x55318
</pre></td></tr><tr><td data-num="67"></td><td></td></tr><tr><td data-num="68"></td><td><pre>MOV W9, #1
</pre></td></tr><tr><td data-num="69"></td><td><pre>MADD W8, W8, W8, W9
</pre></td></tr><tr><td data-num="70"></td><td><pre>MOV W9, #7
</pre></td></tr><tr><td data-num="71"></td><td><pre>UDIV W9, W8, W9
</pre></td></tr><tr><td data-num="72"></td><td><pre>SUB W9, W9, W9,LSL#3
</pre></td></tr><tr><td data-num="73"></td><td><pre>ADD W8, W8, W9
</pre></td></tr><tr><td data-num="74"></td><td><pre>CBNZ W8, #0x55318
</pre></td></tr><tr><td data-num="75"></td><td><pre>'''
</pre></td></tr><tr><td data-num="76"></td><td><pre>CODE_DATA,count = ks.asm(shellcode)
</pre></td></tr><tr><td data-num="77"></td><td><pre>CODE_DATA = bytes(CODE_DATA)
</pre></td></tr><tr><td data-num="78"></td><td><pre>inituc(uc)
</pre></td></tr><tr><td data-num="79"></td><td><pre>try:
</pre></td></tr><tr><td data-num="80"></td><td><pre>    uc.emu_start(0x0, len(CODE_DATA))
</pre></td></tr><tr><td data-num="81"></td><td><pre>except Exception as e:
</pre></td></tr><tr><td data-num="82"></td><td><pre>    print(e)
</pre></td></tr></tbody></table>

输出如下，可以发现 `cmp w9, #0x4c` 将 ZF 标志位置为 1, `b.gt` 大于跳转永远成立， `cbnz` 跳转也永远成立，也就是不管怎么样，最终都会越过中间这段无用的指令，而跳转到 `0x55318` 的位置

<table><tbody><tr><td data-num="1"></td><td><pre>0x0: mov w8, 
</pre></td></tr><tr><td data-num="2"></td><td><pre>0x4: mov w9, 
</pre></td></tr><tr><td data-num="3"></td><td><pre>0x8: str w8, [sp, 
</pre></td></tr><tr><td data-num="4"></td><td><pre>0x12: str w9, [sp, 
</pre></td></tr><tr><td data-num="5"></td><td><pre>0x16: ldr w8, [sp, 
</pre></td></tr><tr><td data-num="6"></td><td><pre>0x20: ldr w9, [sp, 
</pre></td></tr><tr><td data-num="7"></td><td><pre>0x24: cmp w9, 
</pre></td></tr><tr><td data-num="8"></td><td><pre>0x28: b.gt 
</pre></td></tr><tr><td data-num="9"></td><td><pre>0x32: mov w9, 
</pre></td></tr><tr><td data-num="10"></td><td><pre>0x36: madd w8, w8, w8, w9 ; ZF = 0
</pre></td></tr><tr><td data-num="11"></td><td><pre>0x40: mov w9, 
</pre></td></tr><tr><td data-num="12"></td><td><pre>0x44: udiv w9, w8, w9 ; ZF = 0
</pre></td></tr><tr><td data-num="13"></td><td><pre>0x48: sub w9, w9, w9, lsl 
</pre></td></tr><tr><td data-num="14"></td><td><pre>0x52: add w8, w8, w9 ; ZF = 0
</pre></td></tr><tr><td data-num="15"></td><td><pre>0x56: cbnz w8, 
</pre></td></tr></tbody></table>

所以对于这种 ida 无法识别的无用指令，我们直接 nop 掉即可

3.  `F1 F6 77 FF 1F 20 03 D5 1F 20 03 D5 1F 20 03 D5` , 这个花指令主要是用来进行 so 完整性校验的，但是我们主要是进行静态分析，所以对于影响我们看汇编的指令，直接 nop 掉

![](https://oacia.dev/douyin-6shen-init/image-20240805010232627.png)

4.  `EB FF 1F D6 01 00 00 D4` , 这个被 ida 解析成了 svc 调用，实际上看看上下文根本没有给 x8 赋值的语句，所以这八字节也是花指令

![](https://oacia.dev/douyin-6shen-init/image-20240805010341218.png)

5.  `01 00 00 D4 DE 01 70 47` , 同样的，这里出现的 svc 也不是真的 svc, 是花指令，直接 patch 掉就好啦

![](https://oacia.dev/douyin-6shen-init/image-20240805010451682.png)

所以最终的去花代码如下

<table><tbody><tr><td data-num="1"></td><td><pre>import ida_segment
</pre></td></tr><tr><td data-num="2"></td><td><pre>import idautils
</pre></td></tr><tr><td data-num="3"></td><td><pre>import idc
</pre></td></tr><tr><td data-num="4"></td><td><pre>import ida_bytes
</pre></td></tr><tr><td data-num="5"></td><td><pre>from keystone import *
</pre></td></tr><tr><td data-num="6"></td><td><pre>def patch_nop(begin, end):  
</pre></td></tr><tr><td data-num="7"></td><td><pre>    while end &gt; begin:
</pre></td></tr><tr><td data-num="8"></td><td><pre>        ida_bytes.patch_bytes(begin, b'\x1F\x20\x03\xD5')
</pre></td></tr><tr><td data-num="9"></td><td><pre>        begin = begin + 4
</pre></td></tr><tr><td data-num="10"></td><td><pre>        
</pre></td></tr><tr><td data-num="11"></td><td></td></tr><tr><td data-num="12"></td><td><pre>text_seg = ida_segment.get_segm_by_name(".text")
</pre></td></tr><tr><td data-num="13"></td><td><pre>start, end = text_seg.start_ea, text_seg.end_ea
</pre></td></tr><tr><td data-num="14"></td><td></td></tr><tr><td data-num="15"></td><td></td></tr><tr><td data-num="16"></td><td></td></tr><tr><td data-num="17"></td><td></td></tr><tr><td data-num="18"></td><td><pre>pattern = ["01 00 00 D4 DE 01 70 47",
</pre></td></tr><tr><td data-num="19"></td><td><pre>           "EB FF 1F D6 01 00 00 D4",
</pre></td></tr><tr><td data-num="20"></td><td><pre>           "F1 F6 77 FF 1F 20 03 D5 1F 20 03 D5 1F 20 03 D5"]
</pre></td></tr><tr><td data-num="21"></td><td><pre>for i in range(len(pattern)):
</pre></td></tr><tr><td data-num="22"></td><td><pre>    cur_addr = start
</pre></td></tr><tr><td data-num="23"></td><td><pre>    end_addr = end
</pre></td></tr><tr><td data-num="24"></td><td><pre>    while cur_addr &lt; end_addr:
</pre></td></tr><tr><td data-num="25"></td><td><pre>        cur_addr = idc.find_binary(cur_addr, idc.SEARCH_DOWN, pattern[i])
</pre></td></tr><tr><td data-num="26"></td><td><pre>        if cur_addr == idc.BADADDR:
</pre></td></tr><tr><td data-num="27"></td><td><pre>            break
</pre></td></tr><tr><td data-num="28"></td><td><pre>        else:
</pre></td></tr><tr><td data-num="29"></td><td><pre>            print("patch flower code: " + hex(cur_addr))  
</pre></td></tr><tr><td data-num="30"></td><td><pre>            
</pre></td></tr><tr><td data-num="31"></td><td><pre>        cur_addr = idc.next_head(cur_addr)
</pre></td></tr><tr><td data-num="32"></td><td></td></tr><tr><td data-num="33"></td><td></td></tr><tr><td data-num="34"></td><td></td></tr><tr><td data-num="35"></td><td><pre>current_addr = start
</pre></td></tr><tr><td data-num="36"></td><td><pre>while current_addr &lt; end:
</pre></td></tr><tr><td data-num="37"></td><td><pre>    
</pre></td></tr><tr><td data-num="38"></td><td><pre>    if idc.print_insn_mnem(current_addr) == "BR":
</pre></td></tr><tr><td data-num="39"></td><td><pre>        BR_addr = current_addr
</pre></td></tr><tr><td data-num="40"></td><td><pre>        BL_addr,nop_count = 0,0
</pre></td></tr><tr><td data-num="41"></td><td></td></tr><tr><td data-num="42"></td><td><pre>        temp_addr = current_addr
</pre></td></tr><tr><td data-num="43"></td><td><pre>        for _ in range(4):
</pre></td></tr><tr><td data-num="44"></td><td><pre>            if idc.print_insn_mnem(temp_addr) == "BL":
</pre></td></tr><tr><td data-num="45"></td><td><pre>                BL_addr = temp_addr
</pre></td></tr><tr><td data-num="46"></td><td><pre>            elif idc.print_insn_mnem(temp_addr) == "ADD":
</pre></td></tr><tr><td data-num="47"></td><td><pre>                nop_count = idc.get_operand_value(temp_addr, 2)
</pre></td></tr><tr><td data-num="48"></td><td><pre>            temp_addr = idc.prev_head(temp_addr)
</pre></td></tr><tr><td data-num="49"></td><td><pre>        if BL_addr and nop_count:
</pre></td></tr><tr><td data-num="50"></td><td><pre>            print(f"patch BL-BR from {hex(BL_addr)} to {hex(idc.next_head(BL_addr)+nop_count)}")
</pre></td></tr><tr><td data-num="51"></td><td><pre>            patch_nop(BL_addr, idc.next_head(BL_addr) + nop_count)
</pre></td></tr><tr><td data-num="52"></td><td><pre>    
</pre></td></tr><tr><td data-num="53"></td><td><pre>    elif idc.print_insn_mnem(current_addr) == "CBNZ":
</pre></td></tr><tr><td data-num="54"></td><td><pre>        CBNZ_addr = current_addr
</pre></td></tr><tr><td data-num="55"></td><td><pre>        CBNZ_jmp_addr = idc.get_operand_value(current_addr,1)
</pre></td></tr><tr><td data-num="56"></td><td><pre>        BGT_addr = 0
</pre></td></tr><tr><td data-num="57"></td><td><pre>        temp_addr = current_addr
</pre></td></tr><tr><td data-num="58"></td><td><pre>        for _ in range(10):
</pre></td></tr><tr><td data-num="59"></td><td><pre>            if idc.print_insn_mnem(temp_addr) == "B.GT":
</pre></td></tr><tr><td data-num="60"></td><td><pre>                BGT_jmp_addr = idc.get_operand_value(temp_addr,0)
</pre></td></tr><tr><td data-num="61"></td><td><pre>                if CBNZ_jmp_addr==BGT_jmp_addr:
</pre></td></tr><tr><td data-num="62"></td><td><pre>                    print(f"patch B.GT-CBNZ junk code from {hex(CBNZ_addr+4)} to {hex(CBNZ_jmp_addr)}")
</pre></td></tr><tr><td data-num="63"></td><td><pre>                    patch_nop(CBNZ_addr+4, CBNZ_jmp_addr)
</pre></td></tr><tr><td data-num="64"></td><td><pre>                break
</pre></td></tr><tr><td data-num="65"></td><td><pre>            temp_addr = idc.prev_head(temp_addr)
</pre></td></tr><tr><td data-num="66"></td><td></td></tr><tr><td data-num="67"></td><td><pre>    current_addr = idc.next_head(current_addr)
</pre></td></tr></tbody></table>

为了防止其他的 so 对我们的干扰，所以可以写个 demo 加载这个 so，但一加载这个 so 的时候程序就崩溃了并留下了下面这一串报错，既然调用了这个类 `com.bytedance.mobsec.metasec.ml.MS` ，那肯定和算法有些许的联系

![](https://oacia.dev/douyin-6shen-init/image-20240814001314102.png)

而这个 `MS` 类，他继承了 `e0` 类

![](https://oacia.dev/douyin-6shen-init/image-20240814001439387.png)

而这个 e0 类，他又继承了 `ms.bd.c.l` 这个类，这个类就是 `libmetasec_ml.so` 唯一动态注册的函数 `a` 所在的类，在 `e0` 中，又定义了四个函数，但是他们的函数竟然啥内容都没有

<table><tbody><tr><td data-num="1"></td><td><pre>package ms.bd.c;
</pre></td></tr><tr><td data-num="2"></td><td></td></tr><tr><td data-num="3"></td><td><pre>import com.bytedance.covode.number.Covode;
</pre></td></tr><tr><td data-num="4"></td><td></td></tr><tr><td data-num="5"></td><td><pre>public class e0 extends l {
</pre></td></tr><tr><td data-num="6"></td><td><pre>    static {
</pre></td></tr><tr><td data-num="7"></td><td><pre>        Covode.recordClassIndex(1018605);
</pre></td></tr><tr><td data-num="8"></td><td><pre>    }
</pre></td></tr><tr><td data-num="9"></td><td></td></tr><tr><td data-num="10"></td><td><pre>    private static final void Bill() {
</pre></td></tr><tr><td data-num="11"></td><td><pre>    }
</pre></td></tr><tr><td data-num="12"></td><td></td></tr><tr><td data-num="13"></td><td><pre>    public strictfp void Francies() {
</pre></td></tr><tr><td data-num="14"></td><td><pre>    }
</pre></td></tr><tr><td data-num="15"></td><td></td></tr><tr><td data-num="16"></td><td><pre>    public static final void Louis() {
</pre></td></tr><tr><td data-num="17"></td><td><pre>    }
</pre></td></tr><tr><td data-num="18"></td><td></td></tr><tr><td data-num="19"></td><td><pre>    public static final void Zeoy() {
</pre></td></tr><tr><td data-num="20"></td><td><pre>    }
</pre></td></tr><tr><td data-num="21"></td><td><pre>}
</pre></td></tr></tbody></table>

把缺少的类补上去之后，随便调用一个字符串加密的函数

<table><tbody><tr><td data-num="1"></td><td><pre>Log.d(TAG,(String)l.a(0x1000001, 0, 0L, "d8044a", new byte[]{0x30, 41, 3, 72, 10, 0x72, 39, 27, 100, 97, 0x7B, 0x7A, 81, 69, 12, 0x7F, 0x74, 13, 100, 0x76, 59}));
</pre></td></tr></tbody></table>

没想到竟然输出了乱码

![](https://oacia.dev/douyin-6shen-init/image-20240814211204858.png)

为什么调用本来的字符串加密算法不能正常输出解密之后的字符串呢？

我思考了一下有以下几种可能的情况存在

1.  包名 / 签名校验：然而我 hook 了一圈都没找到相关的函数，包名改成和抖音一样的还是不行
2.  异常环境检测：但是我放到正常的手机上，log 输出的还是同样的结果
3.  检查 / 读取特定的文件：难道是检查 app 私有目录下有没有特定的文件，如果有特定的文件，就给解密的算法赋值正确的密钥？
4.  利用 binder 或者 socket 和其他的进程通讯来获取初始密钥？

下面是我对第三种情况读取文件的排查过程，因为考虑到最安全的调用系统函数的方式是 svc 调用

[#](#svc调用) SVC 调用
------------------

svc 的各个 code 对应的含义可以在[此处](https://cs.android.com/android/platform/superproject/+/master:bionic/libc/kernel/uapi/asm-generic/unistd.h "此处")找到

ida 搜索 svc 特征汇编 `01 00 00 D4` 找到如下位置，在 `sub_13E274` 中

![](https://oacia.dev/douyin-6shen-init/image-20240815220549586.png)

对这个函数 hook 一下

<table><tbody><tr><td data-num="1"></td><td><pre>function hook_svc() {
</pre></td></tr><tr><td data-num="2"></td><td><pre>    var module = Process.findModuleByName("libmetasec_ml.so")
</pre></td></tr><tr><td data-num="3"></td><td><pre>    Interceptor.attach(module.base.add(0x13E274),
</pre></td></tr><tr><td data-num="4"></td><td><pre>        {
</pre></td></tr><tr><td data-num="5"></td><td><pre>            onEnter: function (args) {
</pre></td></tr><tr><td data-num="6"></td><td><pre>                console.log('hook svc code-&gt;',this.context.x8.toInt32());
</pre></td></tr><tr><td data-num="7"></td><td><pre>            },
</pre></td></tr><tr><td data-num="8"></td><td><pre>            onLeave: function (retval) {
</pre></td></tr><tr><td data-num="9"></td><td></td></tr><tr><td data-num="10"></td><td><pre>            }
</pre></td></tr><tr><td data-num="11"></td><td><pre>        }
</pre></td></tr><tr><td data-num="12"></td><td><pre>    );
</pre></td></tr><tr><td data-num="13"></td><td><pre>}
</pre></td></tr></tbody></table>

但是却崩溃了？

用 frida 打个内存读写的断点看看

<table><tbody><tr><td data-num="1"></td><td><pre>function memory_read_hook(){
</pre></td></tr><tr><td data-num="2"></td><td><pre>    var module = Process.findModuleByName("libmetasec_ml.so")
</pre></td></tr><tr><td data-num="3"></td><td><pre>    MemoryAccessMonitor.enable(
</pre></td></tr><tr><td data-num="4"></td><td><pre>                {
</pre></td></tr><tr><td data-num="5"></td><td><pre>                    base:module.base.add(0x13E274),
</pre></td></tr><tr><td data-num="6"></td><td><pre>                    size:48
</pre></td></tr><tr><td data-num="7"></td><td><pre>                },{
</pre></td></tr><tr><td data-num="8"></td><td><pre>                    onAccess: function (details) {
</pre></td></tr><tr><td data-num="9"></td><td><pre>                        console.log(details.operation)
</pre></td></tr><tr><td data-num="10"></td><td><pre>                        console.log(get_addr_in_so(details.from));
</pre></td></tr><tr><td data-num="11"></td><td><pre>                    }
</pre></td></tr><tr><td data-num="12"></td><td><pre>                }
</pre></td></tr><tr><td data-num="13"></td><td><pre>            )
</pre></td></tr><tr><td data-num="14"></td><td><pre>}
</pre></td></tr><tr><td data-num="15"></td><td></td></tr><tr><td data-num="16"></td><td><pre>function get_addr_in_so(addr){
</pre></td></tr><tr><td data-num="17"></td><td><pre>    var process_Obj_Module_Arr = Process.enumerateModules();
</pre></td></tr><tr><td data-num="18"></td><td><pre>    for(var i = 0; i &lt; process_Obj_Module_Arr.length; i++) {
</pre></td></tr><tr><td data-num="19"></td><td><pre>        if(addr&gt;process_Obj_Module_Arr[i].base &amp;&amp; addr&lt;process_Obj_Module_Arr[i].base.add(process_Obj_Module_Arr[i].size)){
</pre></td></tr><tr><td data-num="20"></td><td><pre>            return addr.toString(16)+" is in "+process_Obj_Module_Arr[i].name+" offset: 0x"+(addr-process_Obj_Module_Arr[i].base).toString(16);
</pre></td></tr><tr><td data-num="21"></td><td><pre>        }
</pre></td></tr><tr><td data-num="22"></td><td><pre>    }
</pre></td></tr><tr><td data-num="23"></td><td><pre>    return addr.toString(16);
</pre></td></tr><tr><td data-num="24"></td><td><pre>}
</pre></td></tr></tbody></table>

需要注意的是我们需要在对 svc 的 hook 完成之后，延迟三秒在打内存读写断点

![](https://oacia.dev/douyin-6shen-init/image-20240815230204317.png)

frida 打印出了如下日志随后进程退出

![](https://oacia.dev/douyin-6shen-init/image-20240815230400591.png)

跳转到 `0x13e030` 看看，看来是对 svc 函数的完整性做了校验来防 hook

![](https://oacia.dev/douyin-6shen-init/image-20240815230857961.png)

那就对这个函数进行 hook 把校验的值强制改成 `0x312768B` , 没想到又崩溃了，但是打印的地址还是和先前相同，因为 frida `MemoryAccessMonitor` 自身的限制，只在第一次访问内存页的时候产生回调，后续的访问就不显示了，这个特性在 frida 的[官方文档](https://frida.re/docs/javascript-api/#memoryaccessmonitor "官方文档")中有所提及

![](https://oacia.dev/douyin-6shen-init/image-20240816200219287.png)

接下来考虑打个硬件断点看看

stackplz 打硬件断点

hook 代码如下

遇到这个 `unable to create socket: Operation not permitted` , 这是由于测试的 demo 没有给网络权限所以不能建立 socket

![](https://oacia.dev/douyin-6shen-init/image-20240816201801046.png)

在 `AndroidManifest.xml` 里面加上这个权限就可以了

但是貌似 hit 不到， `0x23c00` 跳转过去是空的并没有代码

![](https://oacia.dev/douyin-6shen-init/image-20240816205135003.png)

使用 [rwprocmem33](https://github.com/abcz316/rwProcMem33 " rwprocmem33") 打硬件断点，发现硬件断点命中了上千万次，而检测点的偏移计算出来是 `0x13e0e8`

![](https://oacia.dev/douyin-6shen-init/image-20240816213750249.png)

这个检测点就是我们先前去除垃圾指令后看到的这个函数

![](https://oacia.dev/douyin-6shen-init/image-20240816214140057.png)

这不是和我们之前用 frida 的 `MemoryAccessMonitor` 找到的监测点一模一样吗，那为啥下面这个脚本没办法过检测呢？直接把 checksum 修改成和正确的值 `0x312768B` 一样不应该行得通的嘛

<table><tbody><tr><td data-num="1"></td><td><pre>function bypass_svc_func_check() {
</pre></td></tr><tr><td data-num="2"></td><td><pre>    var module = Process.findModuleByName("libmetasec_ml.so")
</pre></td></tr><tr><td data-num="3"></td><td><pre>    Interceptor.attach(module.base.add(0x13E10C),
</pre></td></tr><tr><td data-num="4"></td><td><pre>        {
</pre></td></tr><tr><td data-num="5"></td><td><pre>            onEnter: function (args) {
</pre></td></tr><tr><td data-num="6"></td><td></td></tr><tr><td data-num="7"></td><td><pre>                console.log('internal svc func check bypass, x8 ',this.context.x8,'-&gt; 0x312768B');
</pre></td></tr><tr><td data-num="8"></td><td><pre>                this.context.x8 = 0x312768B;
</pre></td></tr><tr><td data-num="9"></td><td><pre>            },
</pre></td></tr><tr><td data-num="10"></td><td><pre>            onLeave: function (retval) {
</pre></td></tr><tr><td data-num="11"></td><td></td></tr><tr><td data-num="12"></td><td><pre>            }
</pre></td></tr><tr><td data-num="13"></td><td><pre>        }
</pre></td></tr><tr><td data-num="14"></td><td><pre>    );
</pre></td></tr><tr><td data-num="15"></td><td><pre>}
</pre></td></tr></tbody></table>

但是我们想想，rwprocmem33 打印出来的硬件断点，命中次数达到了上千万次，那么这绝对是另外起了一个线程来专门对这个地方做检测的

所以这个检测线程既然如此暴力，那么我们索性也一不做二不休，直接把检测的汇编暴力 patch 了不就好了！

具体代码如下，使用 `Memory.protect` 将这个 so 设为 `rwx` , 然后直接修改内存，把 `B.NE` 跳转改为 `B` 跳转

![](https://oacia.dev/douyin-6shen-init/image-20240816215710033.png)

这样这个 so 不管起多少线程，其内部线程的执行逻辑都将以我们期望的方式运行

<table><tbody><tr><td data-num="1"></td><td><pre>function bypass_svc_func_check() {
</pre></td></tr><tr><td data-num="2"></td><td><pre>    var libso = Process.getModuleByName("libmetasec_ml.so");
</pre></td></tr><tr><td data-num="3"></td><td><pre>    console.log("[name]:", libso.name);
</pre></td></tr><tr><td data-num="4"></td><td><pre>    console.log("[base]:", libso.base);
</pre></td></tr><tr><td data-num="5"></td><td><pre>    console.log("[size]:", ptr(libso.size));
</pre></td></tr><tr><td data-num="6"></td><td><pre>    console.log("[path]:", libso.path);
</pre></td></tr><tr><td data-num="7"></td><td><pre>    Memory.protect(ptr(libso.base), libso.size, 'rwx');
</pre></td></tr><tr><td data-num="8"></td><td><pre>    Memory.writeByteArray(ptr(libso.base).add(0x13E110),[0X03, 0X00, 0X00, 0X14]);
</pre></td></tr><tr><td data-num="9"></td><td><pre>}
</pre></td></tr></tbody></table>

终于我们也是成功的 hook 到了 svc 的调用码

![](https://oacia.dev/douyin-6shen-init/image-20240816215930992.png)

各个 svc code 的含义如下

<table><tbody><tr><td data-num="1"></td><td><pre>hook svc code-&gt; 56(__NR_openat) open /proc/self/exe
</pre></td></tr><tr><td data-num="2"></td><td><pre>hook svc code-&gt; 62(__NR3264_lseek)
</pre></td></tr><tr><td data-num="3"></td><td><pre>hook svc code-&gt; 63(__NR_read)
</pre></td></tr><tr><td data-num="4"></td><td><pre>hook svc code-&gt; 57(__NR_close)
</pre></td></tr><tr><td data-num="5"></td><td><pre>hook svc code-&gt; 135(__NR_rt_sigprocmask)
</pre></td></tr><tr><td data-num="6"></td><td><pre>hook svc code-&gt; 172(__NR_getpid)
</pre></td></tr><tr><td data-num="7"></td><td><pre>hook svc code-&gt; 178(__NR_gettid)
</pre></td></tr><tr><td data-num="8"></td><td><pre>hook svc code-&gt; 135(__NR_rt_sigprocmask)
</pre></td></tr></tbody></table>

但是感觉没有 open 什么有意义的文件的样子…

[#](#svc函数hook检测的bug) svc 函数 hook 检测的 bug
-----------------------------------------

但是这个检测是有 bug 的，其实这个点我其实在最初就发现了，但是本着严谨的态度于是就顺着这段检测代码想要的方式去过检测

回到检测函数我们看到这个检测函数对 `sub_13E274` 的 12 行汇编进行完整性检测

![](https://oacia.dev/douyin-6shen-init/image-20240816221230480.png)

但是被检查的函数，可是有 13 行的，所以我们直接对 `RET` 这行所在的地址进行 hook, 别的什么都不用做就可以成功的绕过完整性检测～

![](https://oacia.dev/douyin-6shen-init/image-20240816222440542.png)

那么如何才能成功的输出解密后的字符串呢？我又回去看了看 hook `ms.bd.c.l.a` 函数之后打印出来的日志，于是想着去调用最初的几个字符串参数去看看情况

![](https://oacia.dev/douyin-6shen-init/image-20240818165242631.png)

没想到 `.ms` 和 `.md` 一起调用的时候， `com.bytedance.ttnet.TTNetInit` 可以正常输出

![](https://oacia.dev/douyin-6shen-init/image-20240818165608816.png)

而单独调用 `com.bytedance.ttnet.TTNetInit` 输出的却是乱码

![](https://oacia.dev/douyin-6shen-init/image-20240818165636201.png)

那先调用 `.ms` 和 `.md` , 再调用之前解密出乱码的那个密文？

![](https://oacia.dev/douyin-6shen-init/image-20240818165937173.png)

这次字符串被成功的解密出来了！

![](https://oacia.dev/douyin-6shen-init/image-20240818165957226.png)

所以这个加密算法也很明了了，前两次被解密的字符串初始化了密钥，他们和后续字符串解密算法所使用的密钥有所关联的

现在我们用 [jnitrace](https://github.com/chame1eon/jnitrace " jnitrace") 去 trace 一下解密 `.ms` 时的 JNI 调用

<table><tbody><tr><td data-num="1"></td><td><pre>jnitrace -l libmetasec_ml.so -b fuzzy com.ss.android.ugc.awemea
</pre></td></tr></tbody></table>

在 `0x128b24` 的位置读取了传入的字符串密钥和密文，它所在的函数名为 `sub_128AAC`

![](https://oacia.dev/douyin-6shen-init/image-20240818172219054.png)

![](https://oacia.dev/douyin-6shen-init/image-20240818172227363.png)

![](https://oacia.dev/douyin-6shen-init/image-20240820010529912.png)

在 `0x5664c` 的位置是出现了被解密的字符串，它所在的函数名为 `sub_5611C`

![](https://oacia.dev/douyin-6shen-init/image-20240818172235640.png)

进入 `sub_128B88` 后我们发现，实际上在 `sub_5611C` 最后调用的 `sub_128B88` 才是 `NewStringUTF` 的位置

![](https://oacia.dev/douyin-6shen-init/image-20240820010457580.png)

![](https://oacia.dev/douyin-6shen-init/image-20240820010508273.png)

接下来我们在读取输入的函数 `sub_128AAC` 打印一下堆栈，来看看究竟是什么函数调用了他们

结果如下，堆栈回溯的地址都指向了 `sub_5611C`

![](https://oacia.dev/douyin-6shen-init/image-20240820014135813.png)

在这个函数中，初始密钥为 `0x71,0x62,0x13,0x14,0x5f,0x77,0x63,0x41,0x31,0x30` , 随后做了读取 str 密钥，密文的操作，并将初始密钥与传入的密钥循环异或生成二次密钥

![](https://oacia.dev/douyin-6shen-init/image-20240820021932419.png)

随后二次密钥与密文循环异或实现解密，但是如果没有没有实现字符串解密的两次初始化 `.ms` 和 `.md` 的话，将会对解密后的数组再次异或密钥，并异或 0x55, 通过这种方式来隐去密钥的相关特征

![](https://oacia.dev/douyin-6shen-init/image-20240820022014949.png)

所以最终的字符串解密算法如下～

<table><tbody><tr><td data-num="1"></td><td></td></tr><tr><td data-num="2"></td><td><pre>def douyin_str_dec(key: str, enc: list):
</pre></td></tr><tr><td data-num="3"></td><td><pre>    init_key = [0x71, 0x62, 0x13, 0x14, 0x5f, 0x77, 0x63, 0x41, 0x31, 0x30]
</pre></td></tr><tr><td data-num="4"></td><td><pre>    key = [ord(key[i % len(key)]) ^ init_key[i] for i in range(len(init_key))]
</pre></td></tr><tr><td data-num="5"></td><td><pre>    ret = [enc[i] ^ key[i % len(key)] for i in range(len(enc))]
</pre></td></tr><tr><td data-num="6"></td><td><pre>    return bytes(ret).decode()
</pre></td></tr><tr><td data-num="7"></td><td></td></tr><tr><td data-num="8"></td><td></td></tr><tr><td data-num="9"></td><td><pre>print(douyin_str_dec("976bf9",
</pre></td></tr><tr><td data-num="10"></td><td><pre>                     [43, 58, 72, 88, 91, 55, 46, 19, 99, 51, 38, 54, 64, 88, 77, 58, 52, 19, 115, 124, 28,1, 107, 19, 77, 7, 52, 31, 115]))
</pre></td></tr></tbody></table>