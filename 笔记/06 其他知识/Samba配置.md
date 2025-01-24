---
share: "true"
---
# 无用户登录使用
```shell
[shared] 
	path = /home/px
	browsable = yes 
	writable = yes 
	guest ok = yes 
	public = yes
	read only = no 
	forceuser = root 
	forcegroup = root
	create mask = 0755 
	directory mask = 0755
```
```shell
sudo systemctl restart smbd
```

# 指定用户
```shell
[px-share]
   path = /home/px
   valid users = px
   read only = no
   browsable = yes
   guest ok = no
   create mask = 0644           # 新创建的文件权限：rw-r--r--
   directory mask = 0755        # 新创建的目录权限：rwxr-xr-x
   public = no
```
然后将用户加入到`samba`中：
```shell
sudo smbpasswd -a px
```
如果无法连接，则尝试如下操作：
```shell
# 重启
sudo systemctl restart smbd

sudo systemctl status smbd
sudo systemctl status nmbd

# 启动
sudo systemctl start smbd
sudo systemctl start nmbd

# 开机自启动
sudo systemctl enable smbd
sudo systemctl enable nmbd

# 查看当前防火墙规则，并确保允许 Samba 流量：
sudo ufw allow samba

# 测试连接
smbclient -L //localhost -U px
```
我们可以在`windows`中添加一个网络位置，地址如下：
```shell
\\192.xxx.xxx.123\px-share
```