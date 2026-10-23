# 大众 MH2p 车机改装开发笔记（非官方）

在一台 2023 款国产途锐（车机 MH2p_CN_VWG36_P2878）上，从零做出下面几项改装的完整开发过程：每一步遇到什么问题、怎么查到原因、怎么解决。写给想在别的 MH2p 车上自己动手的人作参考。

MH2p（MIB2+ High）是大众集团多个品牌共用的车机平台。据 [MH2p SD ModKit 网站](https://lawpaul.github.io/MH2p_SD_ModKit_Site/) 的列表，用这个平台的有：

- 大众：途锐（2018–2024）
- 奥迪（MIB2+ High）：A1、A6、A7、A8、Q3、Q7、Q8、e-tron（各自有年份范围）
- 保时捷（PCM 5）：911（992）、Cayenne（E3）、Macan（95B）、Panamera（976）
- 兰博基尼（LIS III）：Urus（2018–2023）

本笔记只在途锐上验证过。系统层（QNX、J9 Java、DSI 数据接口、CarPlay 库、显示管理）是平台共用的，换品牌大概率能参考；界面皮肤、仪表的显示编号和坐标、补丁偏移是各车各版本自己的，要重新找。每条结论都标了适用范围（见下面"标注说明"）。

成品（下载即刷的 U 盘包和全部源码）在另一个仓库：**[a78c7/touareg-cn-carplay-mods](https://github.com/a78c7/touareg-cn-carplay-mods)**。本仓库只放文字，不含任何程序和大众文件。

> 本项目与大众汽车、苹果公司无关，不是官方资料。改装车机有风险，可能影响保修，后果自负。

## 目录

| 篇 | 内容 |
| --- | --- |
| [01 调试环境与工具](docs/01-调试环境与工具.md) | SSH、Mac 用的 USB 网卡驱动、QNX 交叉编译的坑、不导出车机文件编译插件（桩类）、车机自带的 dmdt 调试工具、车机 shell 和时钟的限制、无效值 |
| [02 CarPlay 导航上 HUD](docs/02-CarPlay导航上HUD.md) | 让 iPhone 发转向数据、把数据送进大众 HMI（Java 插件）、剩余距离/到达时间、断开清理 |
| [03 CarPlay 地图上仪表](docs/03-CarPlay地图上仪表.md) | 让 iPhone 发第二路视频、ChaCha20 解密、H.264 解码上屏、断开恢复原车地图、车标位置校准 |
| [04 仪表信息框](docs/04-仪表信息框.md) | 从车机读功率/扭矩/G 值/油温等实时数据画在仪表地图上；0–100 计时、最大值、扭矩口径实测、行车记录到 U 盘、左右两个框 |
| [05 全屏 CarPlay 与 U 盘包](docs/05-全屏CarPlay与U盘包.md) | 在车上就地打补丁（不分发大众文件）、「改装设置」程序 |
| [06 给其他车型的建议](docs/06-给其他车型的建议.md) | 换一台 MH2p 车（大众、奥迪、保时捷、兰博基尼 Urus），先查哪些东西、哪些能复用 |

## 标注说明

- **【途锐特有】**：只在这台车、这个软件版本上验证过，别的车多半要重新找（编号、端口、坐标等）。
- **【MH2p 通用】**：平台层面的结论，同平台的车大概率一样（大众、奥迪、保时捷、兰博基尼的 MH2p 车机都有参考价值，但只在途锐上验证过）。

## 测试环境

- 车机：MH2p_CN_VWG36_P2878，QNX 6.6.0，NVIDIA Tegra K1（armle-v7），HMI 是 J9 Java（`/ifs/lsd.jxe`）
- 手机：iPhone（iOS 27）；导航 App 测试过高德地图、苹果地图、百度地图
- 刷机：[MH2p SD ModKit](https://lawpaul.github.io/MH2p_SD_ModKit_Site/)（U 盘刷入）

## 致谢

- [LawPaul / MH2p SD ModKit](https://github.com/LawPaul/MH2p_SD_ModKit)：MH2p 用 U 盘改装的基础。
- [fifthBro / mh2p-cluster](https://github.com/fifthBro/mh2p-cluster)、[mh2p-ssh-access](https://github.com/fifthBro/mh2p-ssh-access)：MH2p 仪表投屏的开拓者，`dio_cluster.so` 和 `cluster.c` 是本项目的起点。
- [luka-dev](https://github.com/luka-dev)：mib2q-carplay-rgi（CarPlay 路线指引和第二屏协议研究）、jxe2jar、qnx65-armv7-toolchain。
- [davecozens / touareg-carplay](https://github.com/davecozens/touareg-carplay)：英国途锐全屏方案。

## 许可

文字内容采用 [CC BY-SA 4.0](LICENSE)：可以转载、修改，需署名并以相同方式共享。

---

# VW MH2p head-unit modding notes (unofficial)

Development notes (Chinese) from building these mods on a 2023 China-market VW Touareg (`MH2p_CN_VWG36_P2878`, QNX 6.6, Tegra K1, J9 Java HMI), written as a reference for other cars on the MH2p (MIB2+ High) platform — VW Touareg, Audi (A1/A6/A7/A8/Q3/Q7/Q8/e-tron), Porsche PCM 5 (992, Cayenne E3, Macan 95B, Panamera 976), Lamborghini Urus (LIS III), per the ModKit vehicle list. Only verified on the Touareg:

- **CarPlay turn-by-turn on the HUD**: `LD_PRELOAD` hook adds iAP2 route-guidance (0x5200–0x5204) to Identify; a J9 OSGi bundle feeds BAP NavSD through ASL datapool 732 / lists 741, 744, 751, 838. J9 only loads class files ≤ version 48.
- **CarPlay map on the instrument cluster**: advertise a second display (`altScreen`, `viewAreas`) in `/info`; stream 111 is ChaCha20-Poly1305 (key from `PairingSessionDeriveKey` with `DataStream-Salt<connID>`); receive on a port the pf firewall allows (6200); NvMedia H.264 decode; show on displayable 58 with the cluster list `[58, 33]` so the stock map (33) stays bound underneath and returns on disconnect.
- **Live info box** (power, torque, G, rpm, oil temperatures, board-computer values) from `DSICarVehicleStates` / `DSICarKombi`, blended over the cluster map; 0–100 km/h timing interpolated between samples, session maxima, a CSV drive log to a USB drive (Java writes chunks to `/tmp`, a root boot loop appends them to the drive), invalid sentinel values (-3276.5, 32770), two boxes sharing one renderer.
- **Building without dumping the head unit**: compile-only stubs generated from the constant pool of an already-published plugin jar; rebuilt classes are byte-identical.
- **Full-screen CarPlay** patched on the car (38 bytes in two entries of `skin3/viewhandler.zip`), so no VW file is distributed.

Binaries and sources: [a78c7/touareg-cn-carplay-mods](https://github.com/a78c7/touareg-cn-carplay-mods). Not affiliated with Volkswagen or Apple. Text licensed CC BY-SA 4.0.
