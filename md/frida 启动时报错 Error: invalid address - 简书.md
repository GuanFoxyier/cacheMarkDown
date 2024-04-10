> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.jianshu.com](https://www.jianshu.com/p/34a4380bb42a)

`报错如下:`

```
{
    "type": "error",
    "description": "Error: invalid address",
    "stack": "Error: invalid address\n    at Object.value [as patchCode] (frida/runtime/core.js:203:1)\n    at ln (frida/node_modules/frida-java-bridge/lib/android.js:1209:1)\n    at pn.activate (frida/node_modules/frida-java-bridge/lib/android.js:1275:1)\n    at mn.replace (frida/node_modules/frida-java-bridge/lib/android.js:1323:1)\n    at Function.set [as implementation] (frida/node_modules/frida-java-bridge/lib/class-factory.js:1017:1)\n    at Function.set [as implementation] (frida/node_modules/frida-java-bridge/lib/class-factory.js:932:1)\n    at installLaunchTimeoutRemovalInstrumentation (/internal-agent.js:424:24)\n    at init (/internal-agent.js:51:3)\n    at c.perform (frida/node_modules/frida-java-bridge/lib/vm.js:12:1)\n    at _performPendingVmOps (frida/node_modules/frida-java-bridge/index.js:250:1)",
    "fileName": "frida/runtime/core.js",
    "lineNumber": 203,
    "columnNumber": 1
}


```

### 关闭 selinux:

```
adb shell 
su
setenforce 0


```