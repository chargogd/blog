在安装monkeydev工具时遇到了些因为网络代理导致的坑爹问题，这里记录下

执行命令出现以下问题

```
sudo /bin/sh -c "$(curl -fsSL https://raw.githubusercontent.com/AloneMonkey/MonkeyDev/master/bin/md-install)"
```

下载进度一直为0

```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:01:06 --:--:--     0
```

过一段时间后超时

```
rl: (7) Failed to connect to codeload.github.com port 443: Operation timed out
Failed to download https://codeload.github.com/AloneMonkey/MonkeyDev/tar.gz/master to /var/folders/zz/zyxvpxvq6csfxvn_n0000000000000/T/sh.NXyeShpv/file.tar.gz
```

### 解决过程

1、尝试配置和删掉代理都无法下载

2、github本身能访问和clone monkeydev master代码

3、该master文件也能通过chrome下载

4、查看md-install基本，找到curl相关代码

```
"$curlPath" --output "$targetPath" "$sourceUrl" || \
		panic $? "Failed to download $sourceUrl to $targetPath"
```

替换原值后是：

```
/usr/bin/curl --output /var/folders/zz/zyxvpxvq6csfxvn_n0000000000000/T/md-install.YutiqsU4/file1.tar.gz https://codeload.github.com/AloneMonkey/MonkeyDev/tar.gz/master
```

单独运行还没到下载就报错，分析是因为当前 用户没有var/folders/zz/zyxvpxvq6csfxvn_n0000000000000/T/目录权限

```
Warning: ile1.tar.gz: Permission denied
```

使用加上sudo来执行，则下载进度一直是0，最后还是同样time out错误

5、所以问题来了，curl在sudo情况下无法下载，单独测试curl 可以下载

6、经过网上的资料查询和重试才得知是因为公司网络配置了代理，虽然当前用户shell 环境http_proxy配置了代理，但在sudo运行时没有复用当前用例的环境变量导致外网github.com下载超时

7、sudo加上-E参数后执行成功

```
sudo -E /bin/sh -c "$(curl -fsSL https://raw.githubusercontent.com/AloneMonkey/MonkeyDev/master/bin/md-install)"
```

8、**遗留疑点**：

sudo查看环境变量也是可以正常获取的，为啥执行curl就无法下载呢？

```
~sudo echo $http_proxy
http:xxproxy.xx.com:8080
```

参考：[Why sudo curl ignores proxy settings?](https://superuser.com/questions/493906/why-sudo-curl-ignores-proxy-settings)