


---


　　大家好，我是痞子衡，是正经搞技术的痞子。今天痞子衡给大家介绍的是**利用i.MXRT10xx系列内部DCP引擎计算CRC32值时需注意数据长度对齐**。


　　MCU 开发里常常需要 CRC 校验来检查数据完整性，CRC 校验既可以纯软件实现也可以借助 MCU 片内外设硬件实现。大部分 MCU 里通常都会包含一个单独的硬件 CRC 外设，但是在 i.MXRT 四位数系列里，翻看参考手册，我们却找不到名为 CRC 的外设，难道这么一款高性能 MCU 不支持硬件 CRC？当然不是！这个功能藏在一个更强大的数学计算引擎外设里。



> * Note：在 i.MXRT10xx 系列上这个引擎是 DCP，在 i.MXRT11xx 系列上这个引擎升级为 CAAM。


　　关于 DCP 引擎使用，痞子衡写过一篇文章 [《DCP计算Hash值时需特别处理L1 D\-Cache》](https://github.com)。最近官方社区里有人提问，当待校验 CRC 数据长度是 4 字节整数倍时，DCP 计算结果和一些[在线网站](https://github.com)上的计算结果保持一致（多项式和配置设置一致），但是当数据长度不是 4 字节对齐时，两者结果就不一致了，这是怎么回事？今天咱们来聊一聊：


### 一、DCP对于CRC支持


　　翻看任何一个 i.MXRT10xx 系列的参考手册，仅在 **System Security** 章节有一小段关于 DCP 特性描述，里面讲了能支持 CRC32，但是关于多项式配置信息没有提及。


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/cnblogs/i.MXRT10xx_DCP_CRC_feature.PNG)


　　既然手册没涉及太多，那直接撸代码吧，可以参考 SDK\\boards\\evkmimxrt10xx\\driver\_examples\\dcp 例程，相关代码足够简单抄录如下。代码里仅 m\_handle.swapConfig 设置会改变 CRC 计算结果（因为对源数据做了 swap 处理），除此以外并未提供其他 CRC 多项式参数配置，因此可以基本认定 DCP 支持的是一个固定参数模式的 CRC32 算法分支，用户无法更改参数。



```
#include "fsl_dcp.h"

dcp_config_t dcpConfig;
DCP_GetDefaultConfig(&dcpConfig);
DCP_Init(DCP, &dcpConfig);

dcp_handle_t m_handle;
m_handle.channel    = kDCP_Channel0;
m_handle.keySlot    = kDCP_KeySlot0;
// 仅这里换成 kDCP_InputByteSwap 会影响 CRC 计算结果（res - 4字节）
m_handle.swapConfig = kDCP_NoSwap;
status = DCP_HASH(DCP, &m_handle, kDCP_Crc32, srcBuf, srcLen, res, &resLen);

```

　　这里痞子衡就不继续卖关子了，DCP 固定支持的就是经典的 CRC32\-MPEG2，其参数如下：


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/cnblogs/Kinetis_Boot_ImageCRC_mpeg2_characteristics.PNG)


### 二、DCP\-CRC32关于数据对齐处理


　　关于 CRC32\-MPEG2 算法实现细节可在 IEEE 802 标准手册里找到，原则上其对于源数据长度是没有对齐要求的，但在具体硬件实现上，不同硬件可能会增加末尾数据对齐处理。


　　痞子衡找了一块 RT1020\-EVK 开发板跑了一下 SDK\\boards\\evkmimxrt1020\\driver\_examples\\dcp 例程，只对例程做了简单修改如下，从测试结果来看，发现 DCP 对源数据做了末尾 4 字节对齐处理（用 0x00 填充）。


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/cnblogs/i.MXRT10xx_DCP_CRC_test.png)


　　DCP 这个数据对齐处理特性说明能在哪里找到呢？我们知道 DCP 是跟芯片安全特性相关的，i.MX RT 芯片除了有普通参考手册（RM）之外，还有一个安全参考手册（SRM），在 SRM 里面会进一步介绍芯片安全相关的外设。在[恩智浦官网 i.MXRT 产品主页](https://github.com)进入具体型号，切换到 Secure Files 选项（这里需要账号登录，并且申请访问权限，可联系 FAE 签 NDA），如果有访问权限，便可以下载 SRM：


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/cnblogs/i.MXRT10xx_DCP_CRC_SecureFile.PNG)


　　在 SRM 里我们看到了 DCP CRC32 对于数据对齐处理策略，与板级测试结果吻合：


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/cnblogs/i.MXRT10xx_DCP_CRC_SRM_note.PNG)


　　文章开头说 i.MXRT11xx 系列里的 CAAM 模块是 i.MXRT10xx 里 DCP 的升级，我们仅从 CRC 支持方面可见一斑，CAAM 对于 CRC 的支持更丰富，除了经典算法，还支持用户自定义参数（这就比较像传统 MCU 里的单独 CRC 外设）：


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/cnblogs/i.MXRT10xx_DCP_CRC_CAAM_fea.PNG)


　　至此，利用i.MXRT10xx系列内部DCP引擎计算CRC32值时需注意数据长度对齐痞子衡便介绍完毕了，掌声在哪里\~\~\~


### 欢迎订阅


文章会同时发布到我的 [博客园主页](https://github.com):[slower加速器](https://jisuanqi.org)、[CSDN主页](https://github.com)、[知乎主页](https://github.com)、[微信公众号](https://github.com) 平台上。


微信搜索"**痞子衡嵌入式**"或者扫描下面二维码，就可以在手机上第一时间看了哦。


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/github/pzhMcu_qrcode_258x258.jpg)


