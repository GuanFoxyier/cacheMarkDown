> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=MzkxNzY5NjQxMg==&mid=2247483876&idx=1&sn=3ab1e8123ec0f3b8845dcd6e56580922&chksm=c1bdfb17f6ca7201540c679cb6b1d234c8345212aa1b4723eeda62dcc95f9a396ae3e7900a78&cur_album_id=3457518829104529411&scene=189#wechat_redirect)

```
0




前言

```

    文章中所有内容仅供学习交流使用，不用于其他任何目的，抓包内容、敏感网址、数据接口均已做脱敏处理，严禁用于商业和非法用途，否则由此产生的一切后果与作者无关。若有侵权，请在 vx【amuncocoL】联系作者

```
1




抓包及定位

```

  

    版本:8.36

    抓包如下:

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8q0nhibOHjUDrNMASPYYcibe6p26UYQtwe5d5YxYdHFDaY3eWpaibicR6zdA2nCmyib5sLFkT2Q2zrwH4Q/640?wx_fmt=jpeg&from=appmsg)

抓包截图中 shield 为本次分析的目标算法。

使用 frida-trace 通过跟进 NSURL 方法来跟踪调用：‍‍‍

```
frida-trace -UF com.xxx.xxx -m *[NSURL URLWithString*]"

```

通过分析调用堆栈找到 shield 算法生成的最外层函数:

```
+[XXXHandle reqAuthorityWithHeaders:request:bodyData:]

```

使用 frace-trace 跟踪下函数的调用和结果（敏感信息已做脱敏处理）‍

```
frida-trace -UF com.xxx.xxx   -m "*[XXXHandle reqAuthorityWithHeaders:request:bodyData:]"

```

结果如下:(敏感信息太多, 使用 new ObjC.Object(args[2]) 能得到更多信息)

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8rnXUNzmRNNfXv8eWU67AmoBAOD37oYibBVlrgRsCoscH5nPlY114ZS9caUIYrq1TaItA3LsNlHV7w/640?wx_fmt=jpeg&from=appmsg)

  

根据入参来固定参数, 在 frida 中写一个主动调用, 防止每次结果都不一样。

 代码如下:(已做脱敏处理, 可以将上一步得到的参数放进去)‍‍‍‍‍‍

```
function callReqAuthority(){
var headers = ObjC.classes.NSMutableDictionary.dictionary();
headers.setObject_forKey_('xxx','xxx');
headers.setObject_forKey_('xxx','xxx');
var NSString = ObjC.classes.NSString;
var urlString = NSString.stringWithString_('xxxxxx');
var url = ObjC.classes.NSURL.URLWithString_(urlString);
var request = ObjC.classes.NSMutableURLRequest.requestWithURL_(url);
var bodyStr = NSString.stringWithString_('xxxxx');
var body = bodyStr.dataUsingEncoding_(4);
var XXXHandle = ObjC.classes.XXXHandle;
XXXHandle.reqAuthorityWithHeaders_request_bodyData_(headers,request,body);
}

```

```
2




Base64




```

算法涉及太多, 我比较喜欢从结果倒推的方式来跟踪, 首先通过主动调用生成一份 trace 日志。‍‍‍‍‍‍‍‍‍

片段如下:

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8rnXUNzmRNNfXv8eWU67Amo41E9rN4LhcLnAIas0hOO4Wb0YrXTib4yHhZMgFiarNa67sMibcoUYMrGA/640?wx_fmt=jpeg&from=appmsg)

0x786c790 部分的 mov x0, x22 r[x22=2838e4640] 在 ida 中对应如下:  
‍‍

![](https://mmbiz.qpic.cn/mmbiz_png/wm415EXtN8rnXUNzmRNNfXv8eWU67AmozjfZj2g7kkGGMsCnzzf4QWLogKNibdlcltPBggjib6nlOtRRqNicvnlNA/640?wx_fmt=png&from=appmsg)

属于 sub_10786C1E8 的返回值, 验证下没问题。

汇编中 x22=2838e4640 对应伪代码中 v59 的结果。这里我是想跟下 v59 也就是汇编中 x22=2838e4640 的内存写入情况, 但是 iOS 中没找到像 unidbg 的写入监控方法。

这里采用通过搜索 trace 结果中的内存 x22=2838e4640 也就是 2838e4640 来进行。‍‍‍‍‍‍‍‍‍‍‍‍‍

搜索如下:

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8rnXUNzmRNNfXv8eWU67AmoeOmCic8vny7Oo0ZuvZFP3fRTRsATqL1dqjibHHBzdQOwcXICACCdmCxw/640?wx_fmt=jpeg&from=appmsg)

首次出现 2838e4640 内存的位置是 0x786e044, 进入这块的 ida 中看一下:‍‍‍‍‍‍‍‍

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8rnXUNzmRNNfXv8eWU67AmovuR1pXrKwF3gE9f7oBhAMDKUq5eALrmkLibYy8JezScWKwWNGe40vWA/640?wx_fmt=jpeg&from=appmsg)

可以看到这里将 CString 转成 OC 代码。‍

CString 来自 v41,v41 来自函数 sub_107873110,frida 跟进下：‍‍‍

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8p9kATdLVhSmwYpW1bShAJzsqWnAtjpVJ3H1xteAjmdHqB4FgbQm5iawangYE0mS0MN8f1w5WibWI8A/640?wx_fmt=jpeg&from=appmsg)

看 sub_107873110 返回结果就是 shield 的 C/C++ 字符串结果，进入 sub_107873110 看看。‍‍‍‍

倒序看下

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8p9kATdLVhSmwYpW1bShAJz67NFHzH3afwcmWP0icoWETI7AeIANjUc99341nicMf5S64YTKagiaPFOA/640?wx_fmt=jpeg&from=appmsg)

再进入 sub_107873408 倒序看到

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8p9kATdLVhSmwYpW1bShAJzTeIZLicxWI7avXcyg6SufIM6b4y7SFMwrlkpq5Foib58UnxWhEvmK32g/640?wx_fmt=jpeg&from=appmsg)

继续看 sub_1078707ec‍‍‍‍‍‍

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8p9kATdLVhSmwYpW1bShAJzQ3bgPUvYW3cyvvJ1ZFz9mgibgrwVXr31MiaEjWxzCByTbSPhj3c6CM6Q/640?wx_fmt=jpeg&from=appmsg)

看到 base64table 字段, hook 下这个函数看看入参:

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8p9kATdLVhSmwYpW1bShAJzhkzZDNEHCEuLFtQ34rBGwqeZI3FFHjibFHxZQNNzJicFTRRT9DhXmibgQ/640?wx_fmt=jpeg&from=appmsg)

验证下是不是标准 base64:‍‍‍

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8p9kATdLVhSmwYpW1bShAJzcXeOxBiaAoP0qar2MgMfgNic5FmzXnXcf3chlU4eDrYibJqLMibYdbVaoQ/640?wx_fmt=jpeg&from=appmsg)

验证后是标准 base64 算法。‍‍‍‍‍‍‍‍‍

```
3




RC4

```

验证完 base64 继续找 base64 的入参从哪来的。

上面分析过调用链条:

```
sub_107873110->sub_107873408->sub_1078707ec。

```

‍‍‍‍

直接看 sub_107873110, 上面打印过 sub_107873110 入参中有 01 01 53 53 , 但是没有字段。

直接说下, 下图标红位置其实是个指针:‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8p9kATdLVhSmwYpW1bShAJzWoczKyTbeJkGzxYpCibCOzwI8LCoNlicmuVU1WibOwX3I5eGZ8mZZ0vKg/640?wx_fmt=jpeg&from=appmsg)

解析下指针内容:‍

```
  var sub_107873110 = idaBase.add(0x107873110);
  Interceptor.attach(sub_107873110,{
    onEnter:function(args){
      this.res = this.context.x8;
     console.log('onEnter sub_107873110',hexdump(args[0].readPointer().add(24).readPointer()),'\n',hexdump(this.context.x8));
    },
    onLeave:function(retval){
      console.log('onLeave sub_107873110',hexdump(this.res.readPointer()))
  }});

```

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8p9kATdLVhSmwYpW1bShAJzu1ewtSQcUpKEbt29npcncYZTQy2Fiaj66TU5tywcuGWPWGbuLvjtsKQ/640?wx_fmt=jpeg&from=appmsg)

可以看出 base64 入参是 01 01 53 53 拼接这个指针中的内容。继续回到调用 sub_107873110 的地方:

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8p9kATdLVhSmwYpW1bShAJzAUbtF3yQibTuibgkibmH6aQn5mV1OZvtBbNAHWksOGicTohEUVOUE3A5wQ/640?wx_fmt=jpeg&from=appmsg)

看到 sub_107872DD8 的第一个参数来自 sub_107872DD8 的第二个参数, 进入 sub_107872DD8 看下。‍‍‍‍

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8p9kATdLVhSmwYpW1bShAJzVnmnBEiaZ5AIwqZlMYVWOB77zrolgd1TEJDfWNib3YyOe6lqtcbiaN7uw/640?wx_fmt=jpeg&from=appmsg)

这里 * v3+24 对应上面指针解析那部分。‍‍‍‍‍‍‍‍‍‍‍‍

直接 Hook 下 sub_1078727E4，sub_107872378。

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8p9kATdLVhSmwYpW1bShAJzm9jwE3CJvQetZSNiafeCKeGtSj3Vm2zia9Q7IlNBYfgfENR97HPcGbvQ/640?wx_fmt=jpeg&from=appmsg)

sub_1078727E4 的返回值是 sub_107872378 的入参，去 sub_1078727E4 看下:

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8p9kATdLVhSmwYpW1bShAJz9hSPhibF7HvAopGP9mt1vdBia4JWAPObkAxyzU2zsuk1FGpUiasoVSuLA/640?wx_fmt=jpeg&from=appmsg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8p9kATdLVhSmwYpW1bShAJz34CcCMkXpp80g13BicyKmXhFu1bSFaYR4gAFzK7D6srclWtNiahHV5Tw/640?wx_fmt=jpeg&from=appmsg)

‍‍‍

这里直接看只看到一堆表, 一开始没看出是什么算法。扣下代码固定好入参, 对着 trace 指令和 ida 伪代码一步一步还原成 c 代码看看。

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8rfLlCdH6YGxlhqYJRLECGjdaRznMEETIc8b10bfSnNFicqerRmlJnqYSd1RNFlzibuytJCXOeQLia6Q/640?wx_fmt=jpeg&from=appmsg)

结果：

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8p9kATdLVhSmwYpW1bShAJzvVelQA6QibOMwkkAQw50Nvn9cAahqYBNyfSeGkxlrPKq63EAJXXX2Xg/640?wx_fmt=jpeg&from=appmsg)

代码运行结果和之前 hook 一致‍‍‍‍‍‍‍‍‍‍‍‍‍

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8p9kATdLVhSmwYpW1bShAJzVxib6TFx1RoBTN2ia1p7PS2nicmgPHIr4sEnjNCkPdCy4HF7Ac3Xib1iaFw/640?wx_fmt=jpeg&from=appmsg)

代码还原过程中发现和 RC4 一致, 那么秘钥为了敏感保护就不说了, 跟着走的话 hook 过的参数中能找到。验证下:‍‍‍‍‍‍‍‍‍‍‍‍‍

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8p9kATdLVhSmwYpW1bShAJz53cGS70FU7rVYBibRIyWrJsXzkuU5J8YdiaVHVH4ILnI36nf12uV1ibhQ/640?wx_fmt=jpeg&from=appmsg)

验证是标准 RC4 算法, 那么看下 rc4 这几个参数的来源。‍‍‍‍‍‍‍‍‍‍‍‍‍‍

![](https://mmbiz.qpic.cn/mmbiz_png/wm415EXtN8p9kATdLVhSmwYpW1bShAJz6Ca01w0FweDKWmz61ok8TzI1Ftw5bjcKWEYiaYoib2g8nh0QfxoicR2Fg/640?wx_fmt=png&from=appmsg)

可以看到前一部分是从 sub_107872DD8 的第一个参数中固定偏移取值, 中间是我们开始固定主动调用的函数中的 build+deviceId。

后 16 位目前未知。

先看一下 IDA:‍‍‍‍‍

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8rfLlCdH6YGxlhqYJRLECGjSA59vQITia2icMicCDU6iaV5DVRuqq24ZeP7aYJvGbuBARXg1v8ibpBWpiag/640?wx_fmt=jpeg&from=appmsg)

三次从 v5 也就是第一个参数的固定偏移处 memcpy 数据, 盲猜是参数的后三部分打印下看看。‍‍‍‍‍‍

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8rfLlCdH6YGxlhqYJRLECGjlFVFRo2UHwDFGeQhQYX3y7icfhfdVsuNFUfsXAiciakZQicIK3e1h2Mo1Q/640?wx_fmt=jpeg&from=appmsg)

三个指针取出内容, 验证没问题都是从第一个参数取出来的, 其中 memcpy1 其实是固定参数中的 build,memcpy2 其实是固定参数时候的 deviceid. 固定位置取参数那里也是固定参数中的拼接, 那么目前只有 memcpy3 的目前未知。那么开始寻找这 16 字节：‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍‍

```
281aba6b0  46 c2 9d 66 95 35 2f 99 e2 50 36 51 e5 31 2d 96  F..f.5/..P6Q.1-.

```

  
‍

```
4




hmac-魔改md5




看一下sub_107872DD8的最外层调用addr_10786DE5C

打印下入参:

是我们要找的值,那就继续找addr_10786DE5C的入参

trace的指令中看到x0=16f58007,往上搜一下16f58007出现的地方。


进入ida中看一看

再看看进入 sub_107871554

简单看下是一个魔改后的MD5,首先能看出来左移的常量不同,

ida中显示为右移需要32减去后和标准的比较。其他地方是否魔改

需要继续确认。

hook下sub_107871554该方法会多次调用,看下第一次打印:




看到hmac填充数据,确认为HmacMD5,MD5有魔改,过程不多叙述,
就是固定入参保持一致,然后根据trace结果或者运行结果进行调试。
大致如下：

还原后拿到结果：




```

```
5




魔改AES

```

 剩下就是还原 HmacMd5 时候的 key, 来到 Hmac 的入参看到 key:

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8qeWEZrYMzGHF3FvdnMzP18K3u2e87iaGbaxmMH4S6LibLuQVichME2hUu3UkY6ib3w4akkAC0boeCic3A/640?wx_fmt=jpeg&from=appmsg)

来到 trace 结果中:‍‍‍‍‍‍‍‍

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8qeWEZrYMzGHF3FvdnMzP188ricjT9r3JsQXceQTdXics4cNwYKCdLp9QRKwrTAibCGo8CfNwJfPLqtQ/640?wx_fmt=jpeg&from=appmsg)

key 的入参来自 x1 寄存器, x1 寄存器中的数据由 282a75e54 地址的数据赋值而来继续向上继续搜索这个地址。‍‍‍‍‍‍‍‍‍‍

最终确认到 key 来自服务器下发的一个字段, 在经过 AES 算法后得出。

frida 打印下相关函数的入参, 0x10786BC8C 中的 base64 数据, 对应的 Bytes 在 0x10787220C 计算后获取到 Key, 如下:

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8qeWEZrYMzGHF3FvdnMzP185Dtmy4A489rYZaxmgHiaARadtaH1LCHXcqkqbyicbgxJiaRNsfS563ibAQ/640?wx_fmt=jpeg&from=appmsg)

看下 ida：‍

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8qeWEZrYMzGHF3FvdnMzP18652n50eqPl2NJBlau1icTAhWUiayVMo4mUVIO7Qhy1JzduI1SUGUUgpA/640?wx_fmt=jpeg&from=appmsg)

图片放不下直接放 ida 伪代码。‍‍‍‍‍‍‍‍‍‍

sub_10786FF44：

```
__int64 __fastcall sub_10786FF44(__int64 a1, __int64 aes_key_Len, _DWORD *a3)
{
  _DWORD *v3; // x19
  int value_10; // w9
  __int64 v5; // x8
  int value_388; // w9
  __int64 v7; // x10
  int *v8; // x9
  int *v9; // x11
  int v10; // w12
  int v11; // w12
  int v12; // w12
  int v13; // w12
  bool v14; // cc
  int *v15; // x8
  int v16; // w9
  unsigned __int64 v17; // x15
  unsigned __int64 v18; // x16
  unsigned __int64 v19; // x15
  unsigned __int64 v20; // x15
  v3 = a3;
  sub_10786F91C((unsigned int *)a1, aes_key_Len, a3);
  value_10 = v3[60];                            // // 0xa  = 10
  if ( value_10 >= 1 )
  {
    v5 = 0LL;
    value_388 = 4 * value_10;
    v7 = value_388 - 4LL;
    v8 = &v3[value_388 + 2];
    v9 = v3 + 2;
    do
    {
      v10 = *(v9 - 2);
      *(v9 - 2) = *(v8 - 2);
      *(v8 - 2) = v10;
      v11 = *(v9 - 1);
      *(v9 - 1) = *(v8 - 1);
      *(v8 - 1) = v11;
      v12 = *v9;
      *v9 = *v8;
      *v8 = v12;
      v13 = v9[1];
      v9[1] = v8[1];
      v8[1] = v13;
      v5 += 4LL;
      v8 -= 4;
      v9 += 4;
      v14 = v5 < v7;
      v7 -= 4LL;
    }
    while ( v14 );
    if ( v3[60] >= 2 )
    {
      v15 = v3 + 7;
      v16 = 1;
      do
      {
        v17 = (unsigned int)*(v15 - 3);
        v18 = (unsigned int)*(v15 - 2);
        LODWORD(v18) = dword_10F659E74[byte_10F65964C[4 * ((v18 >> 16) & 0xFF)]] ^ dword_10F659A74[byte_10F65964C[(v18 >> 22) & 0x3FC]] ^ dword_10F65A274[byte_10F65964C[4 * ((unsigned __int16)v18 >> 8)]] ^ dword_10F65A674[byte_10F65964C[4 * (unsigned __int8)v18]];
        *(v15 - 3) = dword_10F659E74[byte_10F65964C[4 * ((v17 >> 16) & 0xFF)]] ^ dword_10F659A74[byte_10F65964C[(v17 >> 22) & 0x3FC]] ^ dword_10F65A274[byte_10F65964C[4 * ((unsigned __int16)v17 >> 8)]] ^ dword_10F65A674[byte_10F65964C[4 * (unsigned __int8)v17]];
        *(v15 - 2) = v18;
        v19 = (unsigned int)*(v15 - 1);
        *(v15 - 1) = dword_10F659E74[byte_10F65964C[4 * ((v19 >> 16) & 0xFF)]] ^ dword_10F659A74[byte_10F65964C[(v19 >> 22) & 0x3FC]] ^ dword_10F65A274[byte_10F65964C[4 * ((unsigned __int16)v19 >> 8)]] ^ dword_10F65A674[byte_10F65964C[4 * (unsigned __int8)v19]];
        v20 = (unsigned int)*v15;
        *v15 = dword_10F659E74[byte_10F65964C[4 * ((v20 >> 16) & 0xFF)]] ^ dword_10F659A74[byte_10F65964C[(v20 >> 22) & 0x3FC]] ^ dword_10F65A274[byte_10F65964C[4 * ((unsigned __int16)v20 >> 8)]] ^ dword_10F65A674[byte_10F65964C[4 * (unsigned __int8)v20]];
        v15 += 4;
        ++v16;
      }
      while ( v16 < v3[60] );
    }
  }
  return 1LL;
}

```

```
__int64 __fastcall sub_10786F91C(unsigned int *a1, int a2, unsigned int *a3)
{
  __int64 v3; // x8
  int v4; // w8
  unsigned int v6; // w9
  __int64 v7; // x8
  unsigned int *v8; // x10
  int v9; // w0
  unsigned __int64 v10; // x16
  int v11; // w17
  int v12; // w17
  unsigned __int64 v13; // x12
  int v14; // w13
  unsigned int v15; // w15
  int v16; // w14
  int v17; // w15
  unsigned int v18; // w17
  int v19; // w16
  int v20; // w17
  int v21; // w13
  int v22; // w0
  int v23; // w13
  int v24; // w14
  int v25; // w15
  int v26; // w16
  int v27; // w17
  int v28; // w13
  int v29; // w0
  int v30; // w13
  int v31; // w14
  int v32; // w15
  int v33; // w16
  int v34; // w17
  int v35; // w13
  int v36; // w0
  int v37; // w13
  int v38; // w14
  int v39; // w15
  int v40; // w16
  int v41; // w17
  int v42; // w13
  int v43; // w0
  int v44; // w13
  int v45; // w14
  int v46; // w15
  int v47; // w16
  int v48; // w17
  int v49; // w13
  int v50; // w0
  int v51; // w13
  int v52; // w14
  int v53; // w15
  int v54; // w16
  int v55; // w17
  int v56; // w13
  int v57; // w0
  int v58; // w13
  int v59; // w14
  int v60; // w15
  int v61; // w16
  int v62; // w17
  unsigned int v63; // w8
  int v64; // w8
  __int64 v65; // x8
  unsigned int *i; // x10
  unsigned __int64 v67; // x16
  int v68; // w0
  int v69; // w17
  int v70; // w0
  unsigned int v71; // w17
  int v72; // w17
  int v73; // w0
  int v74; // w17
  v3 = 0LL;
  if ( a1 && a3 )
  {
    if ( a2 != 128 && a2 != 256 && a2 != 192 )
      return 0LL;
    if ( a2 == 128 )
    {
      v4 = 10;
    }
    else if ( a2 == 192 )
    {
      v4 = 12;
    }
    else
    {
      v4 = 14;
    }
    a3[60] = v4;
    v6 = bswap32(*a1) ^ 0xF1892131;
    *a3 = v6;
    a3[1] = bswap32(a1[1]) ^ 0xFF001123;
    a3[2] = bswap32(a1[2]) ^ 0xF1001356;
    a3[3] = bswap32(a1[3]) ^ 0xF1234890;
    if ( a2 == 128 )
    {
      v7 = 0LL;
      v8 = a3 + 4;
      do
      {
        v9 = *(v8 - 2);
        v10 = *(v8 - 1);
        v6 ^= (byte_10F658A4C[4 * BYTE2(v10) + 3] << 24) ^ (byte_10F658E4C[4 * BYTE1(v10) + 2] << 16) ^ (RijnDael_AES_10F65924C[4 * (unsigned __int8)v10 + 1] << 8) ^ byte_10F65964C[(v10 >> 22) & 0x3FC] ^ *(_DWORD *)((char *)&unk_10F659A4C + v7);
        v11 = *(v8 - 3) ^ v6;
        *v8 = v6;
        v8[1] = v11;
        v12 = v9 ^ v11;
        v8[2] = v12;
        v8[3] = v12 ^ v10;
        v7 += 4LL;
        v8 += 4;
      }
      while ( v7 != 40 );
    }
    else
    {
      a3[4] = bswap32(a1[4]);
      a3[5] = bswap32(a1[5]);
      if ( a2 == 192 )
      {
        v13 = a3[5];
        v14 = v6 ^ (byte_10F658A4C[4 * BYTE2(v13) + 3] << 24) ^ (byte_10F658E4C[4 * BYTE1(v13) + 2] << 16) ^ (RijnDael_AES_10F65924C[4 * (unsigned __int8)v13 + 1] << 8) ^ byte_10F65964C[(v13 >> 22) & 0x3FC] ^ 0x12310000;
        v15 = a3[2];
        v16 = a3[1] ^ v14;
        a3[6] = v14;
        a3[7] = v16;
        v17 = v15 ^ v16;
        v18 = a3[4];
        v19 = a3[3] ^ v17;
        a3[8] = v17;
        a3[9] = v19;
        v20 = v18 ^ v19;
        LODWORD(v13) = v20 ^ v13;
        v21 = v14 ^ (byte_10F658A4C[4 * (((unsigned int)v13 >> 16) & 0xFF) + 3] << 24) ^ (byte_10F658E4C[4 * ((unsigned __int16)v13 >> 8) + 2] << 16) ^ (RijnDael_AES_10F65924C[4 * (unsigned __int8)v13 + 1] << 8);
        v22 = byte_10F65964C[4 * ((unsigned int)v13 >> 24)];
        a3[10] = v20;
        a3[11] = v13;
        v23 = v21 ^ v22 ^ 0x2000100;
        v24 = v16 ^ v23;
        a3[12] = v23;
        a3[13] = v24;
        v25 = v17 ^ v24;
        v26 = v19 ^ v25;
        a3[14] = v25;
        a3[15] = v26;
        v27 = v20 ^ v26;
        LODWORD(v13) = v27 ^ v13;
        v28 = v23 ^ (byte_10F658A4C[4 * (((unsigned int)v13 >> 16) & 0xFF) + 3] << 24) ^ (byte_10F658E4C[4 * ((unsigned __int16)v13 >> 8) + 2] << 16) ^ (RijnDael_AES_10F65924C[4 * (unsigned __int8)v13 + 1] << 8);
        v29 = byte_10F65964C[4 * ((unsigned int)v13 >> 24)];
        a3[16] = v27;
        a3[17] = v13;
        v30 = v28 ^ v29 ^ 0x4020000;
        v31 = v24 ^ v30;
        a3[18] = v30;
        a3[19] = v31;
        v32 = v25 ^ v31;
        v33 = v26 ^ v32;
        a3[20] = v32;
        a3[21] = v33;
        v34 = v27 ^ v33;
        LODWORD(v13) = v34 ^ v13;
        v35 = v30 ^ (byte_10F658A4C[4 * (((unsigned int)v13 >> 16) & 0xFF) + 3] << 24) ^ (byte_10F658E4C[4 * ((unsigned __int16)v13 >> 8) + 2] << 16) ^ (RijnDael_AES_10F65924C[4 * (unsigned __int8)v13 + 1] << 8);
        v36 = byte_10F65964C[4 * ((unsigned int)v13 >> 24)];
        a3[22] = v34;
        a3[23] = v13;
        v37 = v35 ^ v36 ^ 0x8020200;
        v38 = v31 ^ v37;
        a3[24] = v37;
        a3[25] = v38;
        v39 = v32 ^ v38;
        v40 = v33 ^ v39;
        a3[26] = v39;
        a3[27] = v40;
        v41 = v34 ^ v40;
        LODWORD(v13) = v41 ^ v13;
        v42 = v37 ^ (byte_10F658A4C[4 * (((unsigned int)v13 >> 16) & 0xFF) + 3] << 24) ^ (byte_10F658E4C[4 * ((unsigned __int16)v13 >> 8) + 2] << 16) ^ (RijnDael_AES_10F65924C[4 * (unsigned __int8)v13 + 1] << 8);
        v43 = byte_10F65964C[4 * ((unsigned int)v13 >> 24)];
        a3[28] = v41;
        a3[29] = v13;
        v44 = v42 ^ v43 ^ 0x10102000;
        v45 = v38 ^ v44;
        a3[30] = v44;
        a3[31] = v45;
        v46 = v39 ^ v45;
        v47 = v40 ^ v46;
        a3[32] = v46;
        a3[33] = v47;
        v48 = v41 ^ v47;
        LODWORD(v13) = v48 ^ v13;
        v49 = v44 ^ (byte_10F658A4C[4 * (((unsigned int)v13 >> 16) & 0xFF) + 3] << 24) ^ (byte_10F658E4C[4 * ((unsigned __int16)v13 >> 8) + 2] << 16) ^ (RijnDael_AES_10F65924C[4 * (unsigned __int8)v13 + 1] << 8);
        v50 = byte_10F65964C[4 * ((unsigned int)v13 >> 24)];
        a3[34] = v48;
        a3[35] = v13;
        v51 = v49 ^ v50 ^ 0x30020400;
        v52 = v45 ^ v51;
        a3[36] = v51;
        a3[37] = v52;
        v53 = v46 ^ v52;
        v54 = v47 ^ v53;
        a3[38] = v53;
        a3[39] = v54;
        v55 = v48 ^ v54;
        LODWORD(v13) = v55 ^ v13;
        v56 = v51 ^ (byte_10F658A4C[4 * (((unsigned int)v13 >> 16) & 0xFF) + 3] << 24) ^ (byte_10F658E4C[4 * ((unsigned __int16)v13 >> 8) + 2] << 16) ^ (RijnDael_AES_10F65924C[4 * (unsigned __int8)v13 + 1] << 8);
        v57 = byte_10F65964C[4 * ((unsigned int)v13 >> 24)];
        a3[40] = v55;
        a3[41] = v13;
        v58 = v56 ^ v57 ^ 0x40002000;
        v59 = v52 ^ v58;
        a3[42] = v58;
        a3[43] = v59;
        v60 = v53 ^ v59;
        v61 = v54 ^ v60;
        a3[44] = v60;
        a3[45] = v61;
        v62 = v55 ^ v61;
        LODWORD(v13) = v62 ^ v13;
        a3[46] = v62;
        a3[47] = v13;
        v63 = v58 ^ (byte_10F658A4C[4 * (((unsigned int)v13 >> 16) & 0xFF) + 3] << 24) ^ (byte_10F658E4C[4 * ((unsigned __int16)v13 >> 8) + 2] << 16) ^ (RijnDael_AES_10F65924C[4 * (unsigned __int8)v13 + 1] << 8) ^ byte_10F65964C[4 * ((unsigned int)v13 >> 24)] ^ 0x80002000;
        a3[48] = v63;
        a3[49] = v59 ^ v63;
        v64 = v60 ^ v59 ^ v63;
        a3[50] = v64;
        a3[51] = v61 ^ v64;
      }
      else
      {
        a3[6] = bswap32(a1[6]);
        a3[7] = bswap32(a1[7]);
        v65 = 0LL;
        for ( i = a3 + 8; ; i += 8 )
        {
          v67 = *(i - 1);
          v6 ^= (byte_10F658A4C[4 * BYTE2(v67) + 3] << 24) ^ (byte_10F658E4C[4 * BYTE1(v67) + 2] << 16) ^ (RijnDael_AES_10F65924C[4 * (unsigned __int8)v67 + 1] << 8) ^ byte_10F65964C[(v67 >> 22) & 0x3FC] ^ *(_DWORD *)((char *)&unk_10F659A4C + v65);
          v68 = *(i - 6);
          v69 = *(i - 7) ^ v6;
          *i = v6;
          i[1] = v69;
          v70 = v68 ^ v69;
          v71 = *(i - 5) ^ v70;
          i[2] = v70;
          i[3] = v71;
          if ( v65 == 24 )
            break;
          v72 = *(i - 4) ^ (byte_10F658A4C[4 * (v71 >> 24) + 3] << 24) ^ (byte_10F658E4C[4 * ((v71 >> 16) & 0xFF) + 2] << 16) ^ (RijnDael_AES_10F65924C[4 * ((unsigned __int16)v71 >> 8) + 1] << 8) ^ byte_10F65964C[4 * (unsigned __int8)v71];
          v73 = *(i - 3) ^ v72;
          i[4] = v72;
          i[5] = v73;
          v74 = *(i - 2) ^ v73;
          i[6] = v74;
          i[7] = v74 ^ v67;
          v65 += 4LL;
        }
      }
    }
    v3 = 1LL;
  }
  return v3;
  }

```

```
__n128 __fastcall sub_101CAD760(unsigned __int64 *a1, __n128 *a2, unsigned __int64 a3, __int64 a4, __n128 *a5, void (__fastcall *a6)(__n128 *, __n128 *, __int64))
{
  void (__fastcall *v6)(__n128 *, __n128 *, __int64); // x21
  __n128 *v7; // x19
  __int64 v8; // x22
  unsigned __int64 v9; // x24
  __n128 *v10; // x20
  unsigned __int64 *v11; // x23
  unsigned __int64 v12; // x26
  __n128 *v13; // x8
  __n128 *v14; // x27
  unsigned __int64 v15; // x25
  __n128 *v16; // x0
  unsigned __int64 v17; // x8
  unsigned __int64 v18; // x9
  unsigned __int64 v19; // x10
  _BYTE *v20; // x11
  char *v21; // x12
  char *v22; // x9
  char v23; // w13
  char v24; // t1
  char v25; // t1
  unsigned __int64 v26; // x10
  unsigned __int64 v27; // x10
  int8x8_t *v28; // x11
  unsigned __int64 *v29; // x12
  unsigned __int64 *v30; // x13
  unsigned __int64 v31; // t1
  int8x8_t v32; // d0
  int8x8_t v33; // t1
  __int64 v34; // x9
  unsigned __int64 v35; // x11
  __int64 *v36; // x13
  _QWORD *v37; // x14
  unsigned __int64 v38; // x11
  __int64 v39; // t1
  __n128 *v40; // x10
  __n128 *v41; // x12
  unsigned __int64 v42; // x13
  __n128 v43; // t1
  __n128 result; // q0
  if ( a3 )
  {
    v6 = a6;
    v7 = a5;
    v8 = a4;
    v9 = a3;
    v10 = a2;
    v11 = a1;
    if ( a3 < 0x10 )
    {
      v17 = (unsigned __int64)a5;
      v15 = a3;
      if ( a3 >= 8 )
        goto LABEL_14;
    }
    else
    {
      v12 = 0LL;
      v13 = a5;
      v14 = a2;
      v15 = a3;
      do
      {
        v16 = &v10[v12 / 0x10];
        v16->n128_u64[0] = v13->n128_u64[0] ^ v11[v12 / 8];
        v16->n128_u64[1] = v13->n128_u64[1] ^ v11[v12 / 8 + 1];
        v6(&v10[v12 / 0x10], &v10[v12 / 0x10], v8);
        v15 -= 16LL;
        v13 = v14;
        ++v14;
        v12 += 16LL;
      }
      while ( v15 > 0xF );
      v10 = (__n128 *)((char *)v10 + v12);
      v17 = (unsigned __int64)v10[-1].n128_u64;
      if ( v9 == v12 )
      {
        --v10;
LABEL_36:
        result = *v10;
        *v7 = *v10;
        return result;
      }
      v11 = (unsigned __int64 *)((char *)v11 + v12);
      if ( v15 >= 8 )
      {
LABEL_14:
        v18 = 0LL;
        if ( (v10 >= (__n128 *)((char *)v11 + v15) || v11 >= (unsigned __int64 *)((char *)v10->n128_u64 + v15))
          && ((unsigned __int64)v10 >= v17 + v15 || v17 >= (unsigned __int64)v10->n128_u64 + v15) )
        {
          v18 = v15 & 0xFFFFFFFFFFFFFFF8LL;
          v27 = v15 & 0xFFFFFFFFFFFFFFF8LL;
          v28 = (int8x8_t *)v10;
          v29 = (unsigned __int64 *)v17;
          v30 = v11;
          do
          {
            v31 = *v30;
            ++v30;
            v32.n64_u64[0] = v31;
            v33.n64_u64[0] = *v29;
            ++v29;
            v28->n64_u64[0] = veor_s8(v33, v32).n64_u64[0];
            ++v28;
            v27 -= 8LL;
          }
          while ( v27 );
          if ( v15 == v18 )
          {
LABEL_11:
            if ( v15 > 8 )
            {
              v26 = v15;
              goto LABEL_34;
            }
            if ( (unsigned __int64)v10->n128_u64 + v15 < v17 + 16 && v17 + v15 < (unsigned __int64)v10[1].n128_u64 )
            {
              v26 = v15;
              goto LABEL_34;
            }
            v34 = 16 - v15;
            if ( v15 )
            {
              v35 = 0LL;
            }
            else
            {
              v35 = v34 & 0xFFFFFFFFFFFFFFF0LL;
              v40 = (__n128 *)v17;
              v41 = v10;
              v42 = v34 & 0xFFFFFFFFFFFFFFF0LL;
              do
              {
                v43 = *v40;
                ++v40;
                *v41 = v43;
                ++v41;
                v42 -= 16LL;
              }
              while ( v42 );
              if ( v34 == v35 )
                goto LABEL_35;
              if ( !(v34 & 8) )
              {
                v26 = v34 & 0xFFFFFFFFFFFFFFF0LL;
                do
                {
LABEL_34:
                  *((_BYTE *)v10->n128_u64 + v26) = *(_BYTE *)(v17 + v26);
                  ++v26;
                }
                while ( v26 != 16 );
LABEL_35:
                v6(v10, v10, v8);
                goto LABEL_36;
              }
            }
            v26 = v15 + (v34 & 0xFFFFFFFFFFFFFFF8LL);
            v36 = (__int64 *)(v17 + v35 + v15);
            v37 = (unsigned __int64 *)((char *)v10->n128_u64 + v35 + v15);
            v38 = v35 - (v34 & 0xFFFFFFFFFFFFFFF8LL);
            do
            {
              v39 = *v36;
              ++v36;
              *v37 = v39;
              ++v37;
              v38 += 8LL;
            }
            while ( v38 );
            if ( v34 == (v34 & 0xFFFFFFFFFFFFFFF8LL) )
              goto LABEL_35;
            goto LABEL_34;
          }
        }
LABEL_9:
        v19 = v15 - v18;
        v20 = (char *)v10 + v18;
        v21 = (char *)(v17 + v18);
        v22 = (char *)v11 + v18;
        do
        {
          v24 = *v22++;
          v23 = v24;
          v25 = *v21++;
          *v20++ = v25 ^ v23;
          --v19;
        }
        while ( v19 );
        goto LABEL_11;
      }
    }
    v18 = 0LL;
    goto LABEL_9;
  }
return result;
}

```

可以看到魔改 AES 是查表法实现的, 我是看的头疼 直接扣代码吧。

结合 trace 的汇编结果复现。过程中就是坐屁股跟 trace 日志 看汇编，大致如下:

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8qeWEZrYMzGHF3FvdnMzP18Wo9NHHRy1nQQjcIBbGS4GiaMHMCAtbJH2cl8jAMbYAMgfNVLE3XW07w/640?wx_fmt=jpeg&from=appmsg)

还原结果：‍‍‍‍

![](https://mmbiz.qpic.cn/mmbiz_jpg/wm415EXtN8qeWEZrYMzGHF3FvdnMzP18HQXbOGXhBWFEzRibb9ArCEQIdLItrHbZESzHUX0rvyYOsWjQyKCg0nQ/640?wx_fmt=jpeg&from=appmsg)

```
6




总结

```

这个样本虽然涉及到了几处魔改算法, 我本人没有太多的奇淫巧技, 但是多坐屁股应该都可以搞定的。

撒花~ 今天的分享就到这了, 咱们下次再会~  

`‍`