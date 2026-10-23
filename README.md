# 大众 MH2p 车机改装开发笔记（非官方）

在一台 2023 款国产途锐（车机 MH2p_CN_VWG36_P2878）上，从零做出下面几项改装的完整开发过程：每一步遇到什么问题、怎么查到原因、怎么解决。写给想在别的大众 MH2p 车上自己动手的人作参考。

成品（下载即刷的 U 盘包和全部源码）在另一个仓库：**[a78c7/touareg-cn-carplay-mods](https://github.com/a78c7/touareg-cn-carplay-mods)**。本仓库只放文字，不含任何程序和大众文件。

> 本项目与大众汽车、苹果公司无关，不是官方资料。改装车机有风险，可能影响保修，后果自负。

## 目录

| 篇 | 内容 |
| --- | --- |
| [01 调试环境与工具](docs/01-调试环境与工具.md) | SSH、Mac 用的 USB 网卡驱动、QNX 交叉编译的坑、车机自带的 dmdt 调试工具、车机 shell 的限制 |
| [02 CarPlay 导航上 HUD](docs/02-CarPlay导航上HUD.md) | 让 iPhone 发转向数据、把数据送进大众 HMI（Java 插件）、剩余距离/到达时间、断开清理 |
| [03 CarPlay 地图上仪表](docs/03-CarPlay地图上仪表.md) | 让 iPhone 发第二路视频、ChaCha20 解密、H.264 解码上屏、断开恢复原车地图、车标位置校准 |
| [04 仪表信息框](docs/04-仪表信息框.md) | 从车机读功率/扭矩/G 值/油温等实时数据，叠加画在仪表地图上 |
| [05 全屏 CarPlay 与 U 盘包](docs/05-全屏CarPlay与U盘包.md) | 在车上就地打补丁（不分发大众文件）、「改装设置」程序 |
| [06 给其他车型的建议](docs/06-给其他车型的建议.md) | 换一台大众 MH2p 车，先查哪些东西 |

## 标注说明

- **【途锐特有】**：只在这台车、这个软件版本上验证过，别的车多半要重新找（编号、端口、坐标等）。
- **【MH2p 通用】**：平台层面的结论，同平台的大众车大概率一样（奥迪、保时捷 MH2p 也有参考价值）。

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

Development notes (Chinese) from building these mods on a 2023 China-market VW Touareg (`MH2p_CN_VWG36_P2878`, QNX 6.6, Tegra K1, J9 Java HMI), written as a reference for other VW MH2p cars:

- **CarPlay turn-by-turn on the HUD**: `LD_PRELOAD` hook adds iAP2 route-guidance (0x5200–0x5204) to Identify; a J9 OSGi bundle feeds BAP NavSD through ASL datapool 732 / lists 741, 744, 751, 838. J9 only loads class files ≤ version 48.
- **CarPlay map on the instrument cluster**: advertise a second display (`altScreen`, `viewAreas`) in `/info`; stream 111 is ChaCha20-Poly1305 (key from `PairingSessionDeriveKey` with `DataStream-Salt<connID>`); receive on a port the pf firewall allows (6200); NvMedia H.264 decode; show on displayable 58 with the cluster list `[58, 33]` so the stock map (33) stays bound underneath and returns on disconnect.
- **Live info box** (power, torque, G, rpm, oil temperatures, board-computer values) from `DSICarVehicleStates` / `DSICarKombi`, blended over the cluster map.
- **Full-screen CarPlay** patched on the car (38 bytes in two entries of `skin3/viewhandler.zip`), so no VW file is distributed.

Binaries and sources: [a78c7/touareg-cn-carplay-mods](https://github.com/a78c7/touareg-cn-carplay-mods). Not affiliated with Volkswagen or Apple. Text licensed CC BY-SA 4.0.
