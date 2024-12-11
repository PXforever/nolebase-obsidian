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
