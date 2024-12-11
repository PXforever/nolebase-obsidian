---
share: "true"
---
> 我们先需要下载该软件：https://downloads.clash.wiki/ClashPremium/

然后解压文件：
```shell
sudo apt install p7zip-full
7z x clash-linux-amd64-2023.08.17.gz 
```

我们先赋予执行权限，然后复制到系统的`bin`中，这样可以全局使用：
```shell
chmod +x clash-linux-amd64
sudo cp clash-linux-amd64 /usr/sbin/
```

接着又两种配置方式：
## 方法1(从Windows获取)
在`clash for windows`:
![[笔记/01 附件/Clash For Linux使用方法/file-20241024225918713.png|笔记/01 附件/Clash For Linux使用方法/file-20241024225918713.png]]
![[笔记/01 附件/Clash For Linux使用方法/file-20241025152719289.png|笔记/01 附件/Clash For Linux使用方法/file-20241025152719289.png]]
外层的`Country.mmdb`和`profiles`目录中有一个`xxxx.yml`
复制到`Linux`系统上，然后：
```shell
mkdir -p ~/.config/clash
cp ./xxxx.yml ~/.config/clash/config.yaml
cp Country.mmdb ~/.config/clash/
```

然后执行：
```shell
clash-linux-amd64
```
![[笔记/01 附件/Clash For Linux使用方法/file-20241024230140257.png|笔记/01 附件/Clash For Linux使用方法/file-20241024230140257.png]]

## 方法2（不推荐）
我们有订阅地址，需要下载配置文件：
```shell
wget -O config.yaml "https://xx.xxxx.xxx/api/v1/client/subscribexxxx"
```
下载到的文件一般是`base64`，我们需要将`base64`转换为明文，并且还需要做许多的操作。
具体可以参考：
```url
https://docs.gtk.pw/contents/parser.html#%E7%AE%80%E4%BE%BF%E6%96%B9%E6%B3%95-yaml
https://github.com/InoryS/Clash-Parser-Online?tab=readme-ov-file#%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87
https://github.com/SahanY099/clash_subscription_parser/tree/main
https://github.com/GongT/clash-subscription-processor/tree/master
https://wiki.metacubex.one/config/
```


# 配置
我们可以通过网站：`https://clash.razord.top/#/settings`进行配置：
![[笔记/01 附件/Clash For Linux使用方法/file-20241025152928224.png|笔记/01 附件/Clash For Linux使用方法/file-20241025152928224.png]]
![[笔记/01 附件/Clash For Linux使用方法/file-20241025153246997.png|笔记/01 附件/Clash For Linux使用方法/file-20241025153246997.png]]
![[笔记/01 附件/Clash For Linux使用方法/file-20241025153324295.png|笔记/01 附件/Clash For Linux使用方法/file-20241025153324295.png]]
注意外部控制设置，设置用与后台一致就行，一般不需要设置，错误的话可能导致无法打开该网页。
接着设置代理：
![[笔记/01 附件/Clash For Linux使用方法/file-20241025153620782.png|笔记/01 附件/Clash For Linux使用方法/file-20241025153620782.png]]
然后就OK了。
你也可以通过在`.bashrc`中添加：
```shell
export http_proxy='http://127.0.0.1:7890'
export https_proxy='http://127.0.0.1:7890'
```
# 其他
当然你可以可以选择`clash-verge`这中自带界面的。