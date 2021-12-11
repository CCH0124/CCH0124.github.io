---
title:  掛載硬碟
date: 2018-09-15
description: 因為學校系統的硬碟壞了
tags: [Ubuntu, mount]
draft: false
---
# 事由
那年是我剛進實驗室的第 2 個月，因為我對 linux 系統較熟，所以我接了管理的這一部分。沒想到此系統沒幾天後硬碟竟然壞了，於是我呼叫我朋友幫我處裡硬碟部分（硬碟還原），我負責掛載。

## 分割區切割
Linux 的分割區有 90 多種，但實際只會用到四種，數字表分割代碼
- Linux swap（82）
   - 虛擬記憶體
- Linux（83）
   - 適和使用分割區
- Extended（5）
   - 要劃分四個分割區以上，必須使用
- Linux LVM（8e）
   - LVM 分割區

## 新增硬碟至 ubuntu
檢查第二顆是否有偵測到
```shell=
$ dmesg |grep hd
```
再次確認
```shell
$ sudo fdisk -l /dev/sdb # 確認硬碟分割區狀態
[sudo] password for itachi:

Disk /dev/sdb: 2000.4 GB, 2000398934016 bytes
255 heads, 63 sectors/track, 243201 cylinders, total 3907029168 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disk identifier: 0x00000000

Disk /dev/sdb doesn't contain a valid partition table
```

## 接著開始分割
進入 `fdisk`
```shell
$ sudo fdisk /dev/sdb
Device contains neither a valid DOS partition table, nor Sun, SGI or OSF disklabel
Building a new DOS disklabel with disk identifier 0x5b5b43aa.
Changes will remain in memory only, until you decide to write them.
After that, of course, the previous content won't be recoverable.

Warning: invalid flag 0x0000 of partition table 4 will be corrected by w(rite)

The device presents a logical sector size that is smaller than
the physical sector size. Aligning to a physical sector (or optimal
I/O) size boundary is recommended, or performance may be impacted.

Command (m for help): m # 顯示各種指令
Command action
   a   toggle a bootable flag
   b   edit bsd disklabel
   c   toggle the dos compatibility flag
   d   delete a partition
   l   list known partition types
   m   print this menu
   n   add a new partition
   o   create a new empty DOS partition table
   p   print the partition table
   q   quit without saving changes
   s   create a new empty Sun disklabel
   t   change a partition's system id
   u   change display/entry units
   v   verify the partition table
   w   write table to disk and exit
   x   extra functionality (experts only)
```

### 過程
```shell
Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-3907029167, default 2048):
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-3907029167, default 3907029167):
Using default value 3907029167

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.

```
首先我們要新增一個分割區，步驟如下
1. 新增分割區，輸入 n 按 Enter。
2. 選擇要建立 extended 還是 primary partition，因為我的硬碟全部只要一個分割區，所以我選 primary，輸入 p 按 Enter。
3. 選擇 Partition number，primary 分割區最多可以有四個，隨便選都可以，不過建議選 1，免得以後看起來很奇怪，輸入 1 按 Enter。
4. 輸入開始的 sector，用預設值就可以了，直接按 Enter。
5. 輸入結束的 sector，若是要用最大的容量，就直接按 Enter，若是要指定分割區的大小，就用 +size{K,M,G} 的形式指定，例如指定為 100G 的大小就輸入 +100G 再按 Enter。
6. 最後將分割表寫入硬碟，輸入 w 再按 Enter。

### 檢查
```shell
$ sudo fdisk -l /dev/sdb

Disk /dev/sdb: 2000.4 GB, 2000398934016 bytes
81 heads, 63 sectors/track, 765633 cylinders, total 3907029168 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disk identifier: 0x5b5b43aa

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048  3907029167  1953513560   83  Linux

```

## 格式化（Format）硬碟
分割區劃分後，需格式化後才能使用
- 常用的包括 `ext2`、`ext3`、`ext4`、`xfs` 等
- Windows 底下常用檔案包括 `ntfs`、`fat32`

```shell
$ sudo mkfs -t ext4 /dev/sdb1
mke2fs 1.42 (29-Nov-2011)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
122101760 inodes, 488378390 blocks
24418919 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=4294967296
14905 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968,
        102400000, 214990848

Allocating group tables: done
Writing inode tables: done
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done
```

### 查詢目前檔案系統
- `df -T`
- `mount`
- `cat /etc/fstab`
- 為掛載狀態
   - `file -s /dev/sdb*`

## 查看 UUID
```shell
$ sudo blkid
/dev/sda1: UUID="87231b42-c88b-4111-ae5b-9650f743eafd" TYPE="ext4"
/dev/sda5: UUID="d2ba3a6d-7a41-4eaf-ad01-bc81c279d82a" TYPE="swap"
/dev/sdb1: UUID="59af9c6e-a7e8-4f05-99c0-492e16ec5df9" TYPE="ext4"

```

## /etc/fstab 資訊
- 檔案系統設定檔
- 開機將會自動掛載
- 一行設定一個檔案系統

```shell
# /etc/fstab: static file system information.
  2 #
  3 # Use 'blkid' to print the universally unique identifier for a
  4 # device; this may be used with UUID= as a more robust way to name devices
  5 # that works even if disks are added and removed. See fstab(5).
  6 #
  7 # <file system> <mount point>   <type>  <options>       <dump>  <pass>
  8 proc            /proc           proc    nodev,noexec,nosuid 0       0
  9 # / was on /dev/sda1 during installation
 10 UUID=87231b42-c88b-4111-ae5b-9650f743eafd /               ext4    errors=remount-ro 0       1
 11 # swap was on /dev/sda5 during installation
 12 UUID=d2ba3a6d-7a41-4eaf-ad01-bc81c279d82a none            swap    sw              0       0
 13 # The Data Hardisk
 14 #UUID=f1f0d3c0-7789-4bd5-98a0-52ea86076bdd /home/home1    ext3    defaults        0
 15
 16 UUID=bec08759-3ca6-4982-b8e0-6b00522038b9 /home/home1     ext4    defaults        0
 17 UUID=FC9E-C961 /mnt/usb   vfat    defaults        0
~
```


- file system 
   - 目錄名稱
- mount point   
   - 要掛載的目錄
- type
   - 檔案系統類型（按照格式化的格式）
- options
   - 掛載的選項，`mount -o` 的選項
   - 而外需求從這設定
- dump
   - 0 表示不用 `dump` 備份
   - 1 表示用 `dump` 備份
   - 2 表示其他週期用 `dump` 備份
- pass
   - 開機是否用 `fsck` 檢查分割區
   - 0 不檢查
   - 1、2 檢查，前者優先順序高於後者


## 格式化檔案系統介紹
### ext2
- 無 log 功能
- 不論檔案形式都存在一個 indoe 中，包含使用者資訊、檔案權限或大小等
- 硬碟容量最多支援到 32 TB
- 單一檔案最大支援到 2 TB

### ext3
- 與 ext2 相比多了 log 功能
- 可無痛從 ext2 轉 ext3

硬碟最大支援和單一檔案最大支援是一樣的。

### ext4
和前兩個比
- 硬碟容量最多支援到 1 EB
- 單一檔案最大支援到 16 TB
- 單一目錄最多放 64000 個子目錄（ext3 只能 32000 個）
- 優化 ext3 效率和可靠性

### xfs
- 處裡大型檔案較快速，但建檔刪檔或處裡多個小檔案效率不佳
   - 刪除一個 1 TB 檔案和刪除 1 KB 檔案一樣
- 掛載速度極快，CPU 占用低

### btrfs
- B-tree 檔案系統
- 支援先進磁碟功能
   - 磁碟快照
   - 快照的快照
   - 磁碟陣列
   - 動態掛載
   - 等
- 對 SSD 做優化

安裝的套件為 `btrfs-tools`

## 掛載與卸載
磁碟格式化完後，需使用就要掛載。掛載是一種建立鏈結的關係，不會去動到分割區內原始資料

掛載指令使用 `mount`

|參數|功能|
|---|---|
|-a|掛載所有列在 `/etc/fstab` 的檔案系統|
|-L|掛載特定名稱的檔案系統|
|-t|指定檔案系統類型，支援 `ext2` `ext3` `ext4` 等|
|-r|掛載唯獨模式|
|-w|掛載可讀寫模式|
|-o|async：所有存取不同步進行|
||atime：每次存取自動更新存取時間| 
||auto：自動掛載|
||noatime：不即時更新檔案系統存取時間|
|| ro：掛載為唯獨檔案系統|
|| rw：掛載為可讀寫檔案系統|
|| default：預設選項|

卸載指令使用 `unmount`

|參數|功能|
|---|---|
|-a|將 `/etc/fstab` 內以掛載的檔案系統全部卸載|
|-r|卸載失敗，以唯獨模式掛載|
|-t|只卸載指定檔案系統|

## 掛載 exfat 方式

因為我朋友是在 windows 進行硬碟操作，我記得好像我是要複製東西(從 win 到 Liunx)。因此要掛載需要使用 exfat 方式。

```shell
$ sudo apt-get install exfat-fuse -y
$ sudo mount -t exfat /dev/sdg1 /mnt/usb
```


## 後續還原的問題與處裡

### 網卡問題
ubuntu 12.0 ver
82579v 需要安裝驅動

### 強制更換 MAC

```shell
hwaddress ether AA:BB:CC:DD:EE:FF
```
### apache 問題

2.2 up to 2.4
在 Apache 2.2 是這樣寫：
```php
Order allow, deny
Allow from all
```
在 Apache 2.4 要改成：
```php
Require all granted
```
### 還原遇到的問題

move_uploaded_file 有問題的函示
/home/home1/143GB/...../upload/ 這為上傳檔案的資料夾必須賦予 www-data 權限
```shell
$ sudo chown www-data:www-data /home/home1/143GB/...../upload/

```

### sudo 處裡

```shell
$ sudo vi /etc/sudoers
[sudo] password for jni:
jni is not in the sudoers file.  This incident will be reported.
```

```shell
itachi@ubuntu:~$ sudo vi /etc/sudoers
 # User privilege specification
 username     ALL=(ALL:ALL) ALL

```