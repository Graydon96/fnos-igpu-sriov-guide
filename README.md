# SR-IOV 核显直通飞牛影视转码修复记录
本次使用到的项目地址：https://github.com/strongtz/i915-sriov-dkms  
以下为此次踩坑修复记录，解决步骤可直接跳到步骤6
## 1. SR-IOV 技术简介

SR-IOV，全称是 Single Root I/O Virtualization，是一种 PCIe 设备虚拟化技术。它允许一块物理设备被拆分成多个可独立分配的虚拟功能。

在 SR-IOV 里通常有两类功能：

- PF，Physical Function，物理功能，由宿主机管理，用来创建和控制 VF。通常以00结尾的是PF
- VF，Virtual Function，虚拟功能，可以直通给虚拟机使用。以0X结尾的是VF

对于 Intel 核显来说，开启 SR-IOV 后，Unraid 宿主机可以把同一块核显拆成多个 VF，然后分别分配给不同虚拟机。这样多个虚拟机就有机会同时使用核显能力，例如 VAAPI/QSV 硬件解码和转码。

但 Intel iGPU SR-IOV 对驱动链路要求比较高。宿主机需要能正确创建 VF，虚拟机也需要能正确驱动这个 VF。虚拟机里能看到核显 PCI 型号，并不代表硬件转码已经可用。真正可用的标志通常是虚拟机内出现 `/dev/dri/renderD128` 或类似 render 节点，并且 VAAPI/QSV 能正常调用。

## 2. 问题现象

环境大致如下：

- 宿主机：Unraid
- 虚拟机：飞牛 fnOS
- 核显：Intel AlderLake-S GT1 `[8086:4680]`
- 方案：Unraid 通过 SR-IOV 插件创建 Intel 核显 VF，并直通给飞牛虚拟机
- 飞牛内核：`6.18.18-trim`

最初现象：

- 飞牛系统硬件信息里能看到核显型号。
- 飞牛影视里能看到核显，但无法启用 GPU 加速转码。
- `/dev/dri` 下没有 `renderD128`。
- 内核日志报错：

```text
i915 0000:06:00.0: [drm] *ERROR* Device is non-operational; MMIO access returns 0xFFFFFFFF!
i915 0000:06:00.0: [drm] *ERROR* Device initialization failed (-5)
i915 0000:06:00.0: probe with driver i915 failed with error -5
```

## 3. 根本原因

这次问题不是飞牛影视本身的问题，也不是简单的权限问题。

根因有两层：

1. Intel 核显 VF 不能作为虚拟机主显卡使用。
2. 飞牛默认内核里的 i915 驱动无法正确驱动 Unraid 创建的 Intel iGPU SR-IOV VF。

一开始把核显 VF 设置成虚拟机主显卡时，飞牛里的 i915 无法访问 VF 的 MMIO 区域，因此报：

```text
MMIO access returns 0xFFFFFFFF
```

后来把主显卡改为 Unraid 自带虚拟显卡，将 Intel VF 作为第二显卡后，问题向前推进，但仍然需要 guest 端安装支持 SR-IOV VF 的 i915 驱动。

通过 Ubuntu 24.04 测试确认：

- 未安装 `i915-sriov-dkms` 时，Ubuntu 也无法生成 `renderD128`。
- 安装 `i915-sriov-dkms` 并启用 GuC 后，Ubuntu 可以生成 `/dev/dri/renderD128`。

因此可以确认：Unraid SR-IOV 和 QEMU 直通链路本身是可行的，关键是虚拟机内需要支持 SR-IOV VF 的 i915 驱动。

## 4. Unraid 宿主机配置

### 4.1 添加内核启动参数

在 Unraid Web UI 中进入：

```text
Main -> Flash -> Syslinux Configuration
```

在当前启动项的 `append` 行后面加入：

```text
intel_iommu=on i915.enable_guc=3
```

示例：

```text
append initrd=/bzroot intel_iommu=on i915.enable_guc=3
```

保存后重启 Unraid。

### 4.2 确认启动参数生效

重启后在 Unraid SSH 中执行：

```bash
cat /proc/cmdline
```

确认输出中包含：

```text
intel_iommu=on
i915.enable_guc=3
```

### 4.3 检查 SR-IOV VF 状态

查看核显设备：

```bash
lspci -Dnn | grep -Ei "vga|display|intel"
```

查看 i915 / GuC / HuC 状态：

```bash
dmesg | grep -Ei "i915|guc|huc|drm"
```

正常情况下可以看到类似：

```text
i915 0000:00:02.1: Running in SR-IOV VF mode
i915 0000:00:02.1: GuC firmware PRELOADED version 0.0 submission:SR-IOV VF
i915 0000:00:02.1: HuC firmware PRELOADED
```

### 4.4 确认 VF 绑定到 vfio-pci

飞牛虚拟机要直通的是 VF，例如 `00:02.1`，不是 PF `00:02.0`。

检查 VF 当前驱动：

```bash
lspci -nnk -s 00:02.1
```

目标结果：

```text
Kernel driver in use: vfio-pci
```

如果显示：

```text
Kernel driver in use: i915
```

说明 VF 仍被 Unraid 宿主机占用，需要绑定到 VFIO。

可以在 Unraid Web UI 中进入：

```text
Tools -> System Devices
```

勾选：

```text
00:02.1 Intel Corporation AlderLake-S GT1 [8086:4680]
```

然后选择：

```text
Bind selected to VFIO at boot
```

也可以临时手动绑定：

```bash
echo 0000:00:02.1 > /sys/bus/pci/devices/0000:00:02.1/driver/unbind
echo vfio-pci > /sys/bus/pci/devices/0000:00:02.1/driver_override
echo 0000:00:02.1 > /sys/bus/pci/drivers_probe
```

再次确认：

```bash
lspci -nnk -s 00:02.1
```

## 5. 飞牛虚拟机配置

### 5.1 显卡配置

飞牛 VM 不要把 Intel VF 作为主显卡。

推荐配置：

```text
Machine: Q35
BIOS: OVMF
主显卡: Unraid 自带虚拟显卡 / VNC
第二显卡: Intel VF，例如 00:02.1
不要直通 00:02.0 PF
不要给 Intel VF 添加 ROM 文件
```

如果 VM XML 中需要关闭 ROM BAR，可以在 Intel VF 对应的 `<hostdev>` 中加入：

```xml
<rom bar='off'/>
```

示例：

```xml
<hostdev mode='subsystem' type='pci' managed='yes'>
  <source>
    <address domain='0x0000' bus='0x00' slot='0x02' function='0x1'/>
  </source>
  <rom bar='off'/>
</hostdev>
```

## 6. 飞牛内安装 i915-sriov-dkms

### 6.1 确认内核版本

在飞牛中执行：

```bash
uname -r
```

本次环境输出：

```text
6.18.18-trim
```

### 6.2 安装基础依赖

飞牛的内核是自定义内核，仓库里通常找不到 Ubuntu 风格的 `linux-modules-extra-6.18.18-trim`，这不是关键问题。

先安装基础依赖：

```bash
sudo apt update
sudo apt install -y build-essential dkms wget
```

然后检查是否存在内核 build 目录：

```bash
ls -ld /lib/modules/$(uname -r)/build
```

如果这个目录存在，说明可以尝试编译 DKMS。

### 6.3 安装匹配版本的 i915-sriov-dkms

飞牛内核为 `6.18.18-trim`，对应选择支持 `6.12 ~ 6.19` 区间的 `i915-sriov-dkms` 版本。

下载安装包：

```bash
wget -O /tmp/i915-sriov-dkms.deb "https://github.com/strongtz/i915-sriov-dkms/releases/download/2026.03.05.2/i915-sriov-dkms_2026.03.05.2_amd64.deb"
```

安装：

```bash
sudo dpkg -i /tmp/i915-sriov-dkms.deb
```

如果编译失败，可以查看 DKMS 日志：

```bash
cat /var/lib/dkms/i915-sriov-dkms/*/build/make.log | tail -80
```

## 7. 飞牛系统中启用 GuC 并禁用 xe

SR-IOV VF 需要 GuC submission。否则可能出现：

```text
GuC submission disabled
Device initialization failed (-19)
```

创建 i915 参数文件：

```bash
echo "options i915 enable_guc=3" | sudo tee /etc/modprobe.d/i915.conf
```

禁用 xe：

```bash
echo "blacklist xe" | sudo tee /etc/modprobe.d/blacklist-xe.conf
```

编辑 GRUB：

```bash
sudo nano /etc/default/grub
```

确保 `GRUB_CMDLINE_LINUX_DEFAULT` 中包含：

```text
i915.enable_guc=3 module_blacklist=xe
```

示例：

```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet i915.enable_guc=3 module_blacklist=xe"
```

更新 initramfs 和 GRUB：

```bash
sudo update-initramfs -u -k all
sudo update-grub
sudo reboot
```

## 8. 验证是否成功

重启飞牛后，确认 GuC 参数：

```bash
cat /sys/module/i915/parameters/enable_guc
```

目标输出：

```text
3
```

查看内核日志：

```bash
sudo journalctl -k -b | grep -Ei "i915|sriov|guc|huc|vf|MMIO|failed|error"
```

成功时应能看到类似：

```text
i915: You are using the i915-sriov-dkms module
i915 0000:06:00.0: Running in SR-IOV VF mode
```

确认 i915 驱动绑定：

```bash
lspci -nnk | grep -A5 -Ei "vga|display|intel"
```

目标结果：

```text
Kernel driver in use: i915
```

确认 `/dev/dri`：

```bash
ls -l /dev/dri
```

目标是出现：

```text
renderD128
```

或类似：

```text
renderD129
```

## 9. 飞牛影视开启 GPU 加速转码

当 `/dev/dri/renderD128` 出现后，飞牛影视才能真正调用 GPU 转码。

进入飞牛影视设置界面：

```text
设置 -> 转码 -> GPU 加速转码
```

选择 Intel 核显并启用。

测试视频播放时，如果显示正在转码，并且转码任务使用 GPU，即说明修复成功。

## 10. 排障命令汇总

查看 PCI 显卡：

```bash
lspci -nnk | grep -A5 -Ei "vga|display|intel"
```

查看 `/dev/dri`：

```bash
ls -l /dev/dri
```

查看 DRM 设备：

```bash
find /sys/class/drm -maxdepth 1 -type l -printf "%f -> %l\n"
```

查看 i915 / SR-IOV 日志：

```bash
sudo journalctl -k -b | grep -Ei "i915|sriov|guc|huc|vf|MMIO|failed|error"
```

查看 i915 GuC 参数：

```bash
cat /sys/module/i915/parameters/enable_guc
```

查看 Unraid VF 驱动绑定：

```bash
lspci -nnk -s 00:02.1
```

## 11. 最终结论

本次问题的核心不是飞牛影视无法识别核显，而是飞牛虚拟机中的默认 i915 驱动无法正确驱动 Intel iGPU SR-IOV VF。

最终修复条件包括：

- Unraid 开启 `intel_iommu=on i915.enable_guc=3`
- SR-IOV VF 正确创建
- VF 绑定到 `vfio-pci`
- 飞牛 VM 使用虚拟显卡作为主显卡
- Intel VF 作为第二显卡直通
- 飞牛中安装 `i915-sriov-dkms`
- 飞牛中启用 `i915.enable_guc=3`
- 飞牛中禁用 `xe`
- `/dev/dri` 中出现 `renderD128`

完成后，飞牛影视可以成功开启 GPU 加速转码，并且视频播放测试显示转码正常。
<img width="1233" height="369" alt="9f941509598d8c2b2848ab4aea282c5a" src="https://github.com/user-attachments/assets/3769e3df-c4f4-4478-b823-6d6317d0c2e1" />
<img width="801" height="402" alt="ec50b69db25e84684034ae3215d60771" src="https://github.com/user-attachments/assets/b21a1204-16fa-4522-bcdb-5a233779e24f" />


