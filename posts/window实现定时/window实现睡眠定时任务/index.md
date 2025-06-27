# 🖥Window实现睡眠定时任务

## 1 定时任务设置
控制面版 -> 系统和安全 -> 计划任务 -> 创建任务

![](./Pasted-image-20250627145938.png)

![](assets/Pasted%20image%2020250627150054.png)

![](assets/Pasted%20image%2020250627150340.png)

![](assets/Pasted%20image%2020250627150422.png)

## 2 电源模式设置
### 2.1 电源模式选择
设置 -> 系统 -> 电源 -> 电源模式 -> 平衡（也可以选其他的，选择哪个后面就修改哪个）
### 2.2 电源模式设置
控制面版 -> 系统和安全 -> 电源选项 -> 更改计划设置 -> 更改高级电源设置 

![](assets/Pasted%20image%2020250627151040.png)

## 3 睡眠模式设置
### 3.1 查询睡眠模式 cmd执行
```cmd
powercfg /a
```
正常：
![](assets/Pasted%20image%2020250627151558.png)


### 3.2 若结果为 待机(S0)  修改为(S3) cmd(管理员模式)执行
```
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Power" /v PlatformAoAcOverride /t REG_DWORD /d 0 /f
```
修改完需要重启，重复 2.3.1 ， 结果为 待机(S3) 则正常

## 4 执行
ctrl + x -> 注销或重启 -> 睡眠 -> 等待设置的定时时间执行程序
![](assets/Pasted%20image%2020250627151815.png)


## 引用

**版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。**


---

> 作者: [luqiCraft](https://luqi678.github.io/luqicraft)  
> URL: http://localhost:1313/posts/window%E5%AE%9E%E7%8E%B0%E5%AE%9A%E6%97%B6/window%E5%AE%9E%E7%8E%B0%E7%9D%A1%E7%9C%A0%E5%AE%9A%E6%97%B6%E4%BB%BB%E5%8A%A1/  

