> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/c-x-a/p/17565445.html)

1. 首先下载 httpcanary 的专业版  
链接: [https://pan.baidu.com/s/1xI5NNktBoFiTVLZFNWjSPQ](https://pan.baidu.com/s/1xI5NNktBoFiTVLZFNWjSPQ) 提取码: 67pd  
2. 将下面的 sh 文件，放到手机的 /data/local/tmp 目录, 命令为 cert.sh  
并给权限 chmod 777 cert.sh

```
# cert.sh
set -e # Fail on error
# Create a separate temp directory, to hold the current certificates
# Without this, when we add the mount we can't read the current certs anymore.
mkdir -m 700 /data/local/tmp/cert
# Copy out the existing certificates
cp /system/etc/security/cacerts/* /data/local/tmp/cert/
# Create the in-memory mount on top of the system certs folder
mount -t tmpfs tmpfs /system/etc/security/cacerts
# Copy the existing certs back into the tmpfs mount, so we keep trusting them
mv /data/local/tmp/cert/* /system/etc/security/cacerts/
# Copy our new cert in, so we trust that too
cp /data/local/tmp/87bc3517.0 /system/etc/security/cacerts/
# Update the perms & selinux context labels, so everything is as readable as before
chown root:root /system/etc/security/cacerts/*
chmod 644 /system/etc/security/cacerts/*
chcon u:object_r:system_file:s0 /system/etc/security/cacerts/*
# Delete the temp cert directory & this script itself
rm -r /data/local/tmp/cert
# rm ${injectionScriptPath}
echo "System cert successfully injected"


```

需要注意的是重启手机之后证书就失效了，每次抓包前执行一下就可以了  
3. 设置 httpcanary 的部分

```
 mv HttpCanary.pem HttpCanary.jks
 mv HttpCanary.jks /data/data/com.guoshi.httpcanary.premium/cache/


```

其中 HttpCanary.pem 和 87bc3517.0 我我已经导出到网盘  
链接: [https://pan.baidu.com/s/1MhDHmxDoRFDFFbjkj5XMRQ](https://pan.baidu.com/s/1MhDHmxDoRFDFFbjkj5XMRQ) 提取码: 7we0  
![](https://img2023.cnblogs.com/blog/736399/202307/736399-20230719141331242-1817290559.png)  
4. 测试发现到这里还不行，还要进行最后一步。  
打开 httpcanary 的设置 ->httpcanary 根证书 -> 添加根证书至系统  
到这里才算完成。  
![](https://img2023.cnblogs.com/blog/736399/202309/736399-20230906173809523-1616859562.png)