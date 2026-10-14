+++
title = "用 Multipass 在 Mac 上開三台 Ubuntu VM，打造 Home Lab"
date = 2026-10-14T15:00:00+08:00
draft = false
description = "在 Apple Silicon Mac 上用 Multipass 開三台 Ubuntu VM，設定 SSH、更新系統、讓 VM 互通，最後存快照，為之後架 Kubernetes 做準備。"
tags = ["Homelab", "Multipass", "Linux", "Ubuntu", "SSH", "網路"]
categories = ["筆記"]
series = ["Homelab"]
+++

為了之後練 Kubernetes，我在 Mac 上用 Multipass 開了三台 Ubuntu VM，模擬一個小型機房。這篇記錄建置過程，以及過程中學到的 Linux 和網路觀念。

## 為什麼要三台？

之後要用 K8s 官方的安裝工具 kubeadm，組一個叢集：**1 台 master 負責指揮，2 台 worker 負責跑應用程式**。

| VM | 角色 | IP |
|---|---|---|
| master | 指揮中心（control plane） | 192.168.252.2 |
| worker1 | 跑應用程式 | 192.168.252.3 |
| worker2 | 跑應用程式 | 192.168.252.4 |

worker 開兩台，是為了實際看到 K8s 的「自動修復」：關掉其中一台，上面的應用程式過幾分鐘就會被搬到另一台繼續跑。之後練 Ansible 也用得上，一次設定多台機器，才感受得到自動化省下的工夫。

Mac 上開 VM 的工具很多，像是 Parallels、VMware Fusion、UTM、VirtualBox，但大多要自己下載 ISO、一步步點完安裝畫面。我選了 Ubuntu 官方出的 **Multipass**：一行指令就能開好一台 Ubuntu。

## 1. 開三台 VM

先寫一份 **cloud-init** 設定，讓 VM 開機時自動放入我的 SSH 公鑰、設定時區：

```yaml
#cloud-config
ssh_authorized_keys:
  - ssh-ed25519 AAAA...（我的公鑰）
timezone: Asia/Taipei
package_update: true
```

接著用迴圈一次開三台：

```bash
for name in master worker1 worker2; do
  multipass launch 24.04 --name $name --cpus 2 --memory 2G --disk 20G --cloud-init cloud-init.yaml
done

multipass list   # 確認三台都是 Running，記下 IP
```

> kubeadm 要求每台至少 2GB 記憶體，master 至少 2 CPU。之後不夠用，可以先 `multipass stop`，再用 `multipass set local.<名稱>.memory=4G` 調高。

## 2. 在本機設定 SSH 別名

在 Mac 的 `~/.ssh/config` 加入三台 VM，之後 `ssh master` 就能直接連線，不用記 IP：

```
Host master worker1 worker2
  User ubuntu

Host master
  HostName 192.168.252.2

Host worker1
  HostName 192.168.252.3

Host worker2
  HostName 192.168.252.4
```

共用的設定（`User ubuntu`）寫一次就好，套用到三台。

## 3. 更新系統並重開機

三台都要做，以 master 為例：

```bash
ssh master
sudo apt update && sudo apt upgrade -y
sudo reboot   # 重開機會斷線，自動回到 Mac
```

- `apt update`：更新「軟體清單」，只是確認有哪些新版本
- `apt upgrade`：真正把軟體升級
- `&&`：前一個指令成功才執行下一個

這次更新包含新的 **Linux 核心**，核心要重開機才會換成新版本。等半分鐘左右再 `ssh master` 連回去，用 `uname -r` 確認版本有更新。

## 4. 讓 VM 之間互相認得：/etc/hosts

在 Mac 上能用 `ssh master`，是靠 `~/.ssh/config`；但 VM 之間還不認得彼此的名字，要在每台的 `/etc/hosts` 登記。

### 發現：/etc/hosts 由 cloud-init 管理

改之前先看了一下檔案開頭：

```bash
head -5 /etc/hosts
```

```
# Your system has configured 'manage_etc_hosts' as True.
# ... make changes to the master file in /etc/cloud/templates/hosts.debian.tmpl
```

註解寫得很清楚：`manage_etc_hosts` 是 True，代表這個檔案交給 cloud-init 管理，**每次開機都會用範本 `hosts.debian.tmpl` 重新產生**。直接改 `/etc/hosts`，重開機就會被蓋掉；只改範本，又要等重開機才生效。所以兩個一起改：

```bash
echo "192.168.252.2  master
192.168.252.3  worker1
192.168.252.4  worker2" | sudo tee -a /etc/cloud/templates/hosts.debian.tmpl /etc/hosts
```

> 為什麼不用 `sudo echo ... >> /etc/hosts`？因為 `>>` 是由目前的 shell 寫檔，sudo 管不到，會 Permission denied；交給 `sudo tee` 寫就可以了（`-a` 是加在檔案最後，不會蓋掉原本的內容）。

**心得：改系統設定檔之前，先看開頭的註解。** 它常會告訴你這個檔案由誰管理、該去哪裡改。

### ping 測試

三台都改好後，互相 ping 看看。下面是在 worker2 上 ping master：

```bash
ping -c 3 master
```

`-c 3` 是送 3 個封包就停；不加的話會一直送，要按 Ctrl+C 才會停。

```
PING master (192.168.252.2) 56(84) bytes of data.
64 bytes from master (192.168.252.2): icmp_seq=1 ttl=64 time=0.597 ms
64 bytes from master (192.168.252.2): icmp_seq=2 ttl=64 time=0.489 ms
64 bytes from master (192.168.252.2): icmp_seq=3 ttl=64 time=0.773 ms
```

`master` 被翻成 `192.168.252.2`，代表 `/etc/hosts` 生效了；三個封包都有回應，代表兩台之間網路是通的。

## 5. 存快照

最後把乾淨的狀態存成快照，之後裝 K8s 弄壞了，可以一鍵還原：

```bash
multipass stop --all
for name in master worker1 worker2; do
  multipass snapshot $name --name clean-base
done
multipass start --all

multipass list --snapshots   # 確認快照
```

還原：

```bash
multipass stop master
multipass restore master.clean-base
multipass start master
```

> 小插曲：第一次執行時忘記先 `exit`，在 VM 裡面跑 multipass 指令，結果 `command not found`。**下指令前先看提示字元，確認自己在哪台機器上。**

## 順便學到的網路觀念

### 私有 IP

三台 VM 都是 `192.168.252.x`，這是**私有 IP**，只在內網有效，外面連不進來。常見的三段私有 IP：

| 範圍 | 常見場景 |
|---|---|
| 10.x.x.x | 公司內網、雲端 VPC、K8s |
| 172.16–31.x.x | Docker |
| 192.168.x.x | 家用路由器、虛擬機 |

Multipass 在 Mac 裡建了一個 `192.168.252.0/24` 的虛擬內網，Mac 自己是 `.1`，VM 上網時透過 Mac 轉出去（NAT）。

### 幾個好用的網路指令

```bash
ip addr     # 看 IP、MAC、網路卡狀態
ip neigh    # 看 ARP 表（IP → MAC 的對應）
ip route    # 看路由，default via 192.168.252.1 就是 Mac
ss -tulpn   # 看哪些 port 在監聽
```

從 `ip addr` 看到 IP 後面標著 `dynamic` 和租約時間，代表是 **DHCP** 分配的，所以 VM 的 IP 有可能改變。

從 `ss -tulpn` 學到一個重要觀念：

- `0.0.0.0:22`：監聽所有網路卡，外面連得進來（所以能 SSH）
- `127.0.0.53:53`：只監聽本機，外面連不進來

之後遇到「服務有跑但連不到」，可以先檢查它監聽的是哪個位址。

## 下一步

Home lab 環境準備好了，接下來會在 VM 裡練習 Linux 基礎（權限、行程、systemd），之後就要用 kubeadm 在這三台上架 Kubernetes 叢集 💪