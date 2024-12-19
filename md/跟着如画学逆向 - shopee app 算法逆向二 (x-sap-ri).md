> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [articles.zsxq.com](https://articles.zsxq.com/id_qgwwz65dnnle.html)

> 在上篇文章中说到, 我们主动调用了几次, 返回结果都是不同的

在上篇文章中说到, 我们主动调用了几次, 返回结果都是不同的

> 相同参数, 我们主动多次 call. 可以看到结果是不同的. 只有一个 Key 不同.

接下来, 引用龙哥的文章

> 引用自龙哥文章, 我仅仅是对关键信息做加粗

1.1 引言
------

在使用 Unidbg 模拟执行以及辅助算法还原时，有这样一种需求——**入参不变，返回值也不变** 。本文讨论这个需求的使用场景以及如何实现它。

1.2 问题分析
--------

   
在代码开发过程中，因为各种各样的原因，常常会引入**随机数、时间戳** 等信息参与运算。这些信息随着时间、系统状态等因素不停变化，它们导致函数入参不变的情况下，计算出的结果不固定。在模拟执行以及算法还原时，我们有控制这些干扰项，使得结果固定的需求。

对于模拟执行而言，**如果能固定结果，就能确定 Unidbg 补环境是否符合预期** ，是否和真实环境完全一样。下面详细展开讨论。

在固定干扰项后，使用 **Unidbg 与 Frida 调用目标函数且入参一致时，理论上返回结果也一致，这意味着我们得到了绝对意义上的，完全正确的模拟执行。**

具体操作流程如下

1.  1.
    
    在 Unidbg 中找到和固定干扰项，使得 Unidbg Call 在入参不变的情况下结果固定。
    
2.  2.
    
    参考 Unidbg，用 Frida 同等固定干扰项（使用 inline hook / patch / replace 等等），使得 Frida Call 在入参不变的情况下结果固定。
    

**在顺序上，选择先在 Unidbg 里分析，然后迁移到真机上，这是因为在 Unidbg 找干扰项更容易，所有的系统调用、文件访问、JNI 都由 Unidbg 处理和监控，固定它们并进行测试十分方便。**

对于算法还原而言，如果**能固定干扰项，使得参数不变时结果固定，这会对辅助算法还原帮助巨大** 。这意味着，只要不改变入参，**程序的控制流和数据流就完全固化** 。逆向分析好比闯关卡，分析人员有无数条命，而关卡总是固定，你可以**砸时间不停试错，熟悉并最终击败每一个陷阱和挑战** 。这并非好无副作用，存在代码覆盖率和执行路径的问题，但和它带来的好处相比，是一笔划算买卖。

1.3 干扰项来源
---------

   
这种随机干扰可能来源于 JNI、库函数、系统调用、文件访问等方面。

### 1.3.1 JNI

程序可以通过 JNI 调用 JAVA 中的各种变值，最常见的是时间和随机数。

**时间**

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
@Override
public long callStaticLongMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
    switch (signature){
        case "java/lang/System->currentTimeMillis()J":{
            return System.currentTimeMillis();
        }
    }
    return super.callStaticLongMethodV(vm, dvmClass, signature, vaList);
}

```

**随机数**

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
@Override
public long callStaticLongMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
    switch (signature){
        case "java/lang/System->currentTimeMillis()J":{
            return 1667275121452L;
        }
    }
    return super.callStaticLongMethodV(vm, dvmClass, signature, vaList);
}

```

### 1.3.2 库函数

比如 gettimeoday、clock_gettime 等库函数都可以获取时间信息。

先以 gettimeofday 为例，其函数原型如下。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
int gettimeofday(struct timeval *tv, struct timezone *tz);

```

参数 1 tv 是指向 timeval 结构体的指针，这个结构体定义如下。tz 参数使用不多，此处不表。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
struct timeval {
  long tv_sec; 秒时间戳
  long tv_usec; 余下的微秒
};

```

函数返回 0 代表调用成功，返回 -1 表示调用失败。

当我们只想获取秒级时间戳时，一般像下面这么写代码。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
long gettime(){
    struct timeval t{};
    gettimeofday(&t, nullptr);
    long sec = t.tv_sec;
    return sec;
}

```

当我们想获取毫秒时间戳的时候，一般像下面这么写代码。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
long long gettime2(){
    struct timeval t{};
    gettimeofday(&t, nullptr);
    long sec = t.tv_sec;
    long usec = t.tv_usec;
    return (sec * 1000) + (usec / 1000);
}

```

它是获取时间戳的最常用函数，如果想使用 Frida 在真实环境中固定它，往往会直接替换它。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
function hook_gettimeofday() {
  var addr_gettimeofday = Module.findExportByName(null, "gettimeofday");
​
  Interceptor.replace(addr_gettimeofday, new NativeCallback(function (ptr_tz, ptr_tzp) {
    console.log("hook gettimeofday:", ptr_tz, ptr_tzp, result);
    var t = new Int32Array(ArrayBuffer.wrap(ptr_tz, 8));
    t[0] = 0xAAAA;
    t[1] = 0xBBBB;
  }, "int", ["pointer", "pointer"]));
}

```

除了 gettimeofday，clock_gettime 是另一个高频函数。这是一个语义相当丰富的函数，可以获取真实时间、进程时间、线程时间等多种类型的时间，函数原型如下。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
int clock_gettime(clockid_t __clock, struct timespec* __ts);

```

timespec 比 timeval 更加精确，1 纳秒 = 1000 微秒。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
struct timespec {
    long tv_sec; // 秒时间戳
    long tv_nsec; // 余下的纳秒时间戳
};

```

如果想和 gettimeofday 一样获取毫秒级时间戳，代码如下。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
long long gettime3(){
    struct timespec t = {0, 0};
    clock_gettime(CLOCK_REALTIME, &t);
    return (t.tv_sec*1000) + (t.tv_nsec/1000000);
}

```

参数 1 即 clockid ，它是所谓的时钟。当设置为 CLOCK_REALTIME 时，意味着获取真实时间信息，此时效果和 gettimeofday 一致，区别仅在于时间精度更高。时钟还有相当多的类型

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
#define CLOCK_REALTIME 0
#define CLOCK_MONOTONIC 1
#define CLOCK_PROCESS_CPUTIME_ID 2
#define CLOCK_THREAD_CPUTIME_ID 3
#define CLOCK_MONOTONIC_RAW 4
#define CLOCK_REALTIME_COARSE 5
#define CLOCK_MONOTONIC_COARSE 6
#define CLOCK_BOOTTIME 7
#define CLOCK_REALTIME_ALARM 8
#define CLOCK_BOOTTIME_ALARM 9
#define CLOCK_SGI_CYCLE 10
#define CLOCK_TAI 11

```

CLOCK_MONOTONIC 相关的时钟表示从**某个不确定的点** 开始计时的时间，这个不确定的点往往是开机或开机后的某个时间。

CLOCK_BOOTTIME 相关的时钟表示了开机时间。

CLOCK_PROCESS_CPUTIME_ID 和 CLOCK_THREAD_CPUTIME_ID 代表了 CPU 在当前进程 / 线程上的耗时，相比较这个进程 / 线程的维持时间，CPU 耗时会是一个小很多的数字。

简而言之可以分为真实时间、CPU 时间、开机时间、从不确定的某个时间点开始计时这四部分。

除此之外，还有 time 、ftime 库函数，它们基于 gettimeofday，比如 time 的源码如下。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
int __fastcall time(_DWORD *a1, int a2)
{
    int v3; // r3
    int v5[4]; // [sp+0h] [bp-10h] BYREF
​
    v5[0] = (int)a1;
    v5[1] = a2;
    if ( j_gettimeofday(v5, 0) < 0 )
      return -1;
    v3 = v5[0];
    if ( a1 )
      *a1 = v5[0];
    return v3;
}

```

还有 clock 库函数，它基于 clock_gettime，源码如下。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
int __fastcall clock(int a1, int a2, int a3)
{
    int result; // r0
    int v4; // [sp+0h] [bp-10h] BYREF
    int v5; // [sp+4h] [bp-Ch]
    int v6; // [sp+8h] [bp-8h]
​
    v4 = a1;
    v5 = a2;
    v6 = a3;
    result = j_clock_gettime(2, &v4);
    if ( result != -1 )
      result = v5 / 1000 + 1000000 * v4;
    return result;
}

```

总体上说，在库函数的层面，有不少函数可以获得时间信息。在真实环境上，我们可以用 Frida 对它们逐个 Hook、拦截和替换。同理，可以在 Unidbg 中 Hook gettimeofday 函数并固定它，但这不是最好的办法。

在获取随机数这件事上也类似，有各种各样的 random 函数可用，不容易在库函数的层面做全面拦截。

### 1.3.3 系统调用

样本可以通过内联汇编直接调用系统调用，而且绝大多数干扰项所对应的库函数也都基于系统调用，因此如果拦截和处理系统调用，就可以从根本上处理随机干扰项。

比如 gettimeofday 库函数，它其实只是底层同名系统调用的简单封装。

![](https://xiaopang-1256027372.cos.ap-shenzhen-fsi.myqcloud.com/1663722130840-8b8e8865-54eb-4bdf-9353-0681310ba187.webp)  

clock_gettime 也同理。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
unsigned int __fastcall clock_gettime(clockid_t a1, struct timespec *a2)
{
  unsigned int result; // r0
​
  result = linux_eabi_syscall(__NR_clock_gettime, a1, a2);
  if ( result > 0xFFFFF000 )
    result = j___set_errno_internal(-result);
  return result;
}

```

在 Unidbg 中，这些系统调用都由 Unidbg 自实现，因此可以很方便的做固定或修改。

首先是 gettimeofday，来到`src/main/java/com/github/unidbg/unix/UnixSyscallHandler.java`。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
protected int gettimeofday(Emulator<?> emulator, Pointer tv, Pointer tz) {
    if (log.isDebugEnabled()) {
        log.debug("gettimeofday tv=" + tv + ", tz=" + tz);
    }
​
    if (log.isDebugEnabled()) {
        byte[] before = tv.getByteArray(0, 8);
        Inspector.inspect(before, "gettimeofday tv=" + tv);
    }
    if (tz != null && log.isDebugEnabled()) {
        byte[] before = tz.getByteArray(0, 8);
        Inspector.inspect(before, "gettimeofday tz");
    }
​
    long currentTimeMillis = System.currentTimeMillis();
    long tv_sec = currentTimeMillis / 1000;
    long tv_usec = (currentTimeMillis % 1000) * 1000;
    TimeVal32 timeVal = new TimeVal32(tv);
    timeVal.tv_sec = (int) tv_sec;
    timeVal.tv_usec = (int) tv_usec;
    timeVal.pack();
​
    if (tz != null) {
        Calendar calendar = Calendar.getInstance();
        int tz_minuteswest = -(calendar.get(Calendar.ZONE_OFFSET) + calendar.get(Calendar.DST_OFFSET)) / (60 * 1000);
        TimeZone timeZone = new TimeZone(tz);
        timeZone.tz_minuteswest = tz_minuteswest;
        timeZone.tz_dsttime = 0;
        timeZone.pack();
    }
​
    if (log.isDebugEnabled()) {
        byte[] after = tv.getByteArray(0, 8);
        Inspector.inspect(after, "gettimeofday tv after tv_sec=" + tv_sec + ", tv_usec=" + tv_usec + ", tv=" + tv);
    }
    if (tz != null && log.isDebugEnabled()) {
        byte[] after = tz.getByteArray(0, 8);
        Inspector.inspect(after, "gettimeofday tz after");
    }
    return 0;
}

```

直接将`System.currentTimeMillis()`改为固定值即可固定 gettimeofday 系统调用。

clock_gettime 在 Unidbg 中则在`src/main/java/com/github/unidbg/linux/ARM64SyscallHandler.java`，如果样本采用 32 位环境，则对应于`src/main/java/com/github/unidbg/linux/ARM32SyscallHandler.java`。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
private final long nanoTime = System.nanoTime();
​
protected int clock_gettime(Backend backend, Emulator<?> emulator) {
    int clk_id = backend.reg_read(ArmConst.UC_ARM_REG_R0).intValue();
    Pointer tp = UnidbgPointer.register(emulator, ArmConst.UC_ARM_REG_R1);
    long offset = clk_id == CLOCK_REALTIME ? System.currentTimeMillis() * 1000000L : System.nanoTime() - nanoTime;
    long tv_sec = offset / 1000000000L;
    long tv_nsec = offset % 1000000000L;
    if (log.isDebugEnabled()) {
        log.debug("clock_gettime clk_id=" + clk_id + ", tp=" + tp + ", offset=" + offset + ", tv_sec=" + tv_sec + ", tv_nsec=" + tv_nsec);
    }
    switch (clk_id) {
        case CLOCK_REALTIME:
        case CLOCK_MONOTONIC:
        case CLOCK_MONOTONIC_RAW:
        case CLOCK_MONOTONIC_COARSE:
        case CLOCK_BOOTTIME:
            tp.setInt(0, (int) tv_sec);
            tp.setInt(4, (int) tv_nsec);
            return 0;
        case CLOCK_THREAD_CPUTIME_ID:
            tp.setInt(0, 0);
            tp.setInt(4, 1);
            return 0;
    }
    throw new UnsupportedOperationException("clk_id=" + clk_id);
}

```

其中的时间获取是`System.currentTimeMillis()`和`System.nanoTime()`，可以同理做固定。

在随机数这方面，有一个系统调用叫 getrandom，原型如下，会返回一个随机数组。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
ssize_t getrandom(void *buf, size_t buflen, unsigned int flags);

```

Unidbg 的实现位于`src/main/java/com/github/unidbg/unix/UnixSyscallHandler.java`，同理可以将 Random 改成固定值。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
protected int getrandom(Pointer buf, int bufSize, int flags) {
    Random random = new Random();
    byte[] bytes = new byte[bufSize];
    random.nextBytes(bytes);
    buf.write(0, bytes, 0, bytes.length);
    if (log.isDebugEnabled()) {
        log.debug(Inspector.inspectString(bytes, "getrandom buf=" + buf + ", bufSize=" + bufSize + ", flags=0x" + Integer.toHexString(flags)));
    }
    return bufSize;
}

```

在处理时间戳和随机数相关的系统调用时，Unidbg 并没有什么” 小心机 “，就像处理其他系统那样，Unidbg 采用了一种严肃和公正的策略。和 Unidbg 态度类似的是 Qiling，比如它对 getrandom 的处理如下，同样是按照语义返回随机数组。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
def ql_syscall_getrandom(ql: Qiling, buf: int, buflen: int, flags: int):
    try:
        data = os.urandom(buflen)
        ql.mem.write(buf, data)
    except:
        retval = -1
else:
    ql.log.debug(f'getrandom() CONTENT: {data.hex(" ")}')
    retval = len(data)
​
    return retval

```

和 Unidbg 同类型的另一个工具 —— ExAndroidNativeEmu，则在这项工作上展示了很大的灵活性。在 getrandom 实现上，它采取了硬编码，全部填充`01`，即主动的减少随机情况。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
def _getrandom(self, mu, buf, count, flags):
    mu.mem_write(buf, b"\x01" * count)
    return count

```

在 gettimeofday 以及 clock_gettime 上，它直接考虑了重写或者说固定它俩的情况。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
// path：androidemu/cpu/syscall_hooks.py
OVERRIDE_TIMEOFDAY = False
OVERRIDE_TIMEOFDAY_SEC = 0
OVERRIDE_TIMEOFDAY_USEC = 0
​
OVERRIDE_CLOCK = False
OVERRIDE_CLOCK_TIME = 0

```

比如将 OVERRIDE_TIMEOFDAY 设置为 true，那么在处理 gettimeofday 时，就会使用我们设置的 SEC、USEC 而不是正常的时间访问。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
def _handle_gettimeofday(self, uc, tv, tz):
    """
    If either tv or tz is NULL, the corresponding structure is not set or returned.
    """
​
    if tv != 0:
        ptr_sz = self.__emu.get_ptr_size()
        if OVERRIDE_TIMEOFDAY:
            uc.mem_write(tv + 0, int(OVERRIDE_TIMEOFDAY_SEC).to_bytes(ptr_sz, byteorder='little'))
            uc.mem_write(tv + ptr_sz, int(OVERRIDE_TIMEOFDAY_USEC).to_bytes(ptr_sz, byteorder='little'))
        else:
            timestamp = time.time()
            (usec, sec) = math.modf(timestamp)
            usec = abs(int(usec * 100000))
​
            uc.mem_write(tv + 0, int(sec).to_bytes(ptr_sz, byteorder='little'))
            uc.mem_write(tv + ptr_sz, int(usec).to_bytes(ptr_sz, byteorder='little'))
​
    if tz != 0:
        #timezone结构体不64还是32都是两个4字节成员
        uc.mem_write(tz + 0, int(-120).to_bytes(4, byteorder='little'))  # minuteswest -(+GMT_HOURS) * 60
        uc.mem_write(tz + 4, int().to_bytes(4, byteorder='little'))  # dsttime
​
    return 0

```

相比之下，Unidbg 中想要修改、固定时间、随机数等系统调用，就必须修改源码，或者自实现 SyscallHandler，然后重写自己感兴趣的方法。

### 1.3.4 文件访问

比如直接访问 /dev/urandom、/dev/random 读取字节流，实现随机数的获取。

在 Unidbg 中，其代码逻辑位于`src/main/java/com/github/unidbg/linux/file/DriverFileIO.java`。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
public static DriverFileIO create(Emulator<?> emulator, int oflags, String pathname) {
    if ("/dev/urandom".equals(pathname) || "/dev/random".equals(pathname) || "/dev/srandom".equals(pathname)) {
        return new RandomFileIO(emulator, pathname);
    }
    if ("/dev/alarm".equals(pathname) || "/dev/null".equals(pathname)) {
        return new DriverFileIO(emulator, oflags, pathname);
    }
    if ("/dev/ashmem".equals(pathname)) {
        return new Ashmem(emulator, oflags, pathname);
    }
    if ("/dev/zero".equals(pathname)) {
        return new ZeroFileIO(emulator, oflags, pathname);
    }
    return null;
}
​

```

其实现逻辑位于`src/main/java/com/github/unidbg/linux/file/RandomFileIO.java`。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
protected void randBytes(byte[] bytes) {
    ThreadLocalRandom.current().nextBytes(bytes);
}

```

读者可以自行做固定，就像处理系统调用那样，将它硬编码。

在 ExAndroidNativeEmu 中，则提供了和时间戳类似的 OverRide，位于`androidemu/vfs/file_system.py`。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
OVERRIDE_URANDOM = False
OVERRIDE_URANDOM_INT = 1
​
if filename == '/dev/urandom':
    logging.debug("File opened '%s'" % filename)
    parent = os.path.dirname(file_path)
    if (not os.path.exists(parent)):
        os.makedirs(parent)
        #
    with open(file_path, "wb") as f:
        ran = OVERRIDE_URANDOM_INT
        if (not OVERRIDE_URANDOM):
            ran = random.randint(1, 1<<128)
        b = int(ran).to_bytes(128, byteorder='little')
        f.write(b)

```

> 依然是引用龙哥文章

2.1 自上而下
--------

自上而下是最朴素的方法，从目标函数的起始地址往下调试代码，分析清楚遇到的每一个函数的功能。这个方法论最朴素，也可以获得对目标函数最完全的理解。理论上，我们总是应该采用这个思路。 但在实践上，它并不总是最优解，有两个原因。

1.  1.
    
    在做安全开发和代码保护时，开发者默认分析者会自上而下去做分析，所以也会格外注重保护头部。比如我们可以观察到，很多样本都会在 JNI_OnLoad、目标函数开头等位置塞大量的垃圾指令，阻止反汇编和反编译，但很多子函数或者说中间函数不会做这类保护。换句话说，自上而下分析，意味着你需要直面样本存在的所有代码保护技术。
    
2.  2.
    
    从代码逻辑上说，很多目标函数可以分成三段：环境校验、信息收集、加密与编码。如果我们只是希望还原算法，那么很多时候，只需要确认第三段的输入，以及第三段的具体处理就行了，前两段不需要特别仔细的看。但自上而下的分析处理里，需要花费一些时间在前两段上。 自上而下分析是全面、详尽、耗时的分析办法，我们一般可以把它当成保底的办法。
    

2.2 由果溯因
--------

在 Unidbg 这类分析能力很强的运行环境里，由果溯因是我很喜欢的一个分析办法。这意味着我们直接从函数的结果开始分析，往上找它的来源，来源找清楚了为止。 相比较自上而下，它要分析的逻辑量比较小，只有 1/3 甚至更少，因为和产生运算结果无关的逻辑都被过滤掉了。想要快速的由果溯因，还需要掌握一些小技巧。 它的优点是分析快速，可以避开许多无用逻辑和代码保护，缺点是分析不够全面，而且对分析工具的要求比较高，至少得有内存读写监控。

2.3 关键函数分析
----------

关键函数分析也是常见的一种分析思路，如果说自上而下是顺着逻辑往下分析，由果溯因是从逻辑下游往上游追，关键函数分析就是在执行流上开了很多个小窗口。 我们最多选择的 “关键” 函数是加密函数和工具函数。 加密函数：程序的运算过程主要由标准加密算法构成，所以只要找到这些标准加密算法的实现函数，分析清楚参数和返回值，然后 hook 打印，算法就已经分析出了七七八八。 工具函数：几乎没有程序离得开工具函数，比如`memcpy`、`memcmp`、`strstr`、`strlen`这些函数，hook 它们，就可以一窥程序的数据流和字符串流。如果样本自实现了这些工具函数，而非使用库函数，那么我们要找到并 hook 其自实现函数。 这个分析方法的好处是快速，很多时候比由果溯因都更快，但它是一种更加不全面的分析方法，而且需要分析者对加密算法有较深的理解。

好了, 大家在拜读完龙哥的文章以后, 相信对干扰项和算法分析的方法论有了一个简单的了解. 接下来, 我们继续用 Shopee 来巩固我们所汲取的知识.  
 

3.1 固定随机数
---------

基于上文中的文件访问, 我们把 Unidbg 中的文件访问给启动起来.

*   增加 `IOResolver<AndroidFileIO>`
    

现在我迫不及待的想打印我们

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
public class wvvvuwwu  extends AbstractJni implements IOResolver<AndroidFileIO> {
    public  wvvvuwwu(){
        emulator = AndroidEmulatorBuilder.for64Bit().addBackendFactory(new DynarmicFactory(true)).build();
​
        Memory memory = emulator.getMemory();
        //作者支持19和23两个sdk
        memory.setLibraryResolver(new AndroidResolver(23));
        //创建DalvikVM，利用apk本身，可以为null
        //如果用apk文件加载so的话，会自动处理签名方面的jni，具体可看AbstractJni,这就是利用apk加载的好处
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\resources\\shopee\\com.shopee.my.apk"));
//        vm = emulator.createDalvikVM(null);
        vm.setJni(this);
        vm.setVerbose(true);
        // 开启IO重定向  !!!!!!!!!!!!!!!!!!!!! 注意, 要加这行,否则不生效哦.
        emulator.getSyscallHandler().addIOResolver(this);
​
        //加载so，使用armv8-64速度会快很多
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\resources\\shopee\\libshpssdk.so"),true);
        //调用jni
        dm.callJNI_OnLoad(emulator);
        module = dm.getModule();
        //Jni调用的类，加载so
        cWvvvuwwu = vm.resolveClass("com/shopee/shpssdk/wvvvuwwu");
    }
​
@Override
    public FileResult<AndroidFileIO> resolve(Emulator<AndroidFileIO> emulator, String pathname, int oflags) {
        System.out.println("file open:"+pathname);
        return null;
    }
}
​

```

从日志中, 我们可以看到, 它执行了多次`/dev/urandom`的文件读取操作

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
file open:/dev/__properties__
file open:/proc/stat
file open:/sbin/su
file open:/vendor/bin/su
file open:/system/sbin/su
file open:/system/bin/su
file open:/system/xbin/su
file open:/data/local/su
file open:/data/local/bin/su
file open:/data/local/xbin/su
file open:/sbin/su
file open:/system/bin/su
file open:/system/bin/.ext/su
file open:/system/bin/failsafe/su
file open:/system/sd/xbin/su
file open:/su/xbin/su
file open:/su/bin/su
file open:/magisk/.core/bin/su
file open:/system/usr/we-need-root/su
file open:/system/xbin/su
file open:/system/sbin/su
file open:/vendor/bin/su
file open:/su
file open:/system/lib64/libzygisk.so
file open:/dev/urandom
file open:/data/app/~~oONxdR2wXsaVN2qQTbgsSg==/com.shopee.my-jns-B-GoPqIjRs55vzq5WA==
file open:/dev/urandom
file open:/dev/urandom
file open:/proc/cpuinfo
file open:/dev/urandom
file open:/dev/urandom
file open:/dev/urandom
file open:/dev/urandom

```

所以, 我们现在可以把`/dev/urandom`的值暂时给固定住.

![](https://xiaopang-1256027372.cos.ap-shenzhen-fsi.myqcloud.com/image-20241213190356392.png)

 把 randBytes 的代码给屏蔽掉, 就可以把随机数给固定成 0,0,0,0 了.

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
package com.github.unidbg.linux.file;
​
import com.github.unidbg.Emulator;
import com.github.unidbg.arm.backend.Backend;
import com.github.unidbg.file.linux.IOConstants;
import com.sun.jna.Pointer;
​
import java.util.concurrent.ThreadLocalRandom;
​
public class RandomFileIO extends DriverFileIO {
​
    public RandomFileIO(Emulator<?> emulator, String path) {
        super(emulator, IOConstants.O_RDONLY, path);
    }
​
    @Override
    public int read(Backend backend, Pointer buffer, int count) {
        int total = 0;
        byte[] buf = new byte[Math.min(0x1000, count)];
        randBytes(buf);
        System.out.println(toHex(buf));
        Pointer pointer = buffer;
        while (total < count) {
            int read = Math.min(buf.length, count - total);
            pointer.write(0, buf, 0, read);
            total += read;
            pointer = pointer.share(read);
        }
        return total;
    }
    public static String toHex(byte[] bytes) {
        StringBuilder hexString = new StringBuilder();
        for (byte b : bytes) {
            // 使用 String.format 将字节转换为两位十六进制表示
            hexString.append(String.format("%02X ", b));
        }
        return hexString.toString().trim(); // 去掉末尾多余的空格
    }
    protected void randBytes(byte[] bytes) {
//        ThreadLocalRandom.current().nextBytes(bytes);
    }
}
​

```

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
{"253b8c85": "t/AOoDgQjRflcu8CpUyCpUyCpUy=", "54006a5d": "j/KroHmED6/1cO5jsAwsekhQsSQ=", "7445bb38": "HEmZ77UhssssssssGjh7SdDI4ECew8Ni0CXZ2bqPHq/YYAXe2HFEaAEhgNmv0Q6S6zthC3obx382kCDQOqPOES3fzYA3i2B2ZBTz8BoUh2bl9K7g+Fpf13w1SakKXfJ46LXdHao4YU9/KoL403WHZ7P+viglsqWLeLtJ2Hhz5kPSLFZ0KHteHQb8QEBxNWgV7yAB1bT0tlCLg/PBMMD3BH3hzigQyFcRnLFWSgiG/bcaocJ8JtEgUipW9YQ5JcPYwQoNV/OfyOwHO/VatOh3Vt6jFkWtUkb71kD9hf4X2AzscdYhGE+SD3Clb20H3VHc5aLYPKbAdWF5EUJt5u9frToquzheHLPofeJjIqe46B+MjvE+FjjKNI7xByn96Dr28PAO6ficw7w0tUVJUBNDCrifz6Nms2IqMqOa6EpnH3Ct0k9eyxC7SCycW9juDRUY78teyGOJqd6zVwJ54MsypCGoaznmHHQrmdxrHH+0estvcQxngGgxVkBMdgBxe2ASOHww3aylOQlLWznMEAjiuKmqKsvrha8h9iQqzl1hMu0TRr7qX6Di+0GyEICOY0SyNULYyki7lend67dDpuYOwBf3+W8BD2fD9bpmwIyQ7GGJyQA4SgPV0r9FSmSalb6KnHgUp6iyArkud/jGYM5upLM2t84=", "x-sap-ri": "da155c6700000000000000100100000000000000000000000000"}
​
{"253b8c85": "t/AOoDgQjRflcu8CpUyCpUyCpUy=", "54006a5d": "T1D0AsTaPNqJ/52YsGssekhQsSQ=", "7445bb38": "XS+zJCUhssssssssW6QP9uqT4JO88V+qJQVBT5K3rJyOCjigavaBzYZCv0UMcV5p8M81CJK5Gio/F1nPuYwb//GT6UsZ+uUB4lmxU187RxGnRcyhDhjmlbSqwIcZApRR0oNMWLje3EP4C+9I5CAFtaelujQO1bUzEhR1PGqAOqt1LhqHhcyYAORH4nlrxmIw75xbim1mfQ/cIQuesumawvaBdvQ0syJBDHJBiEuW83rsYiZXkjW5M1hg3/co29fxryoJ6RHEw/5UMsDobutLzhFuvNLjFqckw1WoDXaeDJYFsyKLD258wzMb6y3XLYxqkgbhIQa2VXsNCW6wCLIZGhb8EKth6sVdPlf2FHPHUwnyEH7Bawueb8ZyJQ2ttPivGeofxlmFjvVxra4567mb8ywlQxkZOXicU20ehCiTeLyYoF+5ApqiuawUBjW++FEHyOyjEF0FDYENz0tUcKqPiIyCJRWvdWtIJODCwvyIKlssrDAnBExaFBsfg7yabK0jPBMbSi1m600C6lPxSJhJS2IgiWv2nIkCCH1Ks33qfr2erJziIZCm6VGkqzebewQkBcJPIW6+cK+iY4mslR/gjNtOqtTzC9iV1E4VeaXThlrsoWs74xeIPpRi/QgJYI52dVnBCMzCxqOhEMgYBT4yiToRrsT=", "x-sap-ri": "f0155c6700000000000000100100000000000000000000000000"}
​

```

发现我们的结果还是会变化, 那就还有意味着还有其他的可能在干扰我们. 但是, 目前我们至少已经把`文件访问`的给解决了

 我们直接跳转到`src/main/java/com/github/unidbg/unix/UnixSyscallHandler.java`中, 寻找到会产生变化的数, 统统的打上断点

![](https://xiaopang-1256027372.cos.ap-shenzhen-fsi.myqcloud.com/image-20241213191734428.png)  

通过堆栈, 可以发现是调用了`clock_gettime`, 那么我直接把时间戳固定成`1734088737626L`, 同时输出一下, 别以后忘记了.

![](https://xiaopang-1256027372.cos.ap-shenzhen-fsi.myqcloud.com/image-20241213191946042.png)  

此时, 通过多次的调用, 发现我们已经成功的把随机数给固定了.

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
{"253b8c85": "t/AOoDgQjRflcu8CpUyCpUyCpUy=", "54006a5d": "5DY2BH6xbvG5bB4/soEsSshQsSQ=", "7445bb38": "JUzmVgUhssssssssQdlQxFFfZGYqSaSgzUR+76C5RLev5MRK2RI7yAuYBKAXrGYJbCOnGdXNPqYDmwAcbi1JwCbWl3oTdP2cdJ/dLIJ4k2iGnKBvxm3sqBXJct+9jJ1h8coPVIq9YEYomeQZE9yKcUxfAtuiMHHQ42KMMKA7yXhR+0QioVjE20SSJM1G4PaX1AbR1PP33pEjJ4NpqZP/n7n2gLJilpx8cPsgHjIHpk9mQqguWQLQl76CU8FgJ+wfIGSed22PKNUmdK5axBYGsOrDVOlC63hf01WI9wLVqNHLJ0xO2cJcpvoNSvbkJ8SlIyPHuzmykJCftqsAQliYc12i3chvXSZueWaVsF7nxEp3jmoJgrVitx7QQBfwcYH2hMY1mda5DvScagW2ZoshRrlX3DB1h2Pu8wlCMw39n8fo9tEFJvHTTxgM3Xf5YS0bYs4hmr1iwrAdNNQRy+1lSr4epiJQHfSVCXYcXx5yOlgP5avJ8tYoCSt93QyRZNksZBDAbSuOCp+Wy3bv7nCuE87R0cvSgCv6CNKV6bv4Y5BCUpe+9SCaEs7Sn1OmAI6u/FB+NRdoq3vR8x/n0PGDVZNjAXXkAVtHBy8Iw2hFduWfHqug6v8wmi++SAfzbnpk16GSGUD8+HksBUnwuLptU+wMlpE=", "x-sap-ri": "21185c6700000000000000100100000000000000000000000000"}
​
{"253b8c85": "t/AOoDgQjRflcu8CpUyCpUyCpUy=", "54006a5d": "5DY2BH6xbvG5bB4/soEsSshQsSQ=", "7445bb38": "JUzmVgUhssssssssQdlQxFFfZGYqSaSgzUR+76C5RLev5MRK2RI7yAuYBKAXrGYJbCOnGdXNPqYDmwAcbi1JwCbWl3oTdP2cdJ/dLIJ4k2iGnKBvxm3sqBXJct+9jJ1h8coPVIq9YEYomeQZE9yKcUxfAtuiMHHQ42KMMKA7yXhR+0QioVjE20SSJM1G4PaX1AbR1PP33pEjJ4NpqZP/n7n2gLJilpx8cPsgHjIHpk9mQqguWQLQl76CU8FgJ+wfIGSed22PKNUmdK5axBYGsOrDVOlC63hf01WI9wLVqNHLJ0xO2cJcpvoNSvbkJ8SlIyPHuzmykJCftqsAQliYc12i3chvXSZueWaVsF7nxEp3jmoJgrVitx7QQBfwcYH2hMY1mda5DvScagW2ZoshRrlX3DB1h2Pu8wlCMw39n8fo9tEFJvHTTxgM3Xf5YS0bYs4hmr1iwrAdNNQRy+1lSr4epiJQHfSVCXYcXx5yOlgP5avJ8tYoCSt93QyRZNksZBDAbSuOCp+Wy3bv7nCuE87R0cvSgCv6CNKV6bv4Y5BCUpe+9SCaEs7Sn1OmAI6u/FB+NRdoq3vR8x/n0PGDVZNjAXXkAVtHBy8Iw2hFduWfHqug6v8wmi++SAfzbnpk16GSGUD8+HksBUnwuLptU+wMlpE=", "x-sap-ri": "21185c6700000000000000100100000000000000000000000000"}
​

```

3.2 x-sap-ri
------------

### 3.2.1 后 44 位

在前文中, 我们学习了算法分析的三大方法论, 我们可以尝试用关键函数分析, 对`memcpy`、`memcmp`、`strstr`、`strlen`这几个方法进行监听.

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
emulator.attach().addBreakPoint(module.findSymbolByName("memcpy").getAddress(), new BreakPointCallback() {
        @Override
        public boolean onHit(Emulator<?> emulator, long address) {
            RegisterContext registerContext =emulator.getContext();
            UnidbgPointer src = registerContext.getPointerArg(1);
            int length = registerContext.getIntArg(2);
            Inspector.inspect(src.getByteArray(0, length), "memcpy:"+src.toString());
            return true;
        }
    });

```

   
看看我们发现了什么?

![](https://xiaopang-1256027372.cos.ap-shenzhen-fsi.myqcloud.com/image-20241213234959519.png)  

同时, 还在前面通过 / dev/urandom 发现, 有一些看似与`x-sap-ri`有必然联系的数据.

![](https://xiaopang-1256027372.cos.ap-shenzhen-fsi.myqcloud.com/image-20241213235242737.png)  

那么我们此刻应该想办法确定, 这些随机数与我们的结果到底有着怎么样的联系?

 此时, 需要把 RandomFileIO 进一步的改造. 主要是为了让他每次的随机数 **`有点不同`**

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
package com.github.unidbg.linux.file;
​
import com.github.unidbg.Emulator;
import com.github.unidbg.arm.backend.Backend;
import com.github.unidbg.file.linux.IOConstants;
import com.sun.jna.Pointer;
​
import java.util.concurrent.ThreadLocalRandom;
​
public class RandomFileIO extends DriverFileIO {
    int num = 0;
    public RandomFileIO(Emulator<?> emulator, String path) {
        super(emulator, IOConstants.O_RDONLY, path);
    }
​
    @Override
    public int read(Backend backend, Pointer buffer, int count) {
        int total = 0;
        byte[] buf = new byte[Math.min(0x1000, count)];
        randBytes(buf);
        // buf = new byte[]{0x1,0x1,0x1,0x1};
        buf = new byte[]{(byte) (num), (byte) 0x00, 0x00, (byte) (0xf0+num)};
        num+=1;
        System.out.println("toHex:" + num + " " + toHex(buf));
        Pointer pointer = buffer;
        while (total < count) {
            int read = Math.min(buf.length, count - total);
            pointer.write(0, buf, 0, read);
            total += read;
            pointer = pointer.share(read);
        }
        return total;
    }
    public static String toHex(byte[] bytes) {
        StringBuilder hexString = new StringBuilder();
        for (byte b : bytes) {
            // 使用 String.format 将字节转换为两位十六进制表示
            hexString.append(String.format("%02X ", b));
        }
        return hexString.toString().trim(); // 去掉末尾多余的空格
    }
    protected void randBytes(byte[] bytes) {
//        ThreadLocalRandom.current().nextBytes(bytes);
    }
}
​

```

于是乎, 我们可以发现个非常精细的结论, 我们的目标结果与我们随机数的第一个字节是有关的.

![](https://xiaopang-1256027372.cos.ap-shenzhen-fsi.myqcloud.com/image-20241214144558932.png)  

我们可以进一步的验证. 此时我们需要做的就是把随机数的功能给复原回去.

![](https://xiaopang-1256027372.cos.ap-shenzhen-fsi.myqcloud.com/image-20241214145043845.png)  

再次仔细观察下, 还是有点小细节需要注意的

看到我重复划线的地方. 这几个对不上

也就是第 8,9 次随机数. 由于是随机数, 我们或许没必要太过于在意

![](https://xiaopang-1256027372.cos.ap-shenzhen-fsi.myqcloud.com/image-20241214145439862.png)  

在经过多次模拟执行以后, 得到以下结论

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
21185c67
93d2e9772b93d71f014a3bc22397f7f5add37f6f7316

```

> 这里稍微跟如画大佬的文章有些许出入. 有兴趣的可以进一步的验证猜想

### 3.2.2 前 8 位

到目前为止, 我们前面 8 位还不知是什么? 只知道我们在固定随机数的时候, 前面 8 个字节是由 clock_gettime 的结果所影响. 当我把时间戳改为 0 的时候, 结果变成了 00000000

![](https://xiaopang-1256027372.cos.ap-shenzhen-fsi.myqcloud.com/image-20241214151741014.png)  

猜不出来, 它与时间的关系, 于是我们只能思考着进入 IDA 中看代码逻辑了. 打印下 memcpy 的堆栈.

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
emulator.attach().addBreakPoint(module.findSymbolByName("memcpy").getAddress(), new BreakPointCallback() {
            @Override
            public boolean onHit(Emulator<?> emulator, long address) {
                RegisterContext registerContext =emulator.getContext();
                UnidbgPointer src = registerContext.getPointerArg(1);
                int length = registerContext.getIntArg(2);
                Inspector.inspect(src.getByteArray(0, length), "memcpy:"+src.toString());
                emulator.getUnwinder().unwind();
                return true;
            }
        });

```

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
>-----------------------------------------------------------------------------<
[15:30:26 235]memcpy:unidbg@0xbffff044, md5=67e94e4ff6e091489d33c2cff29067b6, hex=21185c67
size: 4
0000: 21 18 5C 67                                        !.\g
^-----------------------------------------------------------------------------^
[0x040000000][0x040084a98][libshpssdk.so][0x084a98]
[0x040000000][0x040084228][libshpssdk.so][0x084228]
[0x040000000][0x040083d58][libshpssdk.so][0x083d58]
[0x040000000][0x04009122c][libshpssdk.so][0x09122c]
[0x040000000][0x04009a2a8][libshpssdk.so][0x09a2a8]
​

```

我现在怀疑, 它就是 memcpy, 但是这个地址我无法跳转, 所以我们需要在此处下断点

![](https://xiaopang-1256027372.cos.ap-shenzhen-fsi.myqcloud.com/image-20241214153557084.png)  

![](https://xiaopang-1256027372.cos.ap-shenzhen-fsi.myqcloud.com/image-20241214153700079.png)  

我们在此处下断点.

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
emulator.attach().addBreakPoint(module.base + 0x84A98, new BreakPointCallback() {
            @Override
            public boolean onHit(Emulator<?> emulator, long address) {
                return false;
            }
        });

```

跳转到 0x73158 地址看看

![](https://xiaopang-1256027372.cos.ap-shenzhen-fsi.myqcloud.com/image-20241214153721329.png)  

非常棒, 这验证了我们的猜想

![](https://xiaopang-1256027372.cos.ap-shenzhen-fsi.myqcloud.com/image-20241214153751938.png)  

根据 memcpy 的定位

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
void *memcpy(void *dest, const void *src, size_t n)

```

所以参数 3 v7[11] (x2) 就是我们的数据来源, 参数 4(x3) 就是大小

> arm64 的参数传递:
> 
> // 前 8 个参数使用通用寄存器
> 
> X0-X7: 整数 / 指针参数 X0: 第 1 个参数 X1: 第 2 个参数 X2: 第 3 个参数 X3: 第 4 个参数 X4: 第 5 个参数 X5: 第 6 个参数 X6: 第 7 个参数 X7: 第 8 个参数
> 
> // 浮点参数使用 V0-V7: 浮点参数
> 
> X0: 用于返回整数 / 指针值
> 
> // 超过 8 个参数的情况
> 
> [SP + 0]  第 9 个参数 
> 
> [SP + 8]  第 10 个参数 
> 
> [SP + 16]  第 11 个参数 
> 
> // 以此类推

我们通过 unidbg 也可以观察出来

![](https://xiaopang-1256027372.cos.ap-shenzhen-fsi.myqcloud.com/image-20241214154441443.png)  

那么此刻, 我们应该追踪 x2

![](https://xiaopang-1256027372.cos.ap-shenzhen-fsi.myqcloud.com/image-20241214154503967.png)  

我们直接对当前函数进行 trace

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
String traceFile = "unidbg-android/src/test/java/com/shopee/shpssdk/tracecode.txt";
PrintStream traceStream;
try {
    traceStream = new PrintStream(new FileOutputStream(traceFile), true);
    emulator.traceCode(module.base + 0x84224, 0x8422C ).setRedirect(traceStream);
} catch (FileNotFoundException e) {
    e.printStackTrace();
}

```

   
断点打在获取完时间, 然后一步步 unidbg 调试下去.

当执行到下面所示的指令时, 我们会发现

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
=> *[libshpssdk.so*0x091190]*[56fd498b]*0x40091190:*"add x22, x10, x9, lsr #63"

```

可以看到 x10 的值`0x675c1821` 和我们的前 8 位置非常相似 `21 18 5c 67`,  
 

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
>>> x0=0x62924ff371790 x1=0xbffff208 x2=0x4058e000 x3=0x4051a864 x4=0x4051a864 x5=0x15 x6=0x16 x7=0x40556000 x8=0x1 x9=0x19d7060868106 x10=0x675c1821 x11=0x0 x12=0x0 x13=0x40596000 x14=0x0
>>> x15=0x40588000 x16=0x4035b460 x17=0x404cd854 x18=0x40599f50 x19=0xbffff580 x20=0x200 x21=0xbffff598 x22=0xbffff5b8 x23=0x4035ca08 x24=0x406fd000 x25=0x406fd200 x26=0xbffff2f0 x27=0xbffff718 x28=0x178c fp=0xbffff540
>>> q0=0x220000000000000031(2.4E-322, 1.7E-322) q1=0x180(5.3809861030072976E-43) q2=0x0(0.0) q3=0x70000000000000006(3.0E-323, 3.5E-323) q4=0x10000000000000001(4.9E-324, 4.9E-324) q5=0x40000000000000004(2.0E-323, 2.0E-323) q6=0x20000000000000002(1.0E-323, 1.0E-323) q7=0x1310000000000000131(1.507E-321, 1.507E-321) q8=0x0(0.0) q9=0x0(0.0) q10=0x0(0.0) q11=0x0(0.0) q12=0x0(0.0) q13=0x0(0.0) q14=0x0(0.0) q15=0x0(0.0)
>>> q16=0x51310000000000004131(8.2455E-320, 1.0269E-319) q17=0x0(0.0) q18=0x51310000000000004131(8.2455E-320, 1.0269E-319) q19=0x0(0.0) q20=0x0(0.0) q21=0x0(0.0) q22=0x0(0.0) q23=0x0(0.0) q24=0x0(0.0) q25=0x0(0.0) q26=0x0(0.0) q27=0x0(0.0) q28=0x0(0.0) q29=0x0(0.0) q30=0x0(0.0) q31=0x0(0.0)
​

```

那么此时, 我们可以确定, 我们的前面 8 位是由时间经过某种运算而生成的. 然后我们还没有看到的操作应该就是把 x10 进行大小端序的转换成小端序的排序即可.

 下面是 trace 指令, 我们把它弄懂即可. 此处我把时间戳固定成了 1234567890123L

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
// x0 = 1234567890123L
[17:45:49 353][libshpssdk.so 0x2784e4] [00250a9b] 0x402784e4: "madd x0, x8, x10, x9" x8=0x499602d2 x10=0xf4240 x9=0x1e078 => x0=0x462d53c8ab8f8  
// sp = sp + 0x40
[17:45:49 353][libshpssdk.so 0x2784e8] [ff030191] 0x402784e8: "add sp, sp, #0x40" sp=0xbffff200 => sp=0xbffff240
[17:45:49 354][libshpssdk.so 0x2784ec] [c0035fd6] 0x402784ec: "ret"
// x9 = 0x34db;  // 完全覆盖原值
[17:45:49 354][libshpssdk.so 0x09116c] [699b86d2] 0x4009116c: "movz x9, #0x34db" x9=0x1e078 => x9=0x34db
// 
[17:45:49 355][libshpssdk.so 0x091170] [571600f0] 0x40091170: "adrp x23, #0x4035c000" => x23=0x4035c000
// 移位立即数加载指令
// 将 0xd7b6 左移16位 (变成 0xd7b60000)
// 只修改 x9 的第16-31位
// Before:
// x9 = 0000 0000 0000 34db
//                     |保持不变|
// After:
// x9 = 0000 d7b6 0000 34db
//          |修改| |保持不变|
[17:45:49 355][libshpssdk.so 0x091174] [c9f6baf2] 0x40091174: "movk x9, #0xd7b6, lsl #16" x9=0x34db => x9=0xd7b634db
// x23 = x23 + 0xa08          
//     = 0x4035c000 + 0xa08
//     = 0x4035ca08
[17:45:49 356][libshpssdk.so 0x091178] [f7222891] 0x40091178: "add x23, x23, #0xa08" x23=0x4035c000 => x23=0x4035ca08
// 把0xde82 左移32位  (变成 0xde82_0000_0000)
// 只修改 x9 的第32-47位
// Before:
// x9 = xxxx xxxx d7b6 34db
//                 |保持不变|
// After:
// x9 = xxxx de82 d7b6 34db
//          |修改| |保持不变|
[17:45:49 356][libshpssdk.so 0x09117c] [49d0dbf2] 0x4009117c: "movk x9, #0xde82, lsl #32" x9=0xd7b634db => x9=0xde82d7b634db
// 带内存屏障的字节加载指令
// 从x23读取一个字节, 把这个字节加载到w8 ,其他清零
// 内存 0x4035ca08: 01 xx xx xx
//                  |
//                  v
// w8: 0000 0000 0000 0001
// w8 = *(volatile uint8_t*)0x4035ca08;  // 带内存屏障
[17:45:49 356][libshpssdk.so 0x091180] [e8fedf08] 0x40091180: "ldarb w8, [x23]" w8=0x499602d2 x23=0x4035ca08 => w8=0x1
// 0x431b左移48位
// Before:
// x9 = xxxx de82 d7b6 34db
//          |保持不变|
// After:
// x9 = 431b de82 d7b6 34db
//      |修改| |保持不变|
// x9 = 0x431bde82d7b634db
[17:45:49 356][libshpssdk.so 0x091184] [6963e8f2] 0x40091184: "movk x9, #0x431b, lsl #48" x9=0xde82d7b634db => x9=0x431bde82d7b634db
// 有符号乘法高位  Signed Multiply High
// x9: 目标寄存器 x0: 第一个源操作数 (0x462d53c8ab8f8)  x9: 第二个源操作数 (0x431bde82d7b634db)
// 将两个64位数进行有符号乘法  取结果的高64位 存入x9
// x9 = 0x126580b487df3
[17:45:49 357][libshpssdk.so 0x091188] [097c499b] 0x40091188: "smulh x9, x0, x9" x0=0x462d53c8ab8f8 x9=0x431bde82d7b634db => x9=0x126580b487df3
//  算术右移指令
// 将 x9 右移 18位
// 保持符号位
// 结果存入 x10
[17:45:49 358][libshpssdk.so 0x09118c] [2afd5293] 0x4009118c: "asr x10, x9, #0x12" x10=0xf4240 x9=0x126580b487df3 => x10=0x499602d2

```

然后我们就可以把它丢给 GPT 生成代码了

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
def get_x_sap_ri_8():
    # 模拟 clock_gettime 获取时间戳
    timestamp = 1234567890123  # 0x462d53c8ab8f8
​
    # 构建64位常数
    x9 = 0x34db
    x9 = (x9 & 0xFFFF) | (0xd7b6 << 16)
    x9 = (x9 & 0xFFFFFFFF) | (0xde82 << 32)
    x9 = (x9 & 0xFFFFFFFFFFFF) | (0x431b << 48)
    # x9 = 0x431bde82d7b634db
​
    # 模拟从内存读取一个字节
    x8 = 1  # ldarb w8, [x23]
​
    # 128位乘法取高64位
    result = (timestamp * x9) >> 64  # smulh
​
    # 算术右移18位
    result = result >> 18  # asr
​
    return result
​
​
# 测试
result = clock_gettime_278478()
print(hex(result)) 

```

   
然后我们再把结果转换成小端序.

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
def get_x_sap_ri_8():
    # 模拟 clock_gettime 获取时间戳
    timestamp = 1734088737626360  # 0x462d53c8ab8f8
​
    # 构建64位常数
    x9 = 0x34db
    x9 = (x9 & 0xFFFF) | (0xd7b6 << 16)
    x9 = (x9 & 0xFFFFFFFF) | (0xde82 << 32)
    x9 = (x9 & 0xFFFFFFFFFFFF) | (0x431b << 48)
    # x9 = 0x431bde82d7b634db
​
    # 模拟从内存读取一个字节
    x8 = 1  # ldarb w8, [x23]
​
    # 128位乘法取高64位
    result = (timestamp * x9) >> 64  # smulh
​
    # 算术右移18位
    result = result >> 18  # asr = 0x499602d2
​
    # 转换为小端序字节
    little_endian = result.to_bytes(4, byteorder='little')
​
    return ''.join(f'{b:02x}' for b in little_endian)
​
​
# 测试
result = clock_gettime_278478()
print(result)  
#输出: 21185c67

```

### 3.2.3 总共 52 位代码

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

```
import random
​
​
def get_x_sap_ri_8():
    # 模拟 clock_gettime 获取时间戳
    timestamp = 1734088737626360  # 0x462d53c8ab8f8
​
    # 构建64位常数
    x9 = 0x34db
    x9 = (x9 & 0xFFFF) | (0xd7b6 << 16)
    x9 = (x9 & 0xFFFFFFFF) | (0xde82 << 32)
    x9 = (x9 & 0xFFFFFFFFFFFF) | (0x431b << 48)
    # x9 = 0x431bde82d7b634db
​
    # 模拟从内存读取一个字节
    x8 = 1  # ldarb w8, [x23]
​
    # 128位乘法取高64位
    result = (timestamp * x9) >> 64  # smulh
​
    # 算术右移18位
    result = result >> 18  # asr = 0x499602d2
​
    # 转换为小端序字节
    little_endian = result.to_bytes(4, byteorder='little')
​
    return ''.join(f'{b:02x}' for b in little_endian)
​
def get_x_sap_ri_44():
    # 随机44位字节
    random_arr = [random.choice('0123456789abcdef') for i in range(44)]
    # index14 = 1 index15=0 index16 = 1
    random_arr[14] = '1'
    random_arr[15] = '0'
    random_arr[16] = '1'
    return ''.join(random_arr)
def get_x_sap_ri():
    return get_x_sap_ri_8() + get_x_sap_ri_44()
# 测试
x_sap_ri = get_x_sap_ri()
print(x_sap_ri)  

```