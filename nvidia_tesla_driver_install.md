---
tags: notes, install, driver, nvidia
---

# NVIDIA Tesla Driver 安裝筆記
- 測試環境 : CentOS 7 x86_64
- 本文連結 : https://hackmd.io/@kmo/notes_nvidia_tesla_driver_install

## 前言
- Tesla 系列的 GPU 通常用於 Data Center、HPC 等大型機群的情境  
NVIDIA 針對 Tesla 的 driver 特別拉出一份[文件](https://docs.nvidia.com/datacenter/tesla/index.html)說明，並有不同 [support 週期](https://docs.nvidia.com/datacenter/tesla/drivers/#lifecycle) 
- production 環境應避免從 CUDA 安裝包安裝 driver，而是從 NVIDIA driver 頁面下載安裝
  - 在 [CUDA release note](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html) 搜尋 `Tesla`，有此說明 ![](https://i.imgur.com/zTe18Gt.png =500x)

## 檢查 GPU 型號
- 列出系統的 NVIDIA 設備，並確認 GPU 是 Tesla 型號
```bash=
lspci -d 10DE: | grep -i tesla
```
- `10DE` 是 NVIDIA 的 PCIE 代碼
- ref : https://devicehunt.com/view/type/pci/vendor/10DE
## 查閱 Release Notes
- 查閱 [Tesla Driver Release Notes](https://docs.nvidia.com/datacenter/tesla/index.html)
- 建議時常查閱發行版的 `Fixed Issues` 及 `Known Issues`

## 選擇 Tesla Driver 版本
- 參考 [NVIDIA Tesla Driver Lifecycle](https://docs.nvidia.com/datacenter/tesla/drivers/#lifecycle)
- 建議選擇 Long Term Service Branch (LTS)，目前建議選擇 `R450` 系列
![](https://i.imgur.com/ERn8Kfj.png =700x)


## 下載 Tesla Driver
- 下載連結 : https://www.nvidia.com/Download/Find.aspx
- 語言選 `Chinse(Traditional)`，下載連結會是 `tw.` 開頭
![](https://i.imgur.com/km5CqFy.png =500x)

## 放置 Driver 安裝檔到 Local Yum Repository
- 維運大型機群，建議在內部環境建立 local yum repository，維持機群的 driver 版本一致，也能避免因操作失誤等行為誤升級 driver 
- 這邊是已有 local yum repository，下載及放置 driver rpm 到指定路徑的步驟
- local yum repository server 
```bash=
# 下載指定版本 driver 
curl -R -O https://tw.download.nvidia.com/tesla/450.119.04/nvidia-driver-local-repo-rhel7-450.119.04-1.0-1.x86_64.rpm

# 解壓縮 rpm
rpm2cpio nvidia-driver-local-repo-rhel7-450.119.04-1.0-1.x86_64.rpm | cpio -idv

# 移動解壓縮後的資料夾，到指定讀取的路徑
mv ./var/nvidia-driver-local-repo-rhel7-450.119.04 nv_rpms_450.119.04
```
- client 的 yum 設定 `/etc/yum.repos.d/nvidia-local.repo`
```bash=
[nv-450.119.04]
name=yum repository for nv_rpms_450.119.04
baseurl=http://your_repo_server_ip_and_path/nv_rpms_450.119.04
enabled=1
gpgcheck=0
```

## 安裝 Driver
- 安裝步驟
```bash=
# 清除 yum cache
yum clean expire-cache
# 安裝 driver
yum install -y nvidia-driver-latest-dkms

# 如果需要圖形介面 (x window) 則需要再執行此步驟
yum install cuda-drivers
```
- ref : https://docs.nvidia.com/datacenter/tesla/tesla-installation-notes/index.html


## 啟用 Service 以及檢查測試
- 安裝完 driver，建議啟用 service `nvidia-persistenced`
```bash=
# 啟用 nvidia-persistenced 
systemctl start nvidia-persistenced

# 開機自動啟用 nvidia-persistenced
systemctl enable nvidia-persistenced

# 檢查 driver 版本
cat /proc/driver/nvidia/version

# 測試基本指定能否正常執行和顯示
nvidia-smi 

# 如果有異常，可能需要重開機
systemctl reboot
```
ref : https://stackoverflow.com/a/13127714

## 檢查 RPM script
- 檢查 rpm 的 pre/post 的 script，掌握裝 driver 的 rpm 時，額外執行了什麼動作
- 比如有 
  - 新增 nvidia-persistenced user (nvidia-persistenced-latest*.rpm)
  - 新增 kernel cmd (nvidia-driver-latest-*.rpm)
```bash=
# 檢查安裝的 rpm pre/post script
NV_VER=450.119.04
rpm -qp --scripts nvidia-persistenced-latest-dkms-$NV_VER*.rpm
rpm -qp --scripts nvidia-driver-latest-dkms-$NV_VER*.rpm
```

:::spoiler 檢查 `nvidia-persistenced-latest-dkms` 的 script
- `rpm -qp --scripts nvidia-persistenced-latest-dkms-450.119.04-1.el7.x86_64.rpm`
- output
```bash=
preinstall scriptlet (using /bin/sh):
getent group nvidia-persistenced >/dev/null || groupadd -r nvidia-persistenced
getent passwd nvidia-persistenced >/dev/null || \
    useradd -r -g nvidia-persistenced -d /var/run/nvidia-persistenced -s /sbin/nologin \
    -c "NVIDIA persistent software state" nvidia-persistenced
exit 0
postinstall scriptlet (using /bin/sh):

if [ $1 -eq 1 ] ; then
        # Initial installation
        systemctl preset nvidia-persistenced.service >/dev/null 2>&1 || :
fi
preuninstall scriptlet (using /bin/sh):

if [ $1 -eq 0 ] ; then
        # Package removal, not upgrade
        systemctl --no-reload disable nvidia-persistenced.service > /dev/null 2>&1 || :
        systemctl stop nvidia-persistenced.service > /dev/null 2>&1 || :
fi
postuninstall scriptlet (using /bin/sh):

systemctl daemon-reload >/dev/null 2>&1 || :
if [ $1 -ge 1 ] ; then
        # Package upgrade, not uninstall
        systemctl try-restart nvidia-persistenced.service >/dev/null 2>&1 || :
fi
```
:::

:::spoiler 檢查 `nvidia-driver-latest-dkms` 的 script
- `rpm -qp --scripts nvidia-driver-latest-dkms-450.119.04-1.el7.x86_64.rpm`
- output
```bash=
postinstall scriptlet (using /bin/sh):
/usr/sbin/grubby --update-kernel=ALL --args='nouveau.modeset=0 rd.driver.blacklist=nouveau' &>/dev/null
. /etc/default/grub
if [ -z "${GRUB_CMDLINE_LINUX}" ]; then
  echo GRUB_CMDLINE_LINUX="nouveau.modeset=0 rd.driver.blacklist=nouveau" >> /etc/default/grub
else
  for param in nouveau.modeset=0 rd.driver.blacklist=nouveau; do
    echo ${GRUB_CMDLINE_LINUX} | grep -q $param
    [ $? -eq 1 ] && GRUB_CMDLINE_LINUX="${GRUB_CMDLINE_LINUX} ${param}"
  done
  sed -i -e "s|^GRUB_CMDLINE_LINUX=.*|GRUB_CMDLINE_LINUX=\"${GRUB_CMDLINE_LINUX}\"|g" /etc/default/grub
fi
if [ "$1" -eq "2" ]; then
  # Remove no longer needed options
  /usr/sbin/grubby --update-kernel=ALL --remove-args='nomodeset gfxpayload=vga=normal' &>/dev/null
  for param in nomodeset gfxpayload=vga=normal; do
    sed -i -e "s|$param ||g" /etc/default/grub
  done
fi || :
preuninstall scriptlet (using /bin/sh):
if [ "$1" -eq "0" ]; then
  /usr/sbin/grubby --update-kernel=ALL --remove-args='nouveau.modeset=0 rd.driver.blacklist=nouveau' &>/dev/null
  for param in nouveau.modeset=0 rd.driver.blacklist=nouveau; do
    sed -i -e "s|$param ||g" /etc/default/grub
  done
fi ||:
```
:::
## Ansible
- NVIDIA 有維護安裝 driver 的 ansible role
- ref : https://github.com/NVIDIA/ansible-role-nvidia-driver


---
[![CC BY-NC-SA 4.0][cc-by-nc-sa-image]][cc-by-nc-sa] This work is licensed under a [CC BY-NC-SA 4.0][cc-by-nc-sa]

[cc-by-nc-sa]: https://creativecommons.org/licenses/by-nc-sa/4.0
[cc-by-nc-sa-image]: https://licensebuttons.net/l/by-nc-sa/4.0/88x31.png