[toc]

# 环境简介
```shell
# 实体机配置如下
一台联想的拯救者笔记本(9代i7/32G内存/1TB硬盘),安装的是linux mint21
一台联想的p360工作站(i7-12700/5代64G内存/1TB硬盘)，安装的linux mint22
2台机器上都有安装vmware workstation17pro,本次就在这2个vmware workstation17pro上完成


# 拓扑设计
[ 物理机A: 拯救者(32G内存) ]                     [ 物理机B: P360(64G内存) ]
       │                                              │
┌──────┴──────────────────────┐                ┌──────┴──────────────────────┐
│ VMware Workstation 17 Pro   │                │ VMware Workstation 17 Pro   │
│                             │                │                             │
│ ┌─────────────────────────┐ │                │ ┌─────────────────────────┐ │
│ │  PVE-Node-01            │ │                │ │  PVE-Node-02            │ │
│ │ (分配 16G 内存)          │ │                │ │ (分配 32G 内存)          │ │
│ └────────────┬────────────┘ │                │ └────────────┬────────────┘ │
│              │              │                │              │              │
│ ┌────────────┴────────────┐ │                │ ┌────────────┴────────────┐ │
│ │  NFS/ZFS Storage VM     │ │                │ │  VMware ESXi 模拟源     │ │
│ │ (作共享存储，分配 4G)    │ │                │ │ (作为迁移源，分配 16G)    │ │
│ └─────────────────────────┘ │                │ └─────────────────────────┘ │
└──────────────┬──────────────┘                └──────────────┬──────────────┘
               │                                              │
               └───────────────[ 局域网物理网线 ]──────────────┘
                         (设置桥接模式 Bridged Network)


1. 桥接网络（Bridged）：让嵌套虚拟机直接互通
在两台物理机的 Mware Workstation中，把虚拟网卡全部设置为桥接模式(Bridged)。这样PVE-Node-01和 PVE-Node-02即可直接获取家里路由器的IP地址，它们之间可以像真实的物理服务器一样通过网线直接通信

2. 模拟真实环境的各节点分工：
P360工作站(64G内存基本盘):
    PVE-Node-02(32G)：它是你的主力计算节点
    Nested-ESXi-7.0/8.0(16G): 在VMware里面安装一个真正的ESXi系统(vMware跑ESXi非常完美),在里面随便塞一个Linux虚拟机作为"待迁移死者"

拯救者笔记本(32G内存协同盘):
    PVE-Node-01(16G)：与Node-02组成 Proxmox 集群(Cluster)
    NFS-Storage-VM(4G)： 用轻量级Debian挂载一块虚拟盘，配置成NFS服务端，同时作为两台PVE的共享存储

注: pve的默认登录帐号/密码是root/自定义的root密码

物理位置	虚拟机名称 	操作系统	推荐配置	拟定静态IP	核心角色
拯救者笔记本	宿主机自身	Linux Mint21	32G内存		192.168.2.111	实验控制台/仲裁节点
└─ VMware桥接	PVE-Node-01	PVE8.x(Debian)	2核/16G		192.168.2.112	虚拟化集群节点1
└─ VMware桥接	NFS-Storage	Debian12	2核/4G		192.168.2.113	唯一共享存储(存放VM镜像)
注: NFS-Storage节点需要加一块大虚拟盘,这里我定义的是200G

P360工作站	宿主机自身	Linux Mint22	64G内存		192.168.2.120	核心算力宿主机
└─ VMware桥接	PVE-Node-02	PVE8.x(Debian)	4核/32G		192.168.2.121	虚拟化集群节点2(主力)
└─ VMware桥接	Nested-ESXi	VMware ESXi7/8	4核/16G		192.168.2.122	被迁移的"旧怨种"服务器

ESXI版本:
esxi8.0.3 24677879
esxi7.0.3 21930508


# 在这个嵌套环境中可做
实验一：官方原生 API 抽干迁移(ESXi → PVE)
动作： 登录PVE-Node-02后台，使用 PVE 8.x 自带的 ESXi 导入器，直接输入你模拟出来的ESXi的IP和密码
验证： 看着进度条走完，虚拟机自动转换格式并在PVE内部拉起

实验二：纯离线手工格式转换与驱动修复(vmdk → qcow2)
动作： 在本地 Linux Mint 上通过 VMware 导出 .vmdk 磁盘，SCP 传输到 PVE 虚拟机中，手动用 qemu-img 命令行转换，并用 qm importdisk 挂载。
过关标准： 虚拟机不蓝屏、不 Kernel Panic，VirtIO 驱动正常加载


当前只有2个实体机节点,HA还是个问题, 待解决


核心底层流向：数据是怎么跑的？
要实现高可用(HA)和无缝迁移则必须理解接下来的两个底层动作:
1. 迁移时的流量走向 (ESXi → PVE-Node-02)
当你坐在拯救者笔记本前，打开浏览器登录https://192.168.1.21:8006(PVE-Node-02 的后台):
你点击"导入ESXi虚拟机"，PVE-Node-02 会通过网络直接向 192.168.1.22 (ESXi) 发起 API 请求
数据流：P360 内部的 ESXi 虚拟机 → P360 的内部虚拟网络 → PVE-Node-02。这个过程由于在物理机P360内部发生，速度极快
转换完成后，PVE-Node-02 会把这个虚拟机的磁盘文件写入到 192.168.1.12(NFS-Storage)上。此时流量会走物理网线送到拯救者笔记本里

2. 高可用故障演练时的走向(HA Live Migration)
正常情况下，虚拟机运行在 PVE-Node-02 (P360) 上，但它的计算数据在 P360 的内存里，磁盘数据其实是通过网线实时读写拯救者上的 NFS-Storage
模拟故障： 你直接在 P360 上把 PVE-Node-02 这台虚拟机"强制关机"(模拟物理机拔电源)
HA触发： PVE-Node-01(拯救者)发现Node-02失联，立刻在本地接管业务。因为磁盘本来就存在旁边的 NFS-Storage 上，PVE-Node-01 只需要在本地直接读取该磁盘并分配CPU/内存，虚拟机就能在几秒钟内死体复活

避坑指南（直接决定实验成败）
1、物理网线直连注意
如果你的拯救者和P360不通过路由器，而是用一根网线直接对连，Linux Mint默认可能会因为没有DHCP分配IP而无法联网。你必须在两台Mint的网络设置里，把有线网卡(Ethernet)手动设置为手动(Manual)/静态IP(比如一端配.10，一端配.20，掩码255.255.255.0，网关留空)

2、VMware桥接网卡选择
在两台机器的 VMware Workstation 中，进入 Virtual Network Editor(虚拟网络编辑器)，把VMnet0(桥接模式绑定的网卡)从"自动(Automatic)"改为你具体的物理网卡名称(比如：Realtek PCIe GB Controller)。千万别桥接到Wi-Fi网卡去了，Wi-Fi往往不支持嵌套虚拟机的MAC地址伪装

3、NFS存储权限
在Debian12上配置NFS时，/etc/exports 配置文件里一定要加上 no_root_squash 参数。因为 PVE 挂载存储时使用的是 root 权限，如果不加这个参数，PVE 将没有权限在 NFS 上创建虚拟机磁盘



# esxi8开启ssh服务,在esxi黄黑控制台：
→ 输入 root 密码
→ Troubleshooting Options                  # 确认以下2项都开启
Enable ESXi Shell = Enabled
Enable SSH = Enabled

然后按ALT + F1 (alt+f2是退回到黄黑的esxi管理页面)，进去执行/etc/init.d/SSH restart            # 不是小ssh

如果已经进入ESXi Shell，常用运维命令：
启动 SSH：vim-cmd hostsvc/start_ssh
设置开机自启：vim-cmd hostsvc/enable_ssh
关闭 SSH：vim-cmd hostsvc/stop_ssh
取消自启：vim-cmd hostsvc/disable_ssh
查看虚拟机：vim-cmd vmsvc/getallvms
查看datastore：esxcli storage filesystem list
重启管理服务：services.sh restart
查看网卡：esxcli network nic list
查看 VM 进程：esxcli vm process list
查看防火墙: esxcli network firewall ruleset list | grep ssh
开启ssh服务：esxcli network firewall ruleset set -e true -r sshServer

```





# 实际生产中的优化(导读)
```shell
这一部分和该文档中的所有可以看作是分开的, 因为这部分会从网络、性能、排错、安全四个维度进一步的深度优化建议, 因为这部分是在文档做完后加的：

在2026年的中小企业和传统行业市场中，除了互联网大厂和金融巨头外，80%的企业VMware虚拟化集群，规模都在10~50台VM之间
即使遇到拥有100台以上虚拟机的客户，他们在做VMware往pve搬迁时，也绝对不会安排在同一个晚上把100台机器全部切过去。大厂的真实操作叫做"分批割接(Phase-based Migration)"
第一周：迁移 5 台无关紧要的测试环境(灰度验证)
第二周：迁移 15 台边缘业务、内部OA(扩大战果)
第三周：迁移 30 台核心 Web、中间件
第四周：最后动核心数据库
这就意味着在每个割接窗口(通常是周六凌晨0:00 - 4:00)，实际经手操作的虚拟机数量通常只有5~15台

在绝大多数实际项目中，老运维都会明令禁止在搬迁时使用Cloud-init
因为Cloud-init的原生设计场景是"从零初始化一个全新的干净系统(如刚购买的阿里云ess 裸机)"，而不是用来"接管一个已经跑了五六年的老系统"

让 virt-v2v 只做它最擅长的事,把驱动(硬盘、网卡)转好，不要让它乱动网络配置文件


一、 网络层：引入"管理、存储、业务"三网分离架构
在企业内绝对不能把所有流量都堆在 192.168.2.x 这一个扁平网络里。大文件迁移的吞吐量(千兆/万兆)会瞬间把路由器的cpu冲爆，导致整个局域网内其他生产业务全部掉线
在实施前必须在pve和物理交换机上划分VLAN来将流量隔离:
管理网(Management): 用于pve集群通信(Corosync协议极其敏感，延迟超过50ms就会导致集群解体)、Web登录
存储网(Storage/Migration): 用于NFS/ZFS挂载和virt-v2v传输。建议至少10Gbps(万兆)起步，并开启MTU 9000(巨型帧，Jumbo Frames)降低CPU软中断
业务网(VM Network): 虚拟机实际对外提供服务的网络

二、性能层：突破存储与内核的极限吞吐
上述用的是 -o local -os /mnt/pve/... -of qcow2。在企业级全闪存(NVMe SSD)或分布式存储(Ceph/ZFS)环境下，这种打法存在严重的双重写入惩罚和序列化瓶颈
生产级写入优化：
1. 彻底放弃qcow2，全面拥抱raw格式或直接落盘到块设备(块级裸联):
在ceph 或pve LVM-Thin存储上，raw格式是绝对的霸主(性能比qcow2高出15%~30%)，且pve底层快照由存储卷自身实现，不需要依赖qcow2的内部快照
修改上述命令将 -of qcow2 改为 -of raw，或者直接输出到pve的lvm池

2. 多线程并发控制 --parallel：
新版 virt-v2v 配合 nbdcopy 支持多线程并发数据块传输。面对企业几十 T 的盘，不加并发就是纯熬工时
在命令里补上：--parallel 4(根据网络带宽和cpu核心数动态调整，千兆网用1~2，万兆网可以用4~8)

三、 排错层：建立"海难级"故障观测与全链路排查矩阵
企业实施最怕"卡死"和"未知错误"。需为前线实施工程师提供一套标准、递进的排错排查链(Troubleshooting Matrix), 故障现象比如:
1. virt-v2v启动卡在 Opening the source 超过5分钟
导致原因: esxi端为了自保，其sshd触发了高并发连接速率限制(MaxStartups)
排查: 登录esxi，修改/etc/ssh/sshd_config 中的 MaxStartups 100:30:200 和 ClientAliveInterval 60，然后 services.sh restart

2. 传输过程中突发 nbdcopy: Input/output error
导致原因: 物理链路存在坏块或闪断；目标nfs临时空间(/tmp)溢出
排查: 
1). 执行dmesg | grep -i fuse检查sshfs是否断开；
2). 检查pve根分区空间，迁移前使用export TMPDIR=/mnt/pve/nfs-shared-storage1/tmp强行更改virt-v2v的缓存目录

3. 迁移开机后卡在Blinking Cursor(光标闪烁)或找不到引导盘
导致原因: esxi的虚拟机原本是uefi固件，但pve的空壳虚拟机误选了默认的SeaBIOS
排查: 不用重新迁移！直接在pve网页端修改：虚拟机-->硬件-->添加EFI Disk，并将BIOS修改为 OVMF(UEFI)

4. 执行 virt-v2v 突发 libguestfs: error: /usr/bin/supermin ... failed
导致原因: pve宿主机的Linux内核更新后，libguestfs的底层底层沙箱(AppArmor/Access Control)权限没有同步刷新
排查: 
在PVE-Node-02的宿主机终端(root权限)一键重建底层沙箱环境: update-guestfs-appliance    # 强行重新编译并刷新微型内存沙箱的系统镜像
如依然报错则直接在PVE-Node-02终端执行前临时关闭沙箱限制: export LIBGUESTFS_BACKEND=direct
在企业生产环境或魔改系统里，系统的安全策略(AppArmor/SELinux)经常会误杀libguestfs创建的临时进程，导致抛出 supermin ... failed 的权限拒绝错误
使用 export LIBGUESTFS_BACKEND=direct 相当于给 virt-v2v 临时发了一张安全通行证，绕过系统那些层层设防的安全软件审核，让 QEMU 拥有直达特权。这通常是解决 virt-v2v 底层环境报错的终极杀手锏
原理:
当你在pve上运行virt-v2v准备去读写Rocky Linux 8.5的虚拟磁盘时，pve宿主机出于绝对的安全隔离考虑，是不允许外部程序直接把这个未经信任的镜像挂载到pve本地宿主机内核上的(防止镜像里有恶意木马利用内核漏洞攻击pve宿主机)
为了解决安全挂载的问题，libguestfs采用了以下硬核架构：
它会在pve内存里临时启动一个极其微型的、被裁剪过的linux虚拟机沙箱(这个沙箱就叫做Appliance，里面包含一个定制内核和最精简的工具集)
virt-v2v所有的解剖内核、修改配置文件、卸载vmware-tools的高危手术，全部在这个内存沙箱虚拟机里安全执行
手术做完后，沙箱自动销毁

# 先一键刷新沙箱，然后带着直连环境变量轰炸迁移
update-guestfs-appliance && LIBGUESTFS_BACKEND=direct virt-v2v -i vmx "/tmp/esxi_v2v/rocky8.5.vmx" ...


5. 转换进度条卡在Closing the overlay超过10分钟不落盘
导致原因: 目标nfs存储没有开启异步写入(async)，或底层固态硬盘在进行高并发随机写入时，触发了fuse的内核锁竞争(Lock Contention)
排查:
强行降低并发块深度并绕过fuse锁：
在命令最前面补上环境变量，强制采用串行流写入: VIRT_V2V_NO_TUNNELLING=1 virt-v2v ...

6. 迁移拉起后内核报 Kernel Panic - not syncing: VFS: Unable to mount root fs
导致原因: 目标操作系统的旧引导配置中，由于使用了特定硬件的UUID，在转为kvm架构后，系统的initramfs内核镜像没有完全将旧的引导驱动剥离
排查:
硬核原地重建内核引导镜像:
1. 用带有 Rocky 8 ISO 镜像挂载到VM 102，开机进入Rescue Mode(救援模式)
2. 强行进入chroot环境：chroot /mnt/sysimage
3. 重新为当前内核生成标准kvm驱动初始盘：dracut --force --regenerate-all
注: 或者把磁盘类型改成stat,先能进入系统后再执行上一行命令


四、兜底与反向回滚方案(企业级方案的灵魂)
没有考虑回滚(Rollback)的方案在企业里是不合格的
必须加入 [非破坏性实施准则]:
1. 绝对禁忌: 严禁在迁移前卸载esxi源端虚拟机上的VMware Tools！一旦迁移失败或遭遇未知兼容性故障，由于你破坏了原系统环境，将直接导致业务无法原地复活
virt-v2v具备影子转换特性：
它在挂载时只读取VMX/VMDK，所有的修改和VMware Tools的卸载全都是在落盘到pve的那一层目标快照/数据流里动态完成的。原esxi上的虚拟机和文件完好无损

2. 割接验证(Readiness Check):
数据传输完成后，在pve上拉起VM 102之前，必须切断VM 102的虚拟网卡断开网络(Disconnect Network)，防止两台完全一模一样的Rocky 8.5同时在局域网内跑，引发严重的IP冲突和MAC地址漂移。验证应用正常、数据无误后，关掉esxi怨种服务器，再合上pve VM 102的网络电闸
(1). 割接前的"原位不动"准则
严禁提前卸载工具：在virt-v2v成功将数据写入pve并测试通过前，绝对不允许去esxXi卸载原虚拟机的 VMware Tools 或关闭其服务
数据流只读性：virt-v2v基于sshfs或 vpx:// 读取 VMX/VMDK 时，底层全部是只读(Read-Only)挂载。所有内核注入、驱动卸载动作都发生在线转换的数据流和pve端的临时快照中
(2). 灰度割接与断网测试(防止脑裂)
断网拉起:当virt-v2v完成传输并在pve创建好VM 102后，首先点击虚拟机--->硬件--->网卡，将"已连接(Disconnect"的勾去掉
开机静默验证: 在断网状态下开启VM 102，通过pve的VNC Console登录系统，检查数据库、Web 服务、业务数据是否完好。此时原esxi上的虚拟机仍然开机在线跑着业务，整个过程对业务无任何影响
(3). 5分钟极速回滚操作
一旦在pve端的断网测试中发现核心业务应用无法启动，且在15分钟内无法定位解决，立刻触发回滚
停止pve端业务: 直接在pve上将VM 102强制关机(Stop)
清理脏数据： 执行 qm destroy 102 --destroy-unreferenced-disks 瞬间格式化pve端转换失败的虚拟盘，不占用nfs存储空间
确认源端状态： 检查esxi上的虚拟机状态。因为源端一直处于只读状态且保持开机，局域网网络流量完全没有发生过中断
宣告回滚成功： 割接失败，业务零受损，回滚总耗时不超过1分钟





```









# NFS-Storage节点
```shell
rambo@debian2:~$ cat /etc/apt/sources.list
deb https://mirrors.aliyun.com/debian/ bookworm main non-free non-free-firmware contrib
deb-src https://mirrors.aliyun.com/debian/ bookworm main non-free non-free-firmware contrib
deb https://mirrors.aliyun.com/debian-security/ bookworm-security main
deb-src https://mirrors.aliyun.com/debian-security/ bookworm-security main
deb https://mirrors.aliyun.com/debian/ bookworm-updates main non-free non-free-firmware contrib
deb-src https://mirrors.aliyun.com/debian/ bookworm-updates main non-free non-free-firmware contrib
deb https://mirrors.aliyun.com/debian/ bookworm-backports main non-free non-free-firmware contrib
deb-src https://mirrors.aliyun.com/debian/ bookworm-backports main non-free non-free-firmware contrib
#zhongkeda
deb https://mirrors.ustc.edu.cn/debian/ bookworm main non-free non-free-firmware contrib
deb-src https://mirrors.ustc.edu.cn/debian/ bookworm main non-free non-free-firmware contrib
deb https://mirrors.ustc.edu.cn/debian-security/ bookworm-security main
deb-src https://mirrors.ustc.edu.cn/debian-security/ bookworm-security main
deb https://mirrors.ustc.edu.cn/debian/ bookworm-updates main non-free non-free-firmware contrib
deb-src https://mirrors.ustc.edu.cn/debian/ bookworm-updates main non-free non-free-firmware contrib
deb https://mirrors.ustc.edu.cn/debian/ bookworm-backports main non-free non-free-firmware contrib
deb-src https://mirrors.ustc.edu.cn/debian/ bookworm-backports main non-free non-free-firmware contrib


rambo@debian2:~$ sudo apt update && sudo apt install nfs-kernel-server -y




如果新添加了磁盘又不想reboot让机器能识别到, 则需要在线扫描新磁盘(核心), 这是企业里最常用的方法
echo "- - -" | sudo tee /sys/class/scsi_host/host*/scan
fdisk -l 应该就能识别到，如还没出则：
sudo partprobe 或 sudo udevadm trigger


rambo@debian2:~$ sudo lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0  200G  0 disk 
sdb      8:16   0   80G  0 disk 
├─sdb1   8:17   0  953M  0 part /boot
├─sdb2   8:18   0  1.9G  0 part [SWAP]
└─sdb3   8:19   0 77.2G  0 part /

rambo@debian2:~$ sudo fdisk /dev/sda

Welcome to fdisk (util-linux 2.38.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS (MBR) disklabel with disk identifier 0x175d75fa.

Command (m for help): g
Created a new GPT disklabel (GUID: BAD7B077-4EAB-8146-BA04-9E8C196874B2).

Command (m for help): n
Partition number (1-128, default 1): 
First sector (2048-419430366, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-419430366, default 419428351): 

Created a new partition 1 of type 'Linux filesystem' and of size 200 GiB.

Command (m for help): w


rambo@debian2:~$ sudo mkfs.ext4 /dev/sda1


# 创建用于存放虚拟机镜像的目录
rambo@debian2:~$ sudo mkdir -p /mnt/nfs_shares/pve_storage

rambo@debian2:~$ sudo mount /dev/sda1 /mnt/nfs_shares/pve_storage

# 更改权限，让所有匿名用户都有读写权
rambo@debian2:~$ 
sudo chown -R nobody:nogroup /mnt/nfs_shares/pve_storage
sudo chmod 777 /mnt/nfs_shares/pve_storage

rambo@debian2:~$ sudo blkid /dev/sda1
/dev/sda1: UUID="ad8455e4-54fc-4a27-b407-f68914159a00" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="ab0fc1ad-06c2-f448-9cca-f2e785627ebb"

rambo@debian2:~$ echo 'UUID=ad8455e4-54fc-4a27-b407-f68914159a00  /mnt/nfs_shares/pve_storage   ext4   defaults   0  0' | sudo tee -a /etc/fstab

rambo@debian2:~$ sudo systemctl daemon-reload
rambo@debian2:~$ sudo mount -a
rambo@debian2:~$ df -Th | grep sda1
/dev/sda1      ext4      196G   28K  186G   1% /mnt/nfs_shares/pve_storage


编辑/etc/exports配置文件(避坑核心)
rambo@debian2:~$ sudo vim /etc/exports 
/mnt/nfs_shares/pve_storage 192.168.2.0/24(rw,sync,no_subtree_check,no_root_squash,anonuid=65534,anongid=65534)

# 启动并激活服务
rambo@debian2:~$ sudo systemctl restart nfs-kernel-server && sudo systemctl enable nfs-kernel-server

rambo@debian2:~$ sudo showmount -e 192.168.2.113
Export list for 192.168.2.113:
/mnt/nfs_shares/pve_storage 192.168.2.0/24


```







# 2个pve设置
```shell
# 默认web登录用户是root，这里2个密码都是8a

# 修改IP
root@pve:~# nano /etc/network/interfaces
auto lo
iface lo inet loopback
iface ens33 inet manual
source /etc/network/interfaces.d/*
auto vmbr0
iface vmbr0 inet static
        address 192.168.2.121/24
        gateway 192.168.2.1
        bridge-ports ens33
        bridge-stp off
        bridge-fd 0

root@pve:~# cat /etc/hosts
127.0.0.1 localhost.localdomain localhost
192.168.2.142 pve.localdomain pve

# The following lines are desirable for IPv6 capable hosts

::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts

root@pve:~# cat /etc/issue
------------------------------------------------------------------------------
Welcome to the Proxmox Virtual Environment. Please use your web browser to
configure this server - connect to:
  https://192.168.2.121:8006/
------------------------------------------------------------------------------

root@pve:~# reboot




# 同步时间
root@pve:~# timedatectl set-timezone Asia/Shanghai
root@pve:~# timedatectl set-ntp true
root@pve:~# systemctl restart systemd-timesyncd              # 如报错则说明你用的是chrony，执行下一行命令
root@pve:~# systemctl restart chrony


# 安装工具包
1、替换企业源为免订阅源
root@pve:~# mv /etc/apt/sources.list.d/{ceph.list,pve-enterprise.list} .

2、添加免费的国内免订阅源(以清华大学镜像源为例，2026年依然稳定高效)
root@pve:~# 
echo "deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm main contributor non-free non-free-firmware" > /etc/apt/sources.list
echo "deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-updates main contributor non-free non-free-firmware" >> /etc/apt/sources.list
echo "deb https://mirrors.tuna.tsinghua.edu.cn/debian-security bookworm-security main contributor non-free non-free-firmware" >> /etc/apt/sources.list

# 添加 PVE 8 的免费非企业源
root@pve:~# echo "deb https://mirrors.tuna.tsinghua.edu.cn/proxmox/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list

root@pve:~# apt update && apt install vim wget curl net-tools -y



# 非正常修改登录密码
开机，按Ctrl+X 进入到grub模式(蓝色画面)不要按任何键，直接按e键
用上下键把光标移到 linux /boot/vmlinuz-.... quiet 空格 init=/bin/bash 输好后，按Ctrl+X保存退出
然后回重启，输入mount -rw -o remount /  回车
输入passwd回车修改root密码，如果要换其它用户密码则是passwd zhangsan 回车



正常虚拟机存放应该在 /vmfs/volumes/datastore-name/虚拟机名字/

```




# 在2个PVE节点上挂载此NFS存储
![image](./images/12.png)
![image](./images/13.png)
```shell
如是第二次挂载则需要
1. 强制解除可能存在的死锁挂载
umount -f -l /mnt/pve/nfs-shared-storage
(注：-f 是强制，-l 是懒惰卸载，即使连接卡死也能瞬间剥离)

2. 清理残留的空目录(按需)
PVE 在激活存储时如果发现本地目录没有清理干净或状态不对会拒绝执行，直接删掉这个占位目录：
rm -rf /mnt/pve/nfs-shared-storage
重启 PVE 的核心管理服务（关键）
这是最重要的一步，让 PVE 刷新它的存储状态机：
systemctl restart pvedaemon pveproxy pvestatd

如果是第二次或多次导致的nfs挂载不上，nfs在名字和路径等都相同的情况下则需要重启下esxi(玄学问题)

```
![image](./images/14.png)



# 在ESXi8.0上挂载此NFS存储(为迁移做准备)
![image](./images/15.png)
![image](./images/16.png)
![image](./images/17.png)
![image](./images/18.png)

```shell
用以下方法验证是否合格:
在ESXi的ESXi-NFS-Share里随便上传一个ISO
登录PVE-Node-01的终端，查看ls /mnt/pve/nfs-shared-storage/template/iso/。如果你能在PVE里面看到刚刚在ESXi里上传的那个iso文件，说明两家虚拟化巨头已经共用了一个物理心脏
```
![image](./images/19.png)
![image](./images/20.png)



# 在esxi中装2个linux系统(BIOS格式和EFI格式, 为迁移做准备)
```shell
在ESXi里面塞一个测试用的Linux小虚拟机，准备演练大厂最怕、开源最爱的 ESXi → PVE抽干迁移了

⚠️  大坑预警(90%的报错都在这里)
很多新手做实验，在ESXi里安装Linux时，磁盘控制器选的是普通的SATA或IDE。这种迁移到PVE闭着眼睛都能开机，没有任何技术含量
真实的企业VMware环境里，为了追求性能，虚拟机的磁盘控制器100%都是 VMware Paravirtual (PVSCSI)(VMware准虚拟化SCSI网卡/磁盘驱动)

应该怎么做：
在ESXi里面创建虚拟机时，故意将磁盘控制器选为 VMware Paravirtual 或 LSI Logic SAS
将网卡选为VMXNET3
正常安装完Linux系统，并在里面随便写几个文件(比如在 /root/test.txt 里写点字)，证明数据存在




# ============================== 如你和我的环境不同则这里可跳过 =====================================================
# 这里有个很关键的问题，就是我的宿主机是Linux mint,在mint的基础上安装的vmware workstation17pro, 在vmware workstation17pro里面安装了esxi，在esxi中安装linux时因为在ESXi8.0嵌套虚拟化(Nested)环境下，桥接模式安装linux报错连接错误，100%是卡在"网络混杂模式(Promiscuous Mode)" 和 "MAC地址安全策略"上。 如果你和我的环境不同则这里可跳过

# 第一步：修改最外层 VMware Workstation 的虚拟网卡权限(关键)
因为宿主机是 Linux Mint，VMware Workstation 在 Linux 下对桥接网卡的混杂模式有严格的系统权限限制

1、彻底关闭两台物理机上的 VMware Workstation(这里我没关)

2、打开 Linux Mint 的终端，放开网卡的混杂模式权限(假设你的物理网卡叫 eth0 或 enp3s0)
# 给 VMware 的虚拟网卡设备赋予读写权限
rambo@lab:~$ ls -alh /dev/vmnet*
crw------- 1 root root 119, 0  5月 23 05:19 /dev/vmnet0
crw------- 1 root root 119, 1  5月 23 05:20 /dev/vmnet1
crw------- 1 root root 119, 2  5月 23 05:20 /dev/vmnet2
crw------- 1 root root 119, 8  5月 23 05:20 /dev/vmnet8

rambo@lab:~$ sudo chmod a+rw /dev/vmnet*

rambo@lab:~$ ls -alh /dev/vmnet*
crw-rw-rw- 1 root root 119, 0  5月 23 05:19 /dev/vmnet0
crw-rw-rw- 1 root root 119, 1  5月 23 05:20 /dev/vmnet1
crw-rw-rw- 1 root root 119, 2  5月 23 05:20 /dev/vmnet2
crw-rw-rw- 1 root root 119, 8  5月 23 05:20 /dev/vmnet8

3、重新打开 VMware Workstation，确保 ESXi 虚拟机的网卡设置里桥接模式(Bridged)下勾选了"复制物理网络连接状态"(Replicate physical network connection state)

# 第二步: 修改ESXi8.0内部的虚拟交换机安全策略(致命核心)
即使外层放开了，ESXi8.0默认的虚拟交换机(vSwitch)安全策略也是极度严格的，它默认丢弃所有不是ESXi自身发出的流量
1. 登录ESXi8.0 Web后台
2. 点击左侧菜单的Networking(网络) --> 选择Virtual switches(虚拟交换机)标签页
3. 点击默认的vSwitch0，然后点击Edit settings(编辑设置)
4. 展开Security(安全)折叠菜单，将以下三项默认的Reject(拒绝) 全部改为 Accept(接受)
       Promiscuous mode(混杂模式) --> Accept
       MAC address changes(MAC地址更改) --> Accept
       Forged transmits(伪标传输) --> Accept
5. 点击Save(保存)

⚠️  注意：如果你使用了端口组(Port groups)，请点击Port groups标签页，对VM Network同样点击编辑设置，确保它的安全策略也继承了上述的Accept(接受)或者是单独改成了Accept

# 第三步：在Ubuntu 24.04 安装界面重试(这里我直接把要安装的虚拟机关机后重启后网络就都通了,如你的未通则继续以下步骤)
做完上述两步后，底层的网络管道才真正打通：
1.  在 Ubuntu 24.04 的安装界面，选中网络配置(Network Configuration)那一步
2.  如果它还是显示错误，选择那张网卡，点击 **Edit IPv4**，手动把它从 `DHCP` 改为 `Manual`（静态 IP），输入：
       Subnet: 192.168.2.0/24
       Address: 192.168.2.xx (给Ubuntu规划的IP)
       Gateway: 192.168.2.1 (你家路由器的网关，如果没有路由器直连，可以写我Mint的IP)
       Name servers: 8.8.8.8 或留空
3.  保存后点击 Rescan或者是直接下一步



```




# 迁移Linux
## 迁移前奏
```shell
不同的客户情况不同，必须掌握以下两种手段：
方法1: 用PVE8.x官方原生API抽干(首选)
这是 Proxmox 官方近年专门为了应对 VMware 涨价大逃亡开发的"黑科技"。它不需要你登录ESXi 去导文件，PVE 自己会通过 API 去 ESXi 里面"偷"数据
1、在 PVE 界面配置集成： 登录 PVE-Node-02 (P360) 的 Web 后台。点击 Datacenter -> Storage -> Add -> 选择 ESXi
2、连接旧世界: 输入你ESXi的IP(192.168.1.22)、用户名(root)和密码
3、一键拉取: 此时，PVE的左侧菜单会直接多出一个 ESXi 的图标，点开它，你能直接肉眼看到你在ESXi里面运行的那台Linux虚拟机
4、在线导入: 选中它，点击Import(导入)，目标存储选择我们刚刚搭好的 nfs-shared-storage，网络桥接选择 PVE 的本地网卡。点击开始
5、底层逻辑: PVE会在后台自动调用 ovftool 或者内置转换流，把 VMware 的 vmdk 磁盘切片，一边拉取一边直接转换成 PVE 识别的格式，并自动注入 VirtIO 驱动


方法2: 硬核命令行转换 --- 离线冷迁移(突发状况救命用)
如果客户的ESXi版本太旧(比如ESXi 5.5/6.0)，或者网络有防火墙，PVE官方工具连不上，就必须用底层命令行解决：
1、在ESXi里把这台 Linux 虚拟机关机
2、通过 ESXi 的 Datastore 浏览器，把该虚拟机的 .vmdk 文件直接下载到你的宿主机(我的是Linux Mint)，或者通过scp直接传到 PVE-Node-02 的 /root/ 目录下
3、在PVE命令行执行硬核转换：
     qemu-img convert -f vmdk -O qcow2 linux-vm.vmdk  linux-vm.qcow2      # 转换后的就是pve要用的镜像
4. 在PVE里新建一个空虚拟机，然后用命令行把这个 `qcow2` 磁盘强行塞进去：
   qm importdisk <新VM_ID>  linux-vm.qcow2  nfs-shared-storage


方法3: 真正的"原地转生" —— 共享存储零拷贝(推荐，耗时0秒)
如果客户的 VMware 原本就挂载了NAS、SAN或者是像 NFS 这种共享存储，则根本不需要拷贝这1TB的文件
核心步骤：
让pve节点直接通过网络挂载客户现有的VMware存储卷(比如直接挂载同一个nfs目录)
在pve终端，直接用qemu-img读取原 .vmdk，并把转换后的数据直接写入pve自己的本地高速度存储(如 local-lvm 或 Ceph)
或者更绝：kvm本身就原生支持直接读取 .vmdk 格式。可直接在pve里建一个虚拟机，配置文件直接指向那个1TB的.vmdk文件。开机直接用
等后续有空了再在后台在线将磁盘格式转换(Storage Migration)为 qcow2 或 raw。整个迁移过程，应用停机时间不超过5分钟
因为本次我的esxi和pve挂载了同一个Debian NFS存储：
登录pve的终端，进入 /mnt/pve/nfs-shared-storage/ub24-1/ 就能看到40G大小的ub24-1.vmdk
直接在pve终端执行本地转换，把它转到pve本地的local-lvm存储里，或直接在nfs目录下就地转换



方法4: 管道级 --- 边传、边转、边写
如果VMware 和 PVE 处于不同的物理机房，磁盘必须跨网络传输则绝对不能先用scp下到本地再转
要用Linux管道(pipe)结合ssh和qemu-img，让数据变成流。在pve终端执行一行命令：
ssh root@esxi-ip "cat /vmfs/volumes/datastore1/vm/disk.vmdk" | qemu-img convert -f vmdk -O qcow2 /dev/stdin /var/lib/vz/images/101/vm-101-disk-0.qcow2
底层原理：
ssh ... cat：从远程esxi物理机上把1TB的文件读出来后直接吐到网络管道里
|（管道）：在内存中接住网络传过来的数据
/dev/stdin：qemu-img不去读硬盘文件，而是直接守在内存门口(标准输入)，来1MB数据就地转换1MB然后直接写入pve的最终目标硬盘里 
效果： 整个过程不需要任何中转空间，两台机器的硬盘都在全速读写，速度完全取决于你的网络带宽(万兆网络下1TB也就十几分钟) 


方法5: 生产环境终极降噪 --- 用 virt-v2v自动接管
virt-v2v是红帽官方主导的虚拟化迁移工具
在pve上安装好 virt-v2v 后，只需要给它esxi的密码，pve的路径即可，它会自己登录esxi，通过网络流把数据拉过来，在内存里完成 vmdk 到 qcow2 的转换
最重要的是：它会自动分析这个1TB磁盘里的系统，自动卸载里面的VMware Tools，自动安装KVM的 VirtIO驱动，最后在pve把虚拟机创建好



不管用哪个方法，当你把虚拟机迁移到PVE之后，点击Start(开机)
满分结果: 虚拟机顺利看到Grub引导菜单，顺利进入 Linux 登录界面，输入密码进去，/root/test.txt 文件完好无损

翻车结果(企业迁移最常见故障): 屏幕卡在 Gave up waiting for root file system device 或者蓝屏/黑屏，提示找不到硬盘
这就是因为 VMware 的 PVSCSI 驱动到了KVM架构下失效了


```






## 方法1: 用官方原生工具
### EFI格式
![image](./images/21.png)
```shell
# esxi8上的vm (ubuntu24.10)的源
rambo@test1:~$ sudo nano /etc/apt/sources.list
deb http://old-releases.ubuntu.com/ubuntu/ oracular main restricted universe multiverse
deb http://old-releases.ubuntu.com/ubuntu/ oracular-updates main restricted universe multiverse
deb http://old-releases.ubuntu.com/ubuntu/ oracular-security main restricted universe multiverse
deb http://old-releases.ubuntu.com/ubuntu/ oracular-backports main restricted universe multiverse


sudo apt update && sudo apt install vim wget curl net-tools openssh-client openssh-server -y
rambo@test1:~$ sudo systemctl enable --now ssh

# 是efi模式
rambo@test1:~$ ls /boot/efi/
EFI
rambo@test1:~$ ls /boot/efi/EFI/
BOOT  ubuntu


# 模拟数据
rambo@test1:~$ vim test.txt
i don't know
i'm sorry
yes, it's
good morning


# 迁移
esxi8上要迁移的vm必须先关机
```
![image](./images/23.png)
![image](./images/24.png)
![image](./images/25.png)
![image](./images/26.png)
```shell
点击import后的底层逻辑: PVE会在后台自动调用ovftool或者内置转换流，把VMware的vmdk磁盘切片，一边拉取一边直接转换成PVE识别的格式，并自动注入VirtIO驱动

```

![image](./images/27.png)
```shell
# ===================== vm不关机可能会报如下错 =============================
  Rounding up size to full physical extent 4.00 MiB
  Logical volume "vm-100-disk-0" created.
transferred 0.0 B of 128.0 KiB (0.00%)
transferred 128.0 KiB of 128.0 KiB (100.00%)
transferred 128.0 KiB of 128.0 KiB (100.00%)
efidisk0: successfully created disk 'local-lvm:vm-100-disk-0,size=4M'
create full clone of drive (oldesxi8:ha-datacenter/ESXi-NFS-Share/ub24-1/ub24-1.vmdk)
  Logical volume "vm-100-disk-1" created.
transferred 0.0 B of 40.0 GiB (0.00%)
qemu-img: error while reading at byte 0: Input/output error
  Logical volume "vm-100-disk-0" successfully removed.
  Logical volume "vm-100-disk-1" successfully removed.
TASK ERROR: unable to create VM 100 - cannot import from 'oldesxi8:ha-datacenter/ESXi-NFS-Share/ub24-1/ub24-1.vmdk' - copy failed: command '/usr/bin/qemu-img convert -p -n -f vmdk -O raw /run/pve/import/esxi/oldesxi8/mnt/ha-datacenter/ESXi-NFS-Share/ub24-1/ub24-1.vmdk zeroinit:/dev/pve/vm-100-disk-1' failed: exit code 1
# ===========================================================================
```
![image](./images/28.png)
![image](./images/29.png)
![image](./images/30.png)
![image](./images/31.png)
![image](./images/32.png)
![image](./images/33.png)
```shell
# 原因分析
在ESXi里这台虚拟机网卡用的是vmxnet3，还记得吧
但在PVE/KVM里默认变成VirtIO、Intel E1000
所以Linux内核会发现原来的网卡没了，于是原来ens160现在可能变成ens18或者eth0等等
所以系统启动后网络配置还在找ens160,但系统实际只有ens18所以网络直接失效
这是 VMware 迁移里最经典的问题

以下项目企业里迁移时全都会遇到这种情况
VMware
OpenStack
KVM
公有云


# 修复
注: 下面的问题应该是我有挂clash verge导致的，把clash关闭后重启了vmware后就都好了
```
![image](./images/34.png)
![image](./images/35.png)
![image](./images/36.png)
![image](./images/37.png)
![image](./images/38.png)
![image](./images/39.png)



### BIOS格式
<font color=red>**在ESXI8上安装系统**</font>
![image](./images/40.png)
![image](./images/41.png)

```shell
# 需要自己配置IP
[rambo@192 ~]$ sudo nmcli con mod ens192 \
ipv4.addresses 192.168.2.146/24 \
ipv4.gateway 192.168.2.1 \
ipv4.dns 192.168.2.1 \
ipv4.method manual

[rambo@192 ~]$ sudo nmcli con up ens192

[rambo@192 ~]$ ip a 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:0c:29:3e:99:5d brd ff:ff:ff:ff:ff:ff
    inet 192.168.2.146/24 brd 192.168.2.255 scope global noprefixroute ens192
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe3e:995d/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 52:54:00:47:cc:86 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
4: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc fq_codel master virbr0 state DOWN group default qlen 1000
    link/ether 52:54:00:47:cc:86 brd ff:ff:ff:ff:ff:ff


[rambo@192 ~]$ ping -c3 qq.com
PING qq.com (123.150.76.218) 56(84) bytes of data.
64 bytes from 123.150.76.218 (123.150.76.218): icmp_seq=1 ttl=52 time=18.10 ms
64 bytes from 123.150.76.218 (123.150.76.218): icmp_seq=2 ttl=52 time=19.0 ms
64 bytes from 123.150.76.218 (123.150.76.218): icmp_seq=3 ttl=52 time=19.4 ms

--- qq.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 18.977/19.124/19.394/0.221 ms

```


#### 迁移方法
![image](./images/42.png)
```shell
在esxi8上下载rocky8的.vmdk文件时会出现下载2个文件，一个是rocky8.5.vmdk(525B大小)，一个是rocky8.5-flat.vmdk(40G大小)

VMware磁盘体系:
descriptor.vmdk 是元数据
flat.vmdk 是真实数据
delta.vmdk 是snapshot 差异
sesparse.vmdk 是快照扩展

虚拟化存储本质是 metadata + data separation

迁移时最容易踩坑(非常重要)
很多人只拷贝了flat.vmdk
然后VMware/PVE不认, 因为缺descriptor

正确做法企业标准)是两个都要
如果在做 VMware -> PVE，通常只导入 descriptor.vmdk 即可
例如PVE：
qm importdisk 100 rocky8.5.vmdk local-lvm             # descriptor会自动找到flat文件


rocky8.5.vmdk(525B)这是descriptor file(描述文件)里面保存以下信息,几乎不存数据,所以只有几百字节很正常
磁盘类型
geometry
adapter type
指向 flat 文件

descriptor 文件里有什么？
cat rocky8.5.vmdk                  # 可能会看到以下内容
RW 83886080 VMFS "rocky8.5-flat.vmdk"
注: 意思是rocky8.5-flat.vmdk这个虚拟盘真正的数据


rocky8.5-flat.vmdk(40G)这才是真正的虚拟磁盘数据都在这里
系统：
    OS
    文件
    分区
    数据


可理解成：
rocky8.5.vmdk
    ↓
“索引文件”

rocky8.5-flat.vmdk
    ↓
真正磁盘


如果只剩 flat.vmdk 怎么办？(很值钱)真实企业经常碰到的事故
可以手工重建descriptor,例如执行下面的命令然后替换flat:
vmkfstools -c 40G -d thin temp.vmdk

企业里最常见的是 "存储炸了" 最后只剩 -flat.vmdk 此时会修 descriptor 的人很少



```



> 官方API导入虽然方便但会有很多坑,比如：
网卡残留
VMware MAC
cloud-init
VirtIO
DHCP
BIOS/EFI
>
而手工迁移才是真正企业里需要的能力





## 方法2: 用命令行转换
```shell
如果客户的esxi版本太旧(比如esxi5.5/6.0)，或网络有防火墙，PVE官方工具连不上，就必须用命令行解决,这个实验我还用esxi8来做：
1、在esxi里把这台Linux虚拟机关机
2、通过ESXi的Datastore浏览器，把该虚拟机的.vmdk文件直接下载到宿主机，或通过scp直接传到PVE-Node-02的/root/目录下
3、在PVE命令行执行硬核转换：
	qemu-img convert -f vmdk -O qcow2 linux-vm.vmdk linux-vm.qcow2       # 转换后的就是pve要用的镜像
注：
企业里通常更常用(推荐)
qm importdisk 100 xxx.vmdk local-lvm --format raw      或 --format qcow2
raw的优点是性能最好,尤其：
Ceph
LVM-thin
ZFS zvol

而qcow2的缺点是性能损耗，优点:
snapshot
thin
compression


4. 在pve里新建一个空虚拟机，然后用命令行把这个qcow2 磁盘强行塞进去：
   	qm importdisk <新VM_ID> linux-vm.qcow2 nfs-shared-storage

qm importdisk 100 rocky8.5.vmdk local-lvm
qm是Proxmox  管理命令，是PVE最核心的迁移命令之一，类是kvm的virsh,企业里大量都会用：
VMware → PVE
qcow2 → raw
vmdk → lvm-thin

导入后pvw会：
    1) 读取vmdk
    2) 转换格式，通常是raw格式
    3) 写入 local-lvm
    4) 生成：vm-100-disk-0

导入后并不会自动挂载还需进入vm -> Hardware 把 Unused Disk 挂到SCSI、virtio、stat

#==================================================================
逐段拆解
这条命令本质上是在做VMware 磁盘 → Proxmox 存储池的格式转换与导入
importdisk意思是导入虚拟磁盘
本质是转换格式 + 写入 PVE storage
底层通常调用qemu-img

100是vm id
注意(关键), 这里vm必须已经存在
例如你已经创建了一个空vm,其id=100，但不需要硬盘

rocky8.5.vmdk是VMware descriptor文件
PVE会自动找到rocky8.5-flat.vmdk并读取真实数据
如果写成rocky8.5-flat.vmdk通常会失败，因为flat本身缺metadata

local-lvm是pve存储池名字
查看PVE存储池:
数据中心(Datacenter) -> 存储(Storage)
会看到：
local
local-lvm      # 适合生产、高性能、正是运行的vm
nfs-storage    # nfs适合备份、中专、iso、临时迁移等
ceph
# ==================================================================

企业里通常更常用(推荐)
qm importdisk 100 xxx.vmdk local-lvm --format raw      或 --format qcow2
raw的优点是性能最好
尤其：
Ceph
LVM-thin
ZFS zvol

而qcow2的缺点是性能损耗，优点:
snapshot
thin
compression



# 流程总结: 适合小项目
第一步 在esxi中关闭vm
第二步 下载.vmdk、-flat.vmdk
第三步 pve创建空vm(直接删除默认磁盘,cpu/mem正常设置),例如id 101
    注意：
	BIOS/UEFI 要一致
	Machine type 尽量一致
	SCSI Controller推荐VirtIO SCSI



企业真实迁移第一原则先启动, 不是先高性能


```
### bios格式
![image](./images/43.png)
![image](./images/44.png)
![image](./images/45.png)
![image](./images/46.png)
![image](./images/47.png)
![image](./images/48.png)
![image](./images/49.png)
![image](./images/50.png)

```shell
第四步 进到.vmdk和-flat.vmdk所在位置并开始转换
第五步(核心)
root@pve:~# cd /mnt/pve/nfs-shared-storage1/rocky8.5/
root@pve:/mnt/pve/nfs-shared-storage1/rocky8.5# ls -alh *.vmdk
-rw------- 1 root root 40G May 24 15:52 rocky8.5-flat.vmdk
-rw------- 1 root root 528 May 24 15:38 rocky8.5.vmdk

# 用pve官方推荐方式，导入到nfs的挂载上，如果nfs/local-lvm空间不够执行以下命令时会提示，务必注意
root@pve:/mnt/pve/nfs-shared-storage1/rocky8.5# qm importdisk 101 rocky8.5.vmdk  nfs-shared-storage1
importing disk 'rocky8.5.vmdk' to VM 101 ...
Formatting '/mnt/pve/nfs-shared-storage1/images/101/vm-101-disk-0.raw', fmt=raw size=42949672960 preallocation=off
transferred 0.0 B of 40.0 GiB (0.00%)
transferred 413.7 MiB of 40.0 GiB (1.01%)
transferred 823.3 MiB of 40.0 GiB (2.01%)
transferred 1.2 GiB of 40.0 GiB (3.04%)
....
    ....
transferred 39.5 GiB of 40.0 GiB (98.82%)
transferred 39.9 GiB of 40.0 GiB (99.84%)
transferred 40.0 GiB of 40.0 GiB (100.00%)
transferred 40.0 GiB of 40.0 GiB (100.00%)
unused0: successfully imported disk 'nfs-shared-storage1:101/vm-101-disk-0.raw'



第六步 在pve GUI中点击vm -> Hardware双击Unused Disk修改下面的项(一般默认不用改)
    Bus/Device：SCSI
    Controller：VirtIO SCSI
    点击右下角的 "添加"即可
第七步 添加virtio 网卡
第八步 进入系统删除 VMware tools,并安装 qemu-guest-agent
第九步 修复netplan、grub、initramfs



# 下图中要把高级勾选后"OK"才会变成"添加"
```
![image](./images/51.png)
![image](./images/52.png)
```shell
如果下图改stat后也必须同步改boot order,否则BIOS可能找不到启动盘
```
![image](./images/53.png)
![image](./images/54.png)
![image](./images/55.png)
![image](./images/56.png)
![image](./images/57.png)
```shell
# 下图还需设置开机自启动
nmcli con mod ens18 connection.autoconnect yes
```
![image](./images/58.png)
![image](./images/59.png)

```shell
# 稳定后再优化
把网络模型改成 VirtIO

把Disk 改成 VirtIO SCSI会起不来(符合企业迁移的真实情况,先在SATA模式正常启动)，如果你要改，务必记得改后还要改引导
VirtIO更强是因为它不是模拟真实硬件, 而是半虚拟化驱动,性能极高

每改一次启动看一下，如下图先改网络的模型

```

![image](./images/60.png)
![image](./images/61.png)
```shell
下图改完后务必记得还要  改引导 !!!
```
![image](./images/62.png)
![image](./images/63.png)




### efi格式(略)
```shell
efi格式就不测试了,一般也用不到这种转换的方式
EFI迁移的本质也是这样,但EFI比BIOS更复杂, 因为EFI除了rootfs还有EFI System Partition(ESP), 例如：/boot/efi
所以EFI VM迁移时必须：
1）PVE也使用OVMF, 即BIOS = OVMF
2）Machine type 尽量一致, 推荐q35
3）必须添加 EFI Disk

PVE: Add -> EFI Disk否则经常找不到 EFI bootloader
企业里不要混,应该:
BIOS VM 最好迁 BIOS
EFI VM 最好迁 EFI

```






# 方法3和方法2大致相同(略)






# 方法4 管道级实时---半废
```shell
第三步：在pve上建好空壳，拿到最终写入路径
1. 登录 PVE-Node-02 (P360) 的Web后台
2.点击 Create VM（创建虚拟机）：
	General：设置虚拟机ID(例如102)
	OS：选择 "Do not use any media"(不使用任何介质)
	System：保持默认
	Disks：直接把自带的硬盘删掉(点那个垃圾桶图标)。我们需要一个完全没有硬盘的空壳
	一路下一步直到完成
3. 此时，PVE 会在它的本地逻辑卷(local-lvm)中自动为你预留位置，但还没分配实际块


第四步：执行管道级流式传输(核心动作)
# 在宿机上通过配置文件找到vm的物理路径
rambo@e8bit:~$ ssh root@192.168.2.122 "vim-cmd vmsvc/getallvms"
(root@192.168.2.122) Password: 
Vmid     Name                      File                        Guest OS       Version   Annotation
5      rocky8.5   [ESXI-NFS-Share1] rocky8.5/rocky8.5.vmx   centos7_64Guest   vmx-21 

rambo@e8bit:~$ ssh root@192.168.2.122 "ls -alh /vmfs/volumes/"
(root@192.168.2.122) Password: 
total 1548
drwxr-xr-x    1 root     root         512 May 24 17:10 .
drwxr-xr-x    1 root     root         512 May 24 07:36 ..
drwxr-xr-x    1 root     root           8 Jan  1  1970 26ebbad0-04481ffe-1094-c6d5b5452ea0
drwxr-xr-t    1 root     root       76.0K May 23 07:25 6a11563f-e8acc822-7f9f-000c29d7ea74
lrwxr-xr-x    1 root     root          35 May 24 17:10 BOOTBANK1 -> 26ebbad0-04481ffe-1094-c6d5b5452ea0
lrwxr-xr-x    1 root     root          35 May 24 17:10 BOOTBANK2 -> bc6b9068-4e474bd7-f9bf-ca916110a32c
lrwxr-xr-x    1 root     root          17 May 24 17:10 ESXI-NFS-Share1 -> eca5bd35-3a696585          # 查出来的vmdk文件在这里
lrwxr-xr-x    1 root     root          17 May 24 17:10 ESXi-NFS-Share -> c2d6a58c-fb7a181e
lrwxr-xr-x    1 root     root          35 May 24 17:10 OSDATA-6a11563f-e8acc822-7f9f-000c29d7ea74 -> 6a11563f-e8acc822-7f9f-000c29d7ea74
drwxr-xr-x    1 root     root           8 Jan  1  1970 bc6b9068-4e474bd7-f9bf-ca916110a32c
drwxrwxrwx    5 65534    65534       4.0K May 24 02:19 c2d6a58c-fb7a181e
drwxr-xr-x    6 root     root        4.0K May 24 13:03 eca5bd35-3a696585                             # 这个是实际的目录


# 先确认pve上的存储空间
root@pve:~# df -Th
Filesystem                                 Type      Size  Used Avail Use% Mounted on
udev                                       devtmpfs   16G     0   16G   0% /dev
tmpfs                                      tmpfs     3.2G  1.8M  3.2G   1% /run
/dev/mapper/pve-root                       ext4       35G  3.1G   30G  10% /
tmpfs                                      tmpfs      16G   46M   16G   1% /dev/shm
tmpfs                                      tmpfs     5.0M     0  5.0M   0% /run/lock
/dev/fuse                                  fuse      128M   20K  128M   1% /etc/pve
tmpfs                                      tmpfs     3.2G     0  3.2G   0% /run/user/0
192.168.2.113:/mnt/nfs_shares/pve_storage  nfs4       20G   10G  8.6G  54% /mnt/pve/nfs-shared-storage
192.168.2.113:/mnt/nfs_shares/pve_storage1 nfs4       59G   12G   45G  21% /mnt/pve/nfs-shared-storage1     # 还够存放40G的镜像


```






# 方法5 用virt-v2v自动接管
## 迁移BIOS格式
```shell
在2026年的生产环境下, 面对 1TB~10TB 的超大虚拟机, 真正的大厂级开源标准方案就是 virt-v2v (Virtualization Vector). 它是红帽官方主导并深度维护的虚拟化迁移工具, 专门用来做企业级的大逃亡.

virt-v2v的第一性原理: 它不仅做数据流式传输, 更核心的是它自带一个"OS医生" (virt-p2v/v2v-crypto), 能在数据传输的同时, 自动解剖目标盘的内核、卸载 VMware Tools、注入 KVM 的 VirtIO 驱动、重构 initramfs. 真正做到一键接管.

以下是 2026 年最稳固、最适合你当前 PVE-Node-02 (192.168.2.x) 环境的 virt-v2v自动接管实战全流程

第一步: 在pve上准备"武器库"
virt-v2v需要一系列底层工具(如Guestfish虚拟文件系统工具)的支持来解剖Linux磁盘.
在 PVE-Node-02 终端执行以下命令安装核心组件:
root@pve:~# apt-get update && apt-get install -y virt-v2v libguestfs-tools sshfs nbdkit libnbd-bin
virt-v2v在准备向你的NFS写入最终数据时，调用了Linux底层的 NBD(Network Block Device，网络块设备kit) 组件
新版的virt-v2v极度依赖nbdkit来做高效率的流式数据落盘，而pve默认没有带这个包
只安装nbdkit还不够,以为在 Debian/Ubuntu 软件源里, nbdcopy 和 nbdinfo 属于另一个叫做 libnbd-bin 的独立软件包


# 把nfs目录挂载过来
root@pve:~# mkdir -p /tmp/esxi_v2v
root@pve:~# mkdir -p /mnt/pve/nfs-shared-storage1/images/102
root@pve:~# sshfs -o allow_other,idmap=user root@192.168.2.122:/vmfs/volumes/eca5bd35-3a696585/rocky8.5    /tmp/esxi_v2v
注:2.122是esxi地址

root@pve:~# df -Th | egrep '(esxi|mnt)'
192.168.2.113:/mnt/nfs_shares/pve_storage                   nfs4         20G   16G  2.8G  86% /mnt/pve/nfs-shared-storage
192.168.2.113:/mnt/nfs_shares/pve_storage1                  nfs4         59G   21G   36G  37% /mnt/pve/nfs-shared-storage1
root@192.168.2.122:/vmfs/volumes/eca5bd35-3a696585/rocky8.5 fuse.sshfs   59G   21G   36G  37% /tmp/esxi_v2v


# 下一条命令virt-v2v -v -x -i...可查看执行中的日志
root@pve:~# virt-v2v -i vmx "/tmp/esxi_v2v/rocky8.5.vmx" -o local -os /mnt/pve/nfs-shared-storage1/images/102 -of qcow2 --bandwidth 50M
[   0.0] Setting up the source: -i vmx /tmp/esxi_v2v/rocky8.5.vmx
[   1.0] Opening the source
[   5.2] Inspecting the source
[  28.0] Checking for sufficient free disk space in the guest
[  28.0] Converting Rocky Linux 8.5 (Green Obsidian) to run on KVM
virt-v2v: This guest has virtio drivers installed.
[ 102.9] Mapping filesystem data to avoid copying unused and blank areas
[ 104.2] Closing the overlay
[ 104.3] Assigning disks to buses
[ 104.3] Checking if the guest needs BIOS or UEFI to boot
[ 104.3] Setting up the destination: -o disk -os /mnt/pve/nfs-shared-storage1/images/102
[ 105.4] Copying disk 1/1
 100% [****************************************]
[1937.1] Creating output metadata
virt-v2v: warning: unknown guest operating system: linux rocky 8.5 x86_64 
(Rocky Linux 8.5 (Green Obsidian))
[1937.2] Finishing off

root@pve:~# ls -alh /mnt/pve/nfs-shared-storage1/images/102
total 4.8G
drwxr-xr-x 2 root root 4.0K May 25 11:50 .
drwxr-xr-x 4 root root 4.0K May 25 10:53 ..
-rw-r--r-- 1 root root 4.8G May 25 11:50 rocky8.5-sda           # 注入了kvm驱动的裸磁盘数据文件
-rw-r--r-- 1 root root 1.5K May 25 11:50 rocky8.5.xml           # 虚拟机的标准硬件配置文件

由于我们用了-o local(输出到本地目录)模式，pve的虚拟化管理器(pvedaemon)目前还没有把这个磁盘和具体的虚拟机id绑定起来
接下来需要在pve界面里把这个 Rocky Linux 8.5 彻底拉起来并完成闭环

💡 避坑细节: virt-v2v生成的rocky8.5.xml是标准的Libvirt XML硬件定义文件。由于Proxmox(PVE)使用的不是原生libvirt结构，而是自己的 /etc/pve/qemu-server/102.conf 架构，因此这个.xml文件在pve创建完空壳vm并执行 qm rescan 后已经失去了利用价值
完成磁盘挂载后，可以直接执行 rm -f /mnt/pve/nfs-shared-storage1/images/102/rocky8.5.xml 清理垃圾，防止污染存储卷


第一步：在pve上创建一个空壳虚拟机(VM 102)
我们需要给这个磁盘搭一个"家"
1. 登录pve Web界面
2. 点击右上角的 Create VM(创建虚拟机):
	General：VM ID 填 102，名称写 rocky8.5
	OS：选择 Do not use any media(不使用任何介质)
	System：保持默认(如果在esxi里是用UEFI引导的，这里就选OVMF/UEFI；如果是传统引导，保持默认的SeaBIOS)
	Disks：直接点击右下角的垃圾桶图标，把自带的那个磁盘删掉(因为我们要用刚才搬过来的盘，不需要新盘)
	CPU / Memory：根据需求分配，尽量与旧环境一致
	Network：选择你的网桥(通常是vmbr0)，网卡模型保持 VirtIO (semi-virtualized)
3. 一路下一步直到完成

第二步：将搬迁过来的磁盘强行"改名并归位"
因为virt-v2v生成的文件名叫rocky8.5-sda，而pve的NFS存储对磁盘命名有严格的格式规范(格式必须是vm-102-disk-0.qcow2).所以直接做重命名和类型修正
在pve终端执行以下两条命令：
# 1. 顺手把后缀改成标准的.qcow2格式(virt-v2v 实际导出的就是qcow2，只是没加后缀)
root@pve:~# mv /mnt/pve/nfs-shared-storage1/images/102/rocky8.5-sda    /mnt/pve/nfs-shared-storage1/images/102/vm-102-disk-0.qcow2

# 2. 让pve强制重新扫描vm 102的目录，刷新注册表
root@pve:~# qm rescan --vmid 102
rescan volumes...
VM 102 add unreferenced volume 'nfs-shared-storage1:102/vm-102-disk-0.qcow2' as 'unused0' to config


第三步：挂载磁盘并激活开机
1. 回到pve网页后台，点击刚建好的102(rocky8.5)虚拟机
2. 点击Hardware(硬件) 菜单会看到一个黄色的图标写着Unused Disk 0
3. 双击这个 Unused Disk 0:
	Bus/Device(总线/设备): 选择SCSI
	Cache(缓存): 生产环境建议选择Write back(unsafe)或默认，提升nfs性能
	点击Add(添加)
```
![image](./images/64.png)

```shell
4. 点击Options(选项) ---> 双击Boot Order(引导顺序):
	把刚刚添加的scsi0勾选上，并且用鼠标把它拖动到第一位(最顶上)
	点击OK保存,然后开机
```
![image](./images/65.png)
![image](./images/66.png)

```shell
# 登录进去后发现没有IP
1. 修改网卡名
[rambo@lcoalhost ~]$ cd /etc/sysconfig/network-scripts/
[rambo@lcoalhost network-scripts]$ sudo mv ifcfg-ens192  ifcfg-ens18
# ================================================
# 如上一条命令失败后则可用这条命令把原来名为ens192的配置强行绑定到现在的pve虚拟网卡ens18上
# [rambo@lcoalhost network-scripts]$ sudo nmcli con mod ens192 connection.interface-name  ens18
# ================================================

[rambo@lcoalhost network-scripts]$ nmcli con modify ens192  connection.id ens18
[rambo@lcoalhost network-scripts]$ sudo nmcli con mod ens18 \
ipv4.addresses 192.168.2.146/24 \
ipv4.gateway 192.168.2.1 \
ipv4.dns 192.168.2.1 \
connection.autoconnect yes \
ipv4.method manual

[rambo@192 ~]$ sudo nmcli con reload
[rambo@192 ~]$ sudo nmcli con up ens18
[rambo@192 ~]$ sudo nmcli con show
[rambo@192 ~]$ ping qq.com


一切都没问题后不要让sshfs的虚拟挂载一直常驻在本地/tmp里，一旦esxi重启，pve本地的这个目录就会变成"僵尸目录"，导致 df -h 直接卡死
# 迁移大功告成后，强行解绑 SSHFS 挂载点，保持 PVE 节点纯净
[rambo@192 ~]$ fusermount -u /tmp/esxi_v2v && rmdir /tmp/esxi_v2v


```




## 迁移EFI格式
![image](./images/67.png)
```shell
# 在esxi8上
[root@localhost:~] ls -alh /vmfs/volumes/eca5bd35-3a696585/
total 36
drwxr-xr-x    7 root     root        4.0K May 25  2026 .
drwxr-xr-x    1 root     root         512 May 25 04:49 ..
drwxr-xr-x    3 root     root        4.0K May 25  2026 images
drwx------    2 root     root       16.0K May 24 02:26 lost+found
drwxr-xr-x    2 root     root        4.0K May 24 07:52 rocky8.5
drwxr-xr-x    4 root     root        4.0K May 24 13:03 template
drwxr-xr-x    2 root     root        4.0K May 25  2026 ub24.4-1


# 在pve2上
root@pve:~# mkdir -p /tmp/esxi_v2v_ub24
root@pve:~# mkdir -p /mnt/pve/nfs-shared-storage1/images/100
root@pve:~# sshfs -o allow_other,idmap=user root@192.168.2.122:/vmfs/volumes/eca5bd35-3a696585/ub24.4-1    /tmp/esxi_v2v_ub24
root@pve:~# df -Th | grep ub24
root@192.168.2.122:/vmfs/volumes/eca5bd35-3a696585/ub24.4-1 fuse.sshfs   59G   21G   36G  37% /tmp/esxi_v2v_ub24
root@pve:~# virt-v2v -i vmx "/tmp/esxi_v2v/ub24.4-1.vmx" -o local -os /mnt/pve/nfs-shared-storage1/images/100  -of raw --bandwidth 50M
注: 这里我换成了raw格式, 务必注意
[   0.0] Setting up the source: -i vmx /tmp/esxi_v2v_ub24/ub24.4-1.vmx
qemu-nbd: Failed to blk_new_open '/tmp/esxi_v2v_ub24/ub24.4-1.vmdk': Could not open '/tmp/esxi_v2v_ub24/ub24.4-1-flat.vmdk': Operation not permitted
virt-v2v: Exiting on signal SIGINT


# 在pve节点上查看
root@pve:~# ls -alh /tmp/esxi_v2v_ub24/ub24.4-1.vmx
-rwxr-xr-x 1 root root 3.1K May 25 20:57 /tmp/esxi_v2v_ub24/ub24.4-1.vmx
root@pve:~# ls -alh /tmp/esxi_v2v_ub24                      # 以下情况是这台vm还在开着机,所以上面报错了
total 45G
drwxr-xr-x  1 root root 4.0K May 25 20:57 .
drwxrwxrwt 11 root root 4.0K May 25 21:37 ..
-rwxrwxr-x  1 root root   92 May 25 21:38 .lck-0200080000000000
-rwxrwxr-x  1 root root   92 May 25 21:38 .lck-0500080000000000
-rwxrwxr-x  1 root root   92 May 25 21:38 .lck-0700080000000000
-rwxrwxr-x  1 root root   92 May 25 21:38 .lck-1000080000000000
-rw-------  1 root root 4.0G May 25 20:57 ub24.4-1-0c575239.vswp
-rw-r--r--  1 root root 7.3K May 25 20:57 ub24.4-1-1.scoreboard
-rw-------  1 root root  40G May 25 21:37 ub24.4-1-flat.vmdk
-rw-------  1 root root 265K May 25 20:57 ub24.4-1.nvram
-rw-r--r--  1 root root 7.3K May 25 12:57 ub24.4-1.scoreboard
-rw-------  1 root root  474 May 25 20:57 ub24.4-1.vmdk
-rw-r--r--  1 root root    0 May 25 18:20 ub24.4-1.vmsd
-rwxr-xr-x  1 root root 3.1K May 25 20:57 ub24.4-1.vmx
-rw-------  1 root root    0 May 25 20:57 ub24.4-1.vmx.lck
-rw-r--r--  1 root root 211K May 25 20:57 vmware-1.log
-rw-r--r--  1 root root 142K May 25 21:32 vmware.log
-rw-------  1 root root  82M May 25 20:57 vmx-ub24.4-1-c67e44cc1e16ca8270b9b6417c355f1d2ec2fd913e97f4a8e4c99546aab7b22f-1.vswp

# 在pve节点上查看, 以下情况才是关机的状态
root@pve:~# ls -alh /tmp/esxi_v2v_ub24
total 41G
drwxr-xr-x  1 root root 4.0K May 25 21:38 .
drwxrwxrwt 11 root root 4.0K May 25 21:37 ..
-rw-r--r--  1 root root 7.3K May 25 20:57 ub24.4-1-1.scoreboard
-rw-------  1 root root  40G May 25 21:37 ub24.4-1-flat.vmdk
-rw-------  1 root root 265K May 25 20:57 ub24.4-1.nvram
-rw-r--r--  1 root root 7.3K May 25 21:38 ub24.4-1.scoreboard
-rw-------  1 root root  474 May 25 20:57 ub24.4-1.vmdk
-rw-r--r--  1 root root    0 May 25 18:20 ub24.4-1.vmsd
-rwxr-xr-x  1 root root 3.1K May 25 21:38 ub24.4-1.vmx
-rw-r--r--  1 root root 211K May 25 20:57 vmware-1.log
-rw-r--r--  1 root root 160K May 25 21:38 vmware.log

# 然后再看是迁移
root@pve:~# virt-v2v -i vmx "/tmp/esxi_v2v_ub24/ub24.4-1.vmx" -o local -os /mnt/pve/nfs-shared-storage1/images/100  -of  raw --bandwidth 50M
[   0.0] Setting up the source: -i vmx /tmp/esxi_v2v_ub24/ub24.4-1.vmx
[   1.0] Opening the source
[   4.4] Inspecting the source
[  18.0] Checking for sufficient free disk space in the guest
[  18.0] Converting Ubuntu 24.04.1 LTS to run on KVM
virt-v2v: warning: could not determine a way to update the configuration of 
Grub2
virt-v2v: error: libguestfs error: file_architecture: unknown architecture: 
/usr/lib/modules/6.17.0-29-generic/kernel/arch/x86/kvm/kvm.ko.zst

If reporting bugs, run virt-v2v with debugging enabled and include the 
complete output:

  virt-v2v -v -x [...]

报错分析:
这个报错直接戳中了 virt-v2v 引擎在2026年面对全新linux发行版(如Ubuntu 24.04及后续版本)时的一个底层静态解析内核漏洞
用第一性原理来解剖这个报错：file_architecture: unknown architecture: .../kvm.ko.zst
新版内核的压缩算法变了: Ubuntu 24.04引入了非常激进的内核优化，将其所有的内核模块(.ko文件)全部采用了ZSTD (.zst) 算法进行高压缩
工具链老旧无法识别: pve 宿主机上的 libguestfs-tools(也就是virt-v2v依赖的底层文件系统解剖工具)在解析内核架构时，使用的是经典的 file 命令或者早期的魔术字(Magic Number)扫描。它只认识.ko(未压缩)或者 .ko.gz(Gzip压缩)
彻底抓瞎： 当它扫描到 /.../kvm.ko.zst 时，由于无法就地解压并读取这个 ELF 文件的内核架构标记（x86_64），它直接判定为 unknown architecture 并抛出致命异常闪退
在现场给客户处理问题时，如果卡在这一步，我们不需要等待官方更新补丁，我们直接降维打击，绕过 virt-v2v 脆弱的内核扫描
既然virt-v2v的"医生功能"在Ubuntu 24.04的ZSTD内核面前瘫痪了，那就只用它来做最纯粹的数据传输(Data Copy)，强制跳过内核转换
对于老旧的 CentOS 6/7 或者早期的 Windows，我们必须用 virt-v2v 去注入 VirtIO 驱动，否则开机必定蓝屏或找不到盘。

Ubuntu 24.04 是2024年以后发布的现代主流系统，它的标准主流内核(6.8+)在出厂时，早就已经把原生的 KVM VirtIO 驱动(包括virtio_blk、virtio_scsi、virtio_net)硬焊进内核核心了！ 它根本不需要 virt-v2v 去人工注入

# 下条命令中不是 -flat.vmdk, 是直接的.vmdk
root@pve:~# qemu-img convert -p -f vmdk -O raw  /tmp/esxi_v2v_ub24/ub24.4-1.vmdk  /mnt/pve/nfs-shared-storage1/images/100/vm-100-disk-0.raw
    (100.00/100%)


# 方法1
在pve终端强行给vm 100 注入UEFI固件并把引用的EFI盘直接格式化在NFS上：
root@pve:~# qm set 100 --bios ovmf --efidisk0 nfs-shared-storage1:0,format=raw,pre-enrolled=0

# 方法2(我使用的该方法)
因为我是先创建的空vm,所以会用下图的方式修改
```
![image](./images/68.png)
![image](./images/69.png)
![image](./images/70.png)
![image](./images/71.png)


```shell
# 方法3  如果你还没有创建空vm则:
点击 PVE 右上角的 Create VM(创建虚拟机)
切换到第三个标签页System(系统)
找到 BIOS 这一行，默认是 Default (SeaBIOS)，点击下拉菜单，将其修改为 OVMF (UEFI)
选成OVMF(UEFI) 后，下方会立刻弹出一个EFI Storage的下拉框，必须在这里选择你的存储卷(比如你的nfs-shared-storage1)，用来存放UEFI的非易失性变量(NVRAM)
取消勾选 Pre-Enroll keys(除非你的Ubuntu启用了安全启动 Secure Boot，一般企业迁移都不勾选，防止密钥不匹配死锁)


无论使用上述3种方法中的哪种，都需要让pve强制重新扫描并捡起这个.raw硬盘
root@pve:~# qm rescan --vmid 100
rescan volumes...
VM 100 add unreferenced volume 'nfs-shared-storage1:100/vm-100-disk-0.raw' as 'unused0' to config

用命令挂载并强行绑定引导顺序
root@pve:~# qm set 100 --scsi0 nfs-shared-storage1:100/vm-100-disk-0.raw,discard=on && qm set 100 --boot order=scsi0
命令释义:
--scsi0：将磁盘挂载到VirtIO SCSI控制器的0号通道上，这是pve下性能最强、延迟最低的块设备通道
discard=on：开启回收空白块特性。因为我用的是raw格式，开启这个后，未来你在 Ubuntu 24内部执行fstrim，nfs存储端会自动释放被删除文件占用的物理空间，防止存储虚胖
--boot order=scsi0：直接重写虚拟机的配置文件，强行把scsi0顶到最前面，压制住新建的那个空壳EFI盘

或者在图形化中挂载并强行绑定引导顺序(本次用该方法)
```
![image](./images/72.png)
![image](./images/73.png)
![image](./images/74.png)
![image](./images/75.png)
![image](./images/76.png)













