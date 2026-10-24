# cn.cyberIdentity.certification —《国家网络身份认证》App 安全调查

独立静态安全调查 · 报告自包含 · 全部主张可独立复算

天道好轮回，壮哥放过谁？

 👉 https://0x10010.com/?Sepolia=0xbe6dd4ba0fcd16ffe06f68014fd175cfa96d231aab055ecd2e5bd596536461a0
 
---

## 调查对象身份（核验指纹）

任何引用本 repo 结论前，请先核对以下指纹与本报告一致：

| 项 | 值 |
|---|---|
| 官方名称 | 国家网络身份认证（application-label 直读） |
| 包名 / 版本 | `cn.cyberIdentity.certification` / versionName **1.2.40**（versionCode 92） |
| 签名者 DN | `CN=公安部第一研究所, OU=公安部第一研究所, O=公安部第一研究所, ST=BeiJing, C=CN` |
| 证书 SHA-256 | `f9f565086cb680b5417445f493e588349e8527418ece5ba9e91eb5a75c9ebb73` |
| APK SHA-256 | `1b6368cf320d68b7aa6eab67c458cf1f5849ed9da079f1741d8a41f6553dfedf` |
| 监管背景 | 《国家网络身份认证公共服务管理办法》（公安部等六部门，2025-07-15 施行） |

**消歧义**：巴西金融科技公司 CloudWalk（cloudwalk.io）与美国 Westone Laboratories
与本案无关；本 repo 中 "CloudWalk/云从" 仅指云从科技集团股份有限公司（688327）。

## 报告

| 目录 | 内容 |
|---|---|
| [`reports/2026-10-crypto-and-engineering/`](reports/2026-10-crypto-and-engineering/report.zh.md) | **2026-10：密码学使用与安全工程问题调查报告**（公开版 v1.1，中文）——四项主张、逐项二进制地址、规范原文引用、责任归属、审慎声明（明确不指控清单）、公开工具复现指南 |

后续调查报告（如加密区分析、动态观测）将以独立目录追加。

## 核心结论速览

| # | 主张 | 证据强度 |
|---|---|---|
| P1 | 活体 SDK（云从出品）生产库内置 5 个密钥存储位/4 个 SM4 密钥值（其一自述"测试钥"），全部被代码引用 | 指令级（取钥指令链 + 重定位 + `.data` 普查表） |
| P2 | 同库编入 SM4/AES 的 ECB 工作模式原语且存在实际调用 | 指令级（导出符号 + `bl` 调用点） |
| P3 | 业务层存在密钥种子可打印的调试日志面（发布剥离缺失） | 静态字符串证据 |
| P4 | 加固壳仅覆盖 33%，全部安全配置位于未保护区 | 结构测量 |

**明确不作为问题主张的事项**（防夸大）：活体动作码等协议常量在客户端、人脸关键点
检测模型在端侧（行业标准架构）、后端地址可见性等——完整清单见报告 §6。

## 复现

仅依赖公开工具（`unzip` / `xxd` / Android build-tools `aapt`+`apksigner` / LLVM
`llvm-nm`+`llvm-objdump`），逐步命令见
[报告 §7](reports/2026-10-crypto-and-engineering/report.zh.md)。例如密钥直读：

```bash
xxd -s 0x491019 -l 48 apk_x/lib/arm64-v8a/libcloudwalk_sdk.so
# 0x491019 → "cw_test_sm4_key_" ; 0x49103a → "cw121sm095ewgfhn"
```

## 引用

本 repo 提供 [`CITATION.cff`](CITATION.cff)（GitHub 会在仓库首页显示"引用本仓库"按钮，
引用信息以该文件为准）。BibTeX：

```bibtex
@misc{nia-audit-2026,
  author       = {Math Fournity and GLM-5.3},
  title        = {《国家网络身份认证》App（cn.cyberIdentity.certification v1.2.40）
                  密码学使用与安全工程问题调查报告},
  year         = {2026},
  month        = {10},
  howpublished = {\url{https://github.com/math-fournity/cn.cyberIdentity.certification}},
  note         = {静态分析；APK SHA-256
                  1b6368cf320d68b7aa6eab67c458cf1f5849ed9da079f1741d8a41f6553dfedf}
}
```

引用规范要求：注明出处（作者+URL+版本日期），并**同时注明所核验的 APK 指纹**——
本 repo 的结论仅对上述指纹的样本负责。

## 许可

本 repo 的**报告文本与文档**采用
[Creative Commons Attribution 4.0 International (CC BY 4.0)](LICENSE) 许可：
允许任何媒介复制、再发行、修改与商业使用，条件是署名（见 LICENSE 全文）。
代码片段（复现命令）按 CC BY 4.0 一并许可。

## 边界与免责声明

- 调查方式为**静态分析**：未运行该 App、未接触其任何服务器、未申请任何账户；
- 未决事项（如运行期调用占比、活体判定信任模型）在报告 §8 如实列出，结论不越证据边界；
- 责任归属依据静态证据与公开工商信息，**最终认定权在相关监管与司法机构**；
- 本 repo 仅用于安全改进与学术研究目的。
