# 一加1 (bacon) 刷 postmarketOS 说明

针对本次 Actions 编译产物（`C:\Users\Administrator\Desktop\bacon-pmos`）的刷机操作手册。

---

## 一、产物清单与校验结果

已实测通过，可以刷。

| 文件 | 大小 | 说明 | 校验 |
|---|---|---|---|
| `boot.img` | 7.82 MB | 内核 + initramfs | magic=`ANDROID!` ✅，远低于 boot 分区 16MB 上限 ✅ |
| `oneplus-bacon.img` | 3.12 GB | rootfs（内容约 1GB + 预留 2GB） | 实体文件 ✅ |
| `lk2nd.img` | 320 KB | 备用引导 | magic=`ANDROID!` ✅ |
| `qcom-msm8974pro-oneplus-bacon.dtb` | 49 KB | bacon 专属设备树 | 存在 ✅ |
| `initramfs` / `initramfs-extra` / `vmlinuz` | — | 内核组件 | 实体文件 ✅ |
| `dtbs/` | — | 全平台 dtb 集合 | 实体文件 ✅ |

**关键校验项**：
- 符号链接数量：**0**（上一版产物全是悬空链接，已修复）
- boot.img 7.82MB < bacon boot 分区 16MB 上限，这是最容易翻车的点，当前安全

---

## 二、刷机前准备

### 1. 备份（建议）

进 TWRP 做一次完整备份，至少备份 **boot 分区**，存到电脑。后面回退要用。

### 2. 装驱动

Windows 需要 fastboot 驱动：Google USB Driver 或一加官方驱动。装好后设备管理器里不应出现未知设备。

### 3. 进 fastboot 模式

关机 → 按住 **音量上 + 电源** → 震动后松开 → 屏幕显示 `Fastboot Mode`。

```cmd
fastboot devices
```

能列出设备说明连接正常。

### 4. 解锁 bootloader（已解锁可跳过）

```cmd
fastboot oem unlock
```

一加1 不需要申请，直接可解。**解锁会清空手机数据**。

---

## 三、刷入

```cmd
cd C:\Users\Administrator\Desktop\bacon-pmos

fastboot flash boot boot.img
fastboot flash userdata oneplus-bacon.img
fastboot reboot
```

### 注意事项

1. **rootfs 必须刷 userdata 分区**。千万不要刷 system——该分区只有 1.4GB，而镜像有 3.12GB，刷进去会卡在开机界面。
2. 3.12GB 走 USB 2.0 刷入约 **5~15 分钟**，期间不要拔线、不要中断。
3. 一加1 的 userdata 分区：16GB 版约 12GB，64GB 版约 55GB，都装得下 3.12GB。

---

## 四、首次启动与连接

刷完自动重启。pmOS 默认开启 USB 网络，手机端 IP 固定为 `172.16.42.1`：

```cmd
ssh user@172.16.42.1
```

密码：构建时 workflow 里设置的 `user_password`（默认 `147147`）。

**Windows 连不上时**：检查是否多出一块 RNDIS 网卡，手动给它配 `172.16.42.2`，子网掩码 `255.255.255.0`，再连。

---

## 五、登录后第一件事：扩容

rootfs 现在只有 3.12GB，装 Docker 镜像很快就不够。把文件系统扩到整个 userdata 分区：

```bash
df -h /                          # 查看根分区设备名，如 /dev/mmcblk0p28
sudo resize2fs /dev/mmcblk0p28   # 按实际设备名执行
```

扩完再用 `df -h /` 确认容量已变成十几 GB 或五十几 GB。

---

## 六、装 Docker

```bash
sudo apk add docker docker-cli containerd
sudo rc-update add cgroups boot
sudo rc-update add docker default
sudo service docker start
sudo docker version
```

验证：

```bash
sudo docker run --rm hello-world
```

### armv7 的现实限制

bacon 是 32 位 ARM，Docker Hub 上很多镜像已不再构建 `linux/arm/v7`，拉取时会报 `no matching manifest`。建议：

```bash
sudo docker run --platform linux/arm/v7 alpine uname -m
```

优先选仍支持 32 位 ARM 的镜像：Alpine、Debian armhf、部分 linuxserver.io 镜像。

内存只有 3GB，建议开 zram：

```bash
sudo apk add zramswap
sudo rc-update add zramswap default
sudo service zramswap start
```

---

## 七、已知问题与排错

### 1. 刷 rootfs 报 `Unknown chunk type`

改用 lk2nd 引导：

```cmd
fastboot flash boot lk2nd.img
```

重启，看到 lk2nd 画面时**立刻按住音量下**进入 lk2nd bootmenu，然后再刷 rootfs。

### 2. WiFi 能识别但连不上

MAC 地址可能是全零：

```bash
ip link set dev wlan0 address 70:85:c2:d9:2b:e0
service networkmanager restart
```

为避免每次手动改，写个开机脚本 `/etc/init.d/wlanhwaddr` 并 `rc-update add wlanhwaddr default`。

### 3. 屏幕背光关不掉（当服务器用时）

```bash
echo 0 > /sys/class/backlight/lcd-backlight/brightness
```

### 4. 误把 rootfs 刷进 system 分区

```cmd
fastboot format system
```

然后重新刷 userdata。

### 5. 硬件功能限制（无法修复，主线内核现状）

| 可用 | 不可用 |
|---|---|
| 屏幕、触摸、WiFi、蓝牙、移动数据、SMS、电池、USB 网络 | 音频、相机、通话、GPS、USB OTG、传感器；3D 加速仅部分 |

所以它现在的定位是**低功耗微型服务器**，不是日常手机。

---

## 八、回退到 Android

1. 进 fastboot（音量上 + 电源）
2. 刷回备份的 boot：
   ```cmd
   fastboot flash boot boot-backup.img
   ```
3. 要完整回到 Android：进 TWRP 刷 LineageOS 14.1 (bacon) 官方包 + 对应 GApps
4. 也可重新上锁：
   ```cmd
   fastboot oem lock
   ```

---

## 快速复盘

```
进 fastboot → fastboot oem unlock
→ fastboot flash boot boot.img
→ fastboot flash userdata oneplus-bacon.img   ← 必须是 userdata
→ fastboot reboot
→ ssh user@172.16.42.1（密码 147147）
→ sudo resize2fs /dev/mmcblk0p28
→ sudo apk add docker
```
