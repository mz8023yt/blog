---
title: '[ODM] LCM 相关 debug 实例'
date: 2018-06-05 13:08:56
tags:
  - lcm
  - qcom
categories: ODM
---

### 实例1：闪屏问题

#### 问题描述

工厂 OOB 测试先后测试出 3pcs 闪屏机器，不良现象为每隔 2s 左右规律性的出现一次黑屏，上海可生产测试出 1pcs 闪屏机器，不良现象和工厂不良一致

#### 分析过程

1. 分析工厂传回的 log  
   不良现象为每隔 2s 黑屏一次，2s 的频率和 esd check 的频率一致，并且 esd recover 动作掉电确实会黑屏一次，重点怀疑 esd check 这块  

       [  590.217496] lcd esd return_buf[1] = 0x88  status_value = 0x80
       [  593.385461] lcd esd check failed mdss_check_dsi_ctrl_status 175
       [  595.897451] lcd esd return_buf[1] = 0x88  status_value = 0x80
       [  597.856254] lcd esd check failed mdss_check_dsi_ctrl_status 175
       [  600.125425] lcd esd return_buf[1] = 0x88  status_value = 0x80
       [  602.998541] lcd esd check failed mdss_check_dsi_ctrl_status 175
       [  604.342345] lcd esd return_buf[1] = 0x88  status_value = 0x80

   搜索 `lcd esd` 关键字，确认到，确实是 lcd esd check fail，触发 recover 动作(黑屏一次)，正常情况下，recover 动作执行完后，ic 状态应该恢复正常，再次 check 将会 pass，但是，通过 log 中看出，recover 动作后，esd check 仍旧 fail，说明 ic 状态还是异常，再次触发 recover(黑屏一次)，如此循环，出现闪屏

2. 确认是哪一项 check failed  
   查看屏 dtsi 中 esd 相关的属性值，确认到是 09 寄存器第一个字节异常  

       qcom,esd-check-enabled;
       
       /* 指定 esd check 的 command，cmd 最后一次字节是 check 的寄存器 */
       qcom,mdss-dsi-panel-status-command = [
               14 01 00 01 05 00 01 68
               06 01 00 01 05 00 01 09
               14 01 00 01 05 00 01 CC
               14 01 00 01 05 00 01 AF
           ];
       qcom,mdss-dsi-panel-status-command-state = "dsi_lp_mode";
       qcom,mdss-dsi-panel-status-check-mode = "reg_read";
       
       /* 在 command 中指定指定过了 check 的寄存器，这里指定 check 对应寄存器的前几个字节 */
       qcom,mdss-dsi-panel-status-read-length = <1 3 1 1>;
       qcom,mdss-dsi-panel-max-error-count = <2>;
       
       /* 指定 esd check 读取的寄存器的标准值 buf[0]  buf[1] buf[2] buf[3] buf[4]  buf[5] */
       qcom,mdss-dsi-panel-status-value =  <0xC0>, <0x80  0x73   0x04>, <0x0B>, <0xFD>;

   怎么确认的：  
   `qcom,mdss-dsi-panel-status-command` 指定了要 check 哪些寄存器  
   `qcom,mdss-dsi-panel-status-read-length` 指定了要 check 这些寄存器的前几个字节，并将这些字节，依次编号为 0,1,2,3...  
   因此确认到 buf[1] 对应的是 09 寄存器的第一个字节，正常状态下 09h_01 = 0x80，但是异常情况下读取出来的是 0x88, bit3 从 0 变成了 1

3. 查看 spec 确认 09h_01[3] 表示什么含义  

   ![09h_01](https://raw.githubusercontent.com/mz8023yt/blog.material/master/odm/png/odm-lcm-debug-instance-01.png)

   确认到是 VGL 电压异常，看 spec 描述应该是 VGL 电压高于 esd 侦测的标准，导致 09h_01[3] 被置位，接下来需要实际测量下是否真的是这样的情况

4. 交叉主板和三合一，确认到闪屏不良现象跟着三合一模组走  

5. 实测良品和不良显示屏单体 VGL 电压  
   这里贴出模组厂验证情况： 

   ![模组厂验证结果](https://raw.githubusercontent.com/mz8023yt/blog.material/master/odm/png/odm-lcm-debug-instance-02.png)

   确认到不良单体的 VGL 电压较正常品偏低，和步骤3猜想一致  
   
#### 分析结论

此闪屏问题为 LCD 三合一模组单体不良，不良单体 VGL 电压较正常单体偏低，导致触发 esd check fail，并且 recover 无法修正，因此出现闪屏现象。

### 实例2：闪屏问题衍生问题

#### 问题描述

衍生出来问题情况很复杂，一条一条描述：

1. 工厂出现不良的机器，发现在 OOB 测试用的 user b08 版本 100% 闪屏，但是在组装用的 eng b08 版本 100% 不闪屏，客户需要我们理清 eng 和 user 版本的区别点
2. 上面模组厂测试的结果发现，在不同的界面下(桌面、白图、flick图片) VGL 的电压不一致，客户想要理清不一致的原因
3. 同步和玻璃厂 FAE 确认到玻璃要求的 VGL 电压范围为 `-9V ~ -11V`，但是从模组厂测试的结果发现，即使是良品也是不符合玻璃 spec 的，客户追问为什么良品也不满足，是否是软件上设置不对

#### 分析过程

**议题1：eng 和 user 版本的区别点**

1. 将代码回退到 eng B08(Factory分支) 和 user b08(MP分支) 时间节点，对比 lcd 驱动代码和 dtsi 配置文件，驱动和配置完全一模一样。同时使用上海已经拆解的不良品，分别刷 user b08 和 eng b8 版本，使用高精度示波器测量主界面桌面状态下，得到 VGL 电压分别为 8.075V(eng) 和 8.045V(user)，user 和 eng 版本测试结果电压基本一致

2. 考虑到模组厂测试的结果，在不同的界面下 VGL 的电压不一致，怀疑是测试场景的区别导致 eng 和 user 版本的 VGL 电压不一致，因此安排工厂使用工厂不良机在 flicker 图片验证 eng 版本是否闪屏，但是工厂不良机已经打回组装车间维修掉了，不良机丢失

3. 由于不良机丢失，无法验证工厂不良机是否是测试场景的区别导致 eng 和 user 版本的 VGL 电压不一致，但是从验证步骤1的结论看来，user 版本和 eng 版本 VGL 电压基本一致，客户不再追究 user 和 eng 版本的差异了。但是由于我司工厂生产流程是: `贴片 -> 组装 -> 组装测试(eng) -> 前10k的订单会额外进行OOB测试(user) -> 包装 -> 出货`，对于后续出货的机器，并不会再刷 user 版本进行 OOB 测试，目前看到有 1pcs 闪屏机器是在 eng 不闪，user 版本才闪，客户需要提供一种卡关策略，要保证所有的 VGL 偏低的单体不良的机器可以在 eng 版本下 100% 被卡下来，保证不会流到用户手中

4. 根据上面模组厂测试的结果可以看出在 flicker 图片界面，VGL 被拉到最低，最容易触发 esd check fail。因此卡关策略为将 flicker 图片导入工模 lcd 测试项中，要求产线在测试 lcd 测试项是静置手机在 flicker 图片 10s 左右，观是否有闪屏现象。若闪屏，则当做不良品打出，若不闪，则可以保证用户在其他场景下使用也不会闪屏。此方案客户已经认可，并导入了最新版本中

**议题2：为什么不同场景 VGL 电压不一致**

此议题涉及 LCM 内部组成原理，由部件主导回复，软件无需跟进

**议题3：为什么良品三合一的 VGL 电压也不符合玻璃 spec**

1. 确认软件设置的 VGL 值是多少  
   屏 dtsi 配置 C1 寄存器值如下：

       qcom,mdss-dsi-on-command = [
           ... ...
           /* command format len reg 1  2  3  4  5  6  7  8  9  10 11 12 */
           39 01 00 00 00 00 0d  C1  54 00 1E 1E 77 C1 FF FF EE EE EE EE
           ... ...
       ]

   spec 描述如下：

   ![C1寄存器](https://raw.githubusercontent.com/mz8023yt/blog.material/master/odm/png/odm-lcm-debug-instance-03.png)

   确认到软件上设置 C1 寄存器的值为 0x54，对照 spec 确认到 VGL 值是 -11V，符合玻璃 spec 标准。但实际输出仅有 8.5V 左右，约 FAE 一起分析，确认到是 `C1h_06[5]:TURBO` 没有使能 TURBO

2. 软件上为什么没有使能 TURBO  
   项目初期是使能了 TURBO 的，但是后面为了解决射频干扰问题，将其关闭了

3. 解射频干扰时没有考虑玻璃 VGL 电压要求吗?  
   玻璃 spec Gate 电压要求：

   ![spec](https://raw.githubusercontent.com/mz8023yt/blog.material/master/odm/png/odm-lcm-debug-instance-05.png)

   玻璃 spec 上仅仅说明了 typ 值是 -10V，并没有指明 min、max 值，一般没有写明 min 值则认为玻璃没有 min 值的要求，因此当时才将 VGL 适当调低以优化射频干扰问题，后面提到的 `-9V ~ -11V` 是出现闪屏问题后玻璃厂 FAE 提供的电压范围

4. 目前所有三合一模组 100% 不符合玻璃 spec 要怎么处理  
   处理方式：重新开启 TURBO 功能  
   此处理方案需要重新调试射频干扰，并且 lcd 主客观测试需要重测，测试好后还要将新的参数重新烧录到模组 OTP 中，和模组厂确认了目前已经生产了 150k 左右的模组，意味着这 150k 的模组，需要重新返工，并且对于已经装机的机器，要拆机成三合一状态，重烧 OTP，是个大动作，此方案客户和项目都不同意

5. 实测的 VGL 电压偏低是否会出现大批量闪屏的风险  
   唯一的处理方案客户不同意，那只能将就着使用目前 VGL 设定了，但目前实际测试得到的 VGL 电压和 esd 侦测 VGL 的标准(-8.5V)很接近，可能会出现大批量闪屏的风险，因此考虑将 esd 侦测标准从 -8.5V 修改为 -8.0V  
   屏 dtsi 配置 C6 寄存器值如下：

       qcom,mdss-dsi-on-command = [
           ... ...
           /* command format len reg 1  2  3  4  5 */
           39 01 00 00 00 00 06  C6  26 40 CF 7F 20
           ... ...
       ]

    spec 描述如下：  

    ![C6寄存器](https://raw.githubusercontent.com/mz8023yt/blog.material/master/odm/png/odm-lcm-debug-instance-04.png)
    
    因此将 C6_01 从 26 修改为 16 即可配置 esd check 标准为 8.0V


   
