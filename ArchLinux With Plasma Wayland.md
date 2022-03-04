# 安裝前的準備
（UEFI+GPT）
1. 下載ArchLinux每個月更新的iso鏡像
1. 刻錄到U盤（Win: Rufus）
1. 到BIOS或虛擬機設置中關閉Secure Boot
1. 啓動到live環境

# 安裝

1. 關閉reflector服務
```sh
systemctl stop reflector.service
```
2. 確認UEFI啓動
```sh
ls /sys/firmware/efi/efivars
```
3. 聯網  
- 虛擬機或者有線連接：無需額外操作  
- 無線：
```sh
iwctl # 进入交互式命令行
device list # 列出无线网卡设备名，比如无线网卡看到叫 wlan0
station wlan0 scan # 扫描网络
station wlan0 get-networks # 列出所有 wifi 网络
station wlan0 connect [wifi-name] # 进行连接，注意这里无法输入中文。回车后输入密码即可
exit # 连接成功后退出
```
4. 系統時間
```sh
timedatectl set-ntp true # 将系统时间与网络时间进行同步
timedatectl status # 检查服务状态
```
5. 更換國內軟件源
```sh
vim /etc/pacman.d/mirrorlist
Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch # 中国科学技术大学开源镜像站
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch # 清华大学开源软件镜像站
Server = https://repo.huaweicloud.com/archlinux/$repo/os/$arch # 华为开源镜像站
Server = http://mirror.lzu.edu.cn/archlinux/$repo/os/$arch # 兰州大学开源镜像站
```
6. 設置分區，至少包括1個efi啓動分區和1個主分區  
  - 如果要雙啓動到已有的Windows，則使用windows創建的啓動分區
```sh
lsblk # 查看分區情況
cfdisk /dev/[nvme*n1/sd*] # 交互式工具，退出前需將改動顯式寫入磁盤，注意不要炸掉已有的分區（比如重建分區表）
```
7. 格式化爲btrfs並創建子卷  
格式化
```sh
mkfs.btrfs -L [PartitionLabel] /dev/sdxn # -L 指定卷標
```
掛載
```sh
mount -t btrfs -o compress=zstd /dev/sdxn /mnt # -t 指定掛載分區類型，-o compress=zstd 開啓透明壓縮
```
複查掛載情況
```sh
df -h # -h 选项会使输出以人类可读的单位显示
```
創建子卷
```sh
btrfs subvolume create /mnt/@ # 创建 / 目录子卷
btrfs subvolume create /mnt/@home # 创建 /home 目录子卷
```
複查
```sh
btrfs subvolume list -p /mnt
```
卸載
```sh
umount /mnt
```
按子卷掛載，從根目錄開始
```sh
mount -t btrfs -o subvol=/@,compress=zstd /dev/nvme*n1p* /mnt # 挂载 / 目录
mkdir /mnt/home # 创建 /home 目录
mount -t btrfs -o subvol=/@home,compress=zstd /dev/nvme*n1p* /mnt/home # 挂载 /home 目录
mkdir -p /mnt/boot/efi # 创建 /boot/efi 目录
mount /dev/nvme*n1p* /mnt/boot/efi # 挂载 /boot/efi 目录
# 若已有windows啓動分區則要掛載已有的那個，可根據Windows內磁盤管理顯示的大小判斷
# swapon /dev/nvmexn1pn # 挂载交换分区，若有
```
複查掛載情況
```sh
df -h # -h 选项会使输出以人类可读的单位显示
```
8. 安裝系統
```sh
pacstrap /mnt base base-devel linux linux-firmware
```
安裝基礎工具
```sh
pacstrap /mnt dhcpcd iwd vim sudo zsh zsh-completions
```
9. genfstab
```sh
genfstab -U /mnt > /mnt/etc/fstab
```
檢查
```sh
cat /mnt/etc/fstab # 特別檢查啓動分區是不是掛對了
```
10. 進入新系統
```sh
arch-chroot /mnt
```
11. 設置主機名
```sh
vim /etc/hostname
[hostname]
vim /etc/hosts
127.0.0.1   localhost
::1         localhost
127.0.1.1   [hostname].localdomain  [hostname]
```
12. 時區、同步硬件時鐘
```sh
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
hwclock --systohc
```
13. 區域設置
```sh
vim /etc/locale.gen # 去註釋en_US.UTF8和zh_CN.UTF8
locale-gen
echo 'LANG=en_US.UTF-8'  > /etc/locale.conf
```
14. 給root加密碼
```sh
passwd root
```
15. 安裝微碼
```sh
pacman -S intel-ucode # Intel
pacman -S amd-ucode # AMD
```
16. 安裝引導程序
```sh
pacman -S grub efibootmgr os-prober
# 安裝grub到引導分區
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ARCH
# 修改GRUB_CMDLINE_LINUX_DEFAULT爲"loglevel=5 nowatchdog"
vim /etc/default/grub
# 去註釋GRUB_DISABLE_OS_PROBER=false以引導windows
```

# 後續工序

1. 配置root的編輯器
```sh
vim ~/.bash_profile
# export EDITOR='vim'
```
2. 添加非root用戶
```sh
useradd -m -G wheel -s /bin/zsh [username]
# -m 同時創建home -G 指定用戶組 -s 指定用戶使用的shell
passwd [username]
EDITOR=vim visudo # 上一條尚未生效
打開註釋#%wheel ALL=(ALL) ALL 
```
3. 安裝桌面環境
```sh
pacman -S plasma-meta konsole dolphin plasma-wayland-session # plasma-meta 元软件包以及wayland支持、konsole 终端模拟器和 dolphin 文件管理器
```
4. 啓用sddm TODO：移除這個步驟，直接登錄到plasma
```sh
systemctl enable sddm
```
5. 到桌面環境裡關閉iwd，開啓NetworkManager
```sh
sudo systemctl disable iwd # 确保 iwd 开机处于关闭状态，因为其无线连接会与 NetworkManager 冲突
sudo systemctl stop iwd # 立即关闭 iwd
sudo systemctl enable --now NetworkManager # 确保先启动 NetworkManager，并进行网络连接。若 iwd 已经与 NetworkManager 冲突，则执行完上一步重启一下电脑即可
ping www.bilibili.com # 测试网络连通性
```
7. 开启 32 位支持库与 Arch Linux 中文社区仓库（archlinuxcn）
```sh
vim /etc/pacman.conf
# 打開multilib，添加archlinuxcn
```
6. 裝輸入法 TODO：確認哪一步可以加入
```sh
sudo pacman -S fcitx5-rime fcitx5-qt fcitx5-configtool
```
7. Wayland支持：`/etc/environment`
- Firefox
```sh
MOZ_ENABLE_WAYLAND=1
```
- 輸入法
```sh
INPUT_METHOD=fcitx5
GTK_IM_MODULE=fcitx5
QT_IM_MODULE=fcitx5
XMODIFIERS=\@im=fcitx5
SDL_IM_MODULE=fcitx5
```
- Chromium  
在`chrome://flags`裡搜索wayland，打開相關選項
