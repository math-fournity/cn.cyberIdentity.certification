# 《国家网络身份认证》App 密码学使用与安全工程问题调查报告

**版本**：公开版 v1.0（自包含——不依赖任何内部资料，仅依据 APK 本体与公开工具）
**日期**：2026-10-24
**性质**：独立安全研究；静态分析，未运行该 App、未接触其任何服务器
**适用读者**：计算机专业背景，非密码学方向

---

## 摘要

对《国家网络身份认证》Android 客户端（v1.2.40）的静态分析发现四项问题，按严重度排序：

| # | 主张 | 证据强度 | 责任主体 |
|---|---|---|---|
| **P1** | 其人脸活体 SDK（云从科技出品）生产库 `.data` 段内置 **5 个密钥存储位、4 个不同 SM4 密钥值**（编译期固定，其中一把自述"测试钥"），全部经 GOT 重定位被代码引用（合计 16 处指令点） | 指令级（含二进制地址与普查表，§3.1） | 云从科技（科创板 688327） |
| **P2** | 同库编入 SM4 与 AES 的 **ECB 工作模式**原语且存在实际调用 | 指令级（§3.2） | 云从科技 |
| **P3** | App 业务层存在**密钥种子可打印的调试日志面**（发布剥离缺失） | 静态字符串证据（§3.3） | App 开发/集成方体系 |
| **P4** | App 加固壳仅覆盖业务代码的 33%，**全部安全配置位于未保护区** | 结构测量（§3.4） | App 开发/集成方（采购验收责任） |

**明确不作为问题主张的事项**（§6）：活体动作码等协议常量在客户端（业务必需）、
人脸关键点检测模型在端侧（行业标准架构）、后端 IP/域名可见性（信息暴露面）等。

## 1. 调查对象

| 项 | 值（直读自 APK） |
|---|---|
| 官方名称 | 国家网络身份认证（application-label 字段） |
| 包名 / 版本 | `cn.cyberIdentity.certification` / versionName **1.2.40**（versionCode 92） |
| 签名者 | DN：**CN=公安部第一研究所, OU=公安部第一研究所, O=公安部第一研究所, ST=BeiJing, C=CN** |
| 证书指纹 | SHA-256 `f9f565086cb680b5417445f493e588349e8527418ece5ba9e91eb5a75c9ebb73`（SHA-1 `76db302749c810ef6c1788295259a43da8ecaac6`） |
| APK 指纹 | SHA-256 `1b6368cf320d68b7aa6eab67c458cf1f5849ed9da079f1741d8a41f6553dfedf` |
| 监管背景 | 《国家网络身份认证公共服务管理办法》（公安部等六部门，2025-07-15 施行） |

该 App 为"网号/网证"官方申领核验客户端，法定定位是"最小化收集、最小化提供"个人
身份信息——本报告以其自身职能为评判基准。

## 2. 方法与工具（全部公开可复现）

APK 解包（unzip）→ 清单与签名（Android build-tools 的 `aapt`/`apksigner`）→
native 库分析（LLVM 工具链 `llvm-nm`/`llvm-objdump` + 任意十六进制编辑器）→
加固壳格式还原（自写解析脚本，方法已发表于此报告 §7 的复现要点）。
**未使用**该 App 的任何网络服务；未在设备上运行该 App。

## 3. 事实与证据（每项附二进制地址）

### P1 硬编码明文密钥

**位置**：`lib/arm64-v8a/libcloudwalk_sdk.so`（4.6MB，云从科技活体/OCR SDK）。

`.data` 段密钥普查（符号表 × 字节直读 × GOT 重定位 × 指令引用点，四重取证；
文件偏移，任一十六进制编辑器可直接核对；armeabi-v7a 版同值在场）：

| 符号 | 偏移(file) | 值（16 字节） | 形态 | 指令引用点（adrp+ldr→GOT） |
|---|---|---|---|---|
| `aeSKey` | 0x491008 | `CW_128EncryptKey` | ASCII | 0x7d754, 0x7d7b4 |
| `sm4Key` | 0x491019 | `cw_test_sm4_key_` | ASCII（自述测试钥） | 0x7da6c, 0x7da8c |
| `key_boxal` | 0x49102a | `b5911fc279686d931203630d99f16e41` | 二进制 | 0x7da00, 0x7da10 |
| `key_bodgl` | 0x49103a | `cw121sm095ewgfhn` | ASCII | **0x7deac**(sm4Enc 默认钥), 0x7da38, 0x7da60, 0x7df20 |
| `key_bodgl1` | 0x491064 | `cw121sm095ewgfhn`（副本） | ASCII | 无 GOT 引用 |
| `sm4Random16BitKey` | .bss | 运行期随机生成 | 缓冲区 | 6 处（含 0x86438 等） |

**计数结论**：5 个密钥存储位，其中 **4 个编译期固定的 16 字节密钥值**（3 个 ASCII、
1 个二进制；`key_bodgl1` 为 `key_bodgl` 同值副本），另有 1 个运行期随机钥缓冲。
除副本外，**每个固定钥都有代码路径实际取用**（引用点落在密钥选择函数簇
0x7d754-0x7da8c 与加密入口 0x7deac）。

**取钥指令链**（反汇编地址 = 文件偏移）：

```
0x86bc4  Java_cloudwalk_live_api_CwNativeLive_cwSm4Enc   (JNI 导出函数, 符号表 T)
0x86bcc    b  0x74170        ← 尾调用进入实现 sm4Enc
0x7dea8    cbz x24, +0x4     ← 分支：调用方未传钥时走默认钥路径
0x7deac    adrp x1, 0x4a0000
0x7deb0    ldr  x1, [x1, #0xd90]   ← 从 GOT 槽 0x4a0d90 取钥指针
0x7deb8    bl   sm4_setkey_enc     ← 用该钥初始化 SM4
（.rela.dyn 中 offset=0x4a0d90 的 GLOB_DAT 重定向 → 符号 key_bodgl）
```

**主张的精确边界**：默认钥路径在指令级存在且可达 + 测试钥随生产包发布。
不主张"每次加密都使用该钥"（运行期分支占比需动态观测）。

### P2 ECB 工作模式

- `sm4_crypt_ecb` @ **0x7e40c**（导出符号），实际调用点 **0x7d2c8** 与 **0x7dc38**（`bl` 指令）
- `AES_ECB_PKCS7_Encrypt/Decrypt` @ **0x80c98 / 0x80e10**（导出符号）
- SM4 S-box @ .rodata **0x3a0d28**（`d6 90 e9 fe cc e1 3d b7…`，国密 SM4 标准查表）

### P3 调试日志面

App 业务数据中的静态字符串（解包后可直接检索）：`pKseed (SHA-1)为:`（密钥种子
打印语句）、`MAC验证成功！！！`、`authToken已失效`/`authToken验签未通过`。
含中文调试串共 456 条——表明构建管线缺少发布剥离环节。

### P4 壳覆盖

业务 dex 载荷 17.5MB，其中加密岛 5.7MB（33%，尾部熵 7.99 全区间剖面）；其余
67%（含上述全部加密配置、类名、方法名、后端地址）以明文或可还原格式存放。

### P5（补充）App 代码与 SDK 的集成（回应"是否死代码"）

- 云从 Java 层 1,928 个方法注册于 App 的方法表（编译进 App）；
- App 明文代码区 **697 条类型操作指令**（new-instance/check-cast 等）直接引用云从
  类，地址样例：**0x28487e**（`BankOcrResultCallback`）、**0x2848cc**
  （`CardCaptureCallback`）、**0x28484c**（`FaceStageCallback`）、**0x28186e**
  （`CwLiveConfig`）——银行卡 OCR/证件抓拍/人脸活体流程在指令级使用该 SDK。
- 边界：指向 `cwSm4Enc` 的具体 Java 调用语句位于 33% 加密区内，静态未证。

## 4. 规范依据（原文引用，链接可核验）

1. **NIST SP 800-38A** §6.1（官方 PDF）："In the ECB mode, under a given key, any
   given plaintext block always gets encrypted to the same ciphertext block. **If this
   property is undesirable in a particular application, the ECB mode should not be used.**"
   https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-38a.pdf
2. **GB/T 39786-2021**《信息系统密码应用基本要求》（密评基线）——国家密码管理局
   发布说明：https://www.oscca.gov.cn/sca/xxgk/2021-03/19/content_1060852.shtml
   （配套：GB/T 17964-2021《分组密码算法的工作模式》；密评实践中多分组数据使用
   ECB 属不符合项）
3. **OWASP MASVS**：CRYPTO-1（"The app employs current strong cryptography and uses
   it according to industry best practices."）、CRYPTO-2（密钥管理）。
   https://mas.owasp.org/MASVS/
4. **国密社区库**（gmsm 文档）："基于安全考虑，最好不要使用该模式。"
   https://github.com/emmansun/gmsm/blob/develop/docs/sm4.md
5. **教科书口径**：https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation
   （ECB "lack of diffusion… fails to hide data patterns"）
6. **P1 的依据（密钥管理的规范要求，原文引用）**：
   - **NIST SP 800-57 Part 1 Rev.5**《Key Management Recommendation》：
     > "Secret and private keys need to be protected against **unauthorized
     > disclosure**, and all keys need to be protected against modification."
     > （对称密钥须防止未授权披露。）同文件并以规范语气要求："the keying material
     > **shall** be protected…one or more of the appropriate protection mechanisms
     > for stored information in Section 6.2.2 **shall** be used."
     https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r5.pdf
   - **OWASP MASVS-CRYPTO-2**（原文）：
     > "The app performs key management according to industry best practices.
     > **Even the strongest cryptography would be compromised by poor key
     > management.** This control covers the management of cryptographic keys
     > throughout their lifecycle, including key generation, **storage and
     > protection**."
     https://mas.owasp.org/controls/MASVS-CRYPTO-2/
   - **GB/T 39786-2021**《信息系统密码应用基本要求》：密评基线对"密钥管理"
     （生成、存储、分发、更换、销毁全生命周期）提出分级要求——静态评审中，
     编译期固定对称密钥随公开分发包发布属密钥存储不符合项（标准正文以官方文本为准，
     发布说明见 https://www.oscca.gov.cn/sca/xxgk/2021-03/19/content_1060852.shtml）。

## 4b. 对预期反驳的回应："通信加密用的默认钥写在包里，不是也行吗？"

**不行的理由不是偏好，是密码学的定义性质**：APK 是任何人可获取的公开分发物，
写入其中的对称密钥对全世界公开。对称加密的保密性**完全依赖密钥的私密性**（上引
NIST 原文即该原则的规范表述）——密钥公开后，"加密"对任何第三方退化为"编码"：
任何人可解密历史流量/数据、可伪造合法密文。这不是"做得不够好"，是**该密钥参与的
保密性主张不成立**。

**正确的做法（且本 App 环境内三条路都是现成的）**：
1. **会话密钥运行期随机生成**——该 SDK 自己就实现了这条路：`cwGetRandom16Sm4key`
   （导出函数 @0x83ac0）与 `sm4Random16BitKey` 缓冲（6 处引用）证明厂商完全掌握
   正确模式，固定钥族属于同一代码库中的反面并存；
2. **密钥存入 Android Keystore**（设备绑定、不可导出）——Android 官方文档原文：
   > "The Android Keystore system lets you store cryptographic keys in a container
   > to make them **more difficult to extract from the device**. Once keys are in
   > the keystore… the key material remaining **non-exportable**."
   > （https://developer.android.com/privacy-and-security/keystore）
   本 App minSdk 24，Keystore 自 API 18+ 可用，无任何兼容性障碍；
3. **服务端密钥分发/服务端密码运算**——App 内已集成卫士通（电科网安）密码服务
   平台 SDK（`cn.com.westone.wcspsdk`），即业界标准的"密钥不下发"架构，说明
   供应链上正确的架构选项触手可及。

## 5. 责任归属

| 主体 | 公司 | 代码 | 承担 |
|---|---|---|---|
| 云从科技集团股份有限公司（官网 cloudwalk.cn） | — | **科创板 688327** | P1+P2（逐条位于 libcloudwalk_sdk.so） |
| 公安部第一研究所（App 签名方）及其全资子公司北京中盾安信科技发展有限公司（anicert.cn，CTID 平台唯一运营方） | — | 未上市 | P3 业务侧、P4（集成/验收） |
| 中电科网络安全科技股份有限公司（原卫士通） | — | 深交所 002268 | 组件在场，无缺陷主张 |
| 每日互动（300766）/百度/友盟/华为 | — | — | 组件在场，无缺陷主张 |

消歧义：巴西金融科技公司 CloudWalk（cloudwalk.io）与美国 Westone Laboratories
与本案无关。云从科技自述参与 91 项标准制定（含人脸识别国标）——此背景与 P1 并陈。
待核：中盾安信法定代表人登记信息存在两种公开记载（杨林/符东昇），以国家企业
信用信息公示系统为准。

## 6. 明确不作为问题主张的事项（审慎声明）

为避免夸大，以下事项**不构成**本报告的指控：①活体动作码与结果枚举（LiveEye/
FACE_ATTACK 等）在客户端——业务协议必需；②人脸关键点检测模型/代码在端侧——
端侧活体的行业标准架构；③后端 IP 与域名可见——信息暴露面，行业普遍；④模板包名
残留（com.example.*）——工程卫生问题；⑤上述 SDK 符号可见性——JNI 导出机制所必需。
真正构成问题的判定，须落在"密钥管理/工作模式/信任模型"等可检验项上。

## 7. 复现指南（公开工具，任何人可独立验证）

```bash
# 0) 校验对象
shasum -a 256 nia.apk        # 1b6368cf320d68b7aa6eab67c458cf1f5849ed9da079f1741d8a41f6553dfedf
# 1) 版本与签名
aapt dump badging nia.apk | head -2
apksigner verify --print-certs nia.apk
# 2) 解包
unzip nia.apk -d apk_x
# 3) 密钥直读（P1）——十六进制编辑器跳到文件偏移：
#    0x49103a → "cw121sm095ewgfhn" ; 0x491019 → "cw_test_sm4_key_"
xxd -s 0x491019 -l 48 apk_x/lib/arm64-v8a/libcloudwalk_sdk.so
# 4) 指令链（P1/P2）
llvm-nm -D apk_x/lib/arm64-v8a/libcloudwalk_sdk.so | grep -E "cwSm4Enc|sm4_crypt_ecb|key_bodgl|sm4Key"
llvm-objdump -d --start-address=0x7dea8 --stop-address=0x7debc apk_x/lib/arm64-v8a/libcloudwalk_sdk.so
llvm-objdump -d --start-address=0x7d2c0 --stop-address=0x7d2cc apk_x/lib/arm64-v8a/libcloudwalk_sdk.so
# 5) 调试日志面（P3）
grep -a "pKseed" apk_x/classes.dex | head
```

## 8. 证据边界（未决事项）

1. 运行期业务流对 `cwSm4Enc` 的具体调用与默认钥分支占比（需动态观测或解密 33% 加密区）；
2. 活体判定结论是否端侧自证（信任模型问题——如证实则升格为重要发现）；
3. `pKseed` 日志所属组件的精确定位；
4. 中盾安信法人登记信息核验。

## 9. 方法论声明

本调查过程中，两项初步结果（一个字符串索引映射、一张方法调用图）经自证检验
（类名一致性验证、噪声基线对照）证伪后**主动撤回，未进入任何主张**。本报告全部
结论均可由 §7 步骤独立复算。

---

*本报告仅用于安全改进目的。责任归属依据静态证据与公开工商信息，最终认定权在
相关监管与司法机构。*
