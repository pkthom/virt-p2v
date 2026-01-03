# virt-p2v

CentOS5.11[(こちらで構築)](https://github.com/pkthom/centos_5.11)の物理マシンを、CentOS6.10(KVMホスト)[(こちらで構築)](https://github.com/pkthom/centos_6.10)にp2vする

## 事前チェック

### 受け入れ側(CentOS6.10)チェック

- ✅KVM/Libvirtは動いている
<img width="629" height="199" alt="image" src="https://github.com/user-attachments/assets/7bd95b2e-327c-4a83-ab23-7b25a2b4b1bf" />

- ✅受け入れられるだけの容量がある

CentOS5.11は、1.6G

<img width="643" height="177" alt="image" src="https://github.com/user-attachments/assets/e1be2093-7394-444b-bcbe-f37ebba125dd" />

CentOS6.10は、/ に40GB残ってる

<img width="657" height="222" alt="image" src="https://github.com/user-attachments/assets/0aa8bad5-d848-4aaf-9e55-e86e936cd961" />

- ✖virt-v2vが入っている -> 入ってない

<img width="899" height="107" alt="image" src="https://github.com/user-attachments/assets/c72b12f0-b224-4562-8218-0fe8baf57d41" />



### P2V対象側(CentOS5.11)チェック

- ✅KVMホスト(CentOS6.10)にパスワードなしでSSHできる

<img width="649" height="267" alt="image" src="https://github.com/user-attachments/assets/c8a07bef-5eaf-41a8-b7cd-012f263f2180" />

- ✖virt-v2vホスト(RHEL10)にパスワードなしでSSHできる　→　できない no kex alg　

<img width="453" height="49" alt="image" src="https://github.com/user-attachments/assets/29824bb4-dadf-40d7-8996-e36184cdec64" />

-> 物理マシン側のSSHが古すぎる 

RHEL 9や10では、セキュリティを確保するために古い暗号がシステムレベルで封じられています。これを一時的に解除します。（以下をRHEL10で実行）
```
ubuntu@rhel10:~$ sudo update-crypto-policies --set LEGACY
[sudo] ubuntu のパスワード:
Setting system policy to LEGACY
Note: System-wide crypto policies are applied on application start-up.
It is recommended to restart the system for the change of policies
to fully take place.
ubuntu@rhel10:~$
```
※⚠上記だけでは解決せず、以下が必要になるが、上記がないと、以下をやっても　no kex algが出るので、上記は必要です

まだSSHできない　エラーが変わる　no hostkey alg

<img width="402" height="50" alt="image" src="https://github.com/user-attachments/assets/53363771-5a90-4395-a55e-e404cd1e2d89" />

以下をRHEL10で実行
```
ubuntu@rhel10:/etc/ssh/sshd_config.d# cat 05-hostkey-legacy.conf
HostKey /etc/ssh/ssh_host_rsa_key
HostKeyAlgorithms +ssh-rsa
ubuntu@rhel10:/etc/ssh/sshd_config.d# cat 10-centos5-compat.conf
Match Address <<CentOS5.11のIPアドレス>>
    PubkeyAcceptedAlgorithms +ssh-rsa
ubuntu@rhel10:/etc/ssh/sshd_config.d# sshd -t
ubuntu@rhel10:/etc/ssh/sshd_config.d# systemctl restart sshd
ubuntu@rhel10:/etc/ssh/sshd_config.d#
```
-> SSHできるようになるので、鍵を置く

✅パスワードなしでSSHできるようになった

<img width="358" height="30" alt="image" src="https://github.com/user-attachments/assets/f839e7b4-349d-4795-a5a2-19a9210d8371" />

※CentOS5側のconfigは、centos6-8300用のものと同じ

<img width="466" height="365" alt="image" src="https://github.com/user-attachments/assets/508d2f09-5b55-477c-9974-f6d791f1e09c" />


- ✖virt-p2vが入っている -> 入ってない

<img width="900" height="128" alt="image" src="https://github.com/user-attachments/assets/84fd799c-4426-4b8a-b13c-acb733c650e4" />

### virt-v2v実施側チェック(RHEL10)

- ✅KVMホスト(CentOS6.10)にパスワードなしでSSHできる

```
ubuntu@localhost:~$ cat .ssh/config
Host centos6-8300
    HostName <<centos6-8300のIPアドレス>>
    User root
    IdentityFile ~/.ssh/id_rsa_centos6-8300
    HostKeyAlgorithms +ssh-rsa
    PubkeyAcceptedAlgorithms +ssh-rsa
    IdentitiesOnly yes
    ServerAliveInterval 60
ubuntu@localhost:~$
```
鍵はわざと古い形式で作らないと受け入れてくれない　エラーが出るわけではないが、鍵を向こうに置いてきてもパスワードを求められる
```
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa_centos6-8300
```
古い鍵を置きに行く
```
ssh-copy-id centos6-8300
```
✅パスワードなしでSSHできるようになる




## RHEL10に変換ツールを入れる

最初は入ってない

<img width="706" height="116" alt="image" src="https://github.com/user-attachments/assets/e73b9905-b896-470f-a2fc-3799219f6b44" />

登録がないと、レポジトリが使えない

<img width="1576" height="251" alt="image" src="https://github.com/user-attachments/assets/4e0f4acf-ef11-4dc2-b68b-8f54b3f1e1d3" />

以下で登録
```
sudo subscription-manager register --username 'ユーザー名' --password 'パスワード'
```

✅登録済みなことを確認

<img width="727" height="222" alt="image" src="https://github.com/user-attachments/assets/ceb1d10e-5875-423a-a1ed-68025e780eaf" />

こちらでも確認
```
sudo subscription-manager identity || true
```

BaseOS/AppStream を有効化
```
sudo subscription-manager repos \
  --enable=rhel-10-for-x86_64-baseos-rpms \
  --enable=rhel-10-for-x86_64-appstream-rpms
```
<img width="1178" height="168" alt="image" src="https://github.com/user-attachments/assets/8803f03a-11a8-4bb6-80bc-015648202198" />

✅有効になったことを確認

<img width="1448" height="225" alt="image" src="https://github.com/user-attachments/assets/934956a9-b491-4873-b8c5-6ac420be7d2a" />

virt-v2v / libguestfs をインストール
```
sudo dnf -y install virt-v2v virt-p2v libguestfs-tools-c libvirt-client
```
<img width="722" height="212" alt="image" src="https://github.com/user-attachments/assets/41c7ed7e-9ce9-48ad-b4cc-b6809dda1c70" />


✅インストールできたことを確認
```
virt-v2v --version
guestfish --version
virt-p2v-make-disk --help
```
<img width="811" height="275" alt="image" src="https://github.com/user-attachments/assets/df838be7-c5c9-40b8-9f9b-e47a3caa0b8a" />


## RHEL10から、CentOS6にvirsh接続テスト

```
ubuntu@rhel10:~$ virsh -c qemu+ssh://root@centos6-8300/system list --all
 Id   名前          状態
----------------------------
 1    almalinux10   実行中

ubuntu@rhel10:~$
```

## centos6-8300 に保存先（ストレージ）を用意
```
[root@centos6-8300 images]# pwd
/var/lib/libvirt/images
[root@centos6-8300 images]# ls
CentOS-6.10-DVD1.iso  almalinux10.img
[root@centos6-8300 images]# mkdir p2v
```

## rhel10で virt-p2v ISO を作成

エラー
```
ubuntu@rhel10:~$ sudo virt-p2v-make-disk --output /tmp/virt-p2v.iso
[sudo] ubuntu のパスワード:
virt-builder: error: cannot find os-version ‘rhel-10.0’ with
architecture ‘x86_64’.
Use --list to list available guest types.

If reporting bugs, run virt-builder with debugging enabled and include the
complete output:

  virt-builder -v -x [...]
```


<details>
    <summary>virt-builder --list</summary>

```
ubuntu@rhel10:~$ virt-builder --list
alma-8.5                 x86_64     AlmaLinux 8.5
centos-6                 x86_64     CentOS 6.6
centos-7.0               x86_64     CentOS 7.0
centos-7.1               x86_64     CentOS 7.1
centos-7.2               aarch64    CentOS 7.2 (aarch64)
centos-7.2               x86_64     CentOS 7.2
centos-7.3               x86_64     CentOS 7.3
centos-7.4               x86_64     CentOS 7.4
centos-7.5               x86_64     CentOS 7.5
centos-7.6               x86_64     CentOS 7.6
centos-7.7               x86_64     CentOS 7.7
centos-7.8               x86_64     CentOS 7.8
centos-8.0               x86_64     CentOS 8.0
centos-8.2               x86_64     CentOS 8.2
centosstream-8           x86_64     CentOS Stream 8
centosstream-9           x86_64     CentOS Stream 9
cirros-0.3.1             x86_64     CirrOS 0.3.1
cirros-0.3.5             x86_64     CirrOS 0.3.5
debian-6                 x86_64     Debian 6 (Squeeze)
debian-7                 sparc64    Debian 7 (Wheezy) (sparc64)
debian-7                 x86_64     Debian 7 (wheezy)
debian-8                 x86_64     Debian 8 (jessie)
debian-9                 x86_64     Debian 9 (stretch)
debian-10                x86_64     Debian 10 (buster)
debian-11                x86_64     Debian 11 (bullseye)
debian-12                x86_64     Debian 12 (bookworm)
debian-13                x86_64     Debian 13 (trixie)
fedora-41                x86_64     Fedora® 41 Server
fedora-41                aarch64    Fedora® 41 Server (aarch64)
fedora-42                x86_64     Fedora® 42 Server
freebsd-11.1             x86_64     FreeBSD 11.1
scientificlinux-6        x86_64     Scientific Linux 6.5
ubuntu-10.04             x86_64     Ubuntu 10.04 (Lucid)
ubuntu-12.04             x86_64     Ubuntu 12.04 (Precise)
ubuntu-14.04             x86_64     Ubuntu 14.04 (Trusty)
ubuntu-16.04             x86_64     Ubuntu 16.04 (Xenial)
ubuntu-18.04             x86_64     Ubuntu 18.04 (bionic)
ubuntu-20.04             x86_64     Ubuntu 20.04 (focal)
fedora-18                x86_64     Fedora® 18
fedora-19                x86_64     Fedora® 19
fedora-20                x86_64     Fedora® 20
fedora-21                aarch64    Fedora® 21 Server (aarch64)
fedora-21                armv7l     Fedora® 21 Server (armv7l)
fedora-21                ppc64      Fedora® 21 Server (ppc64)
fedora-21                ppc64le    Fedora® 21 Server (ppc64le)
fedora-21                x86_64     Fedora® 21 Server
fedora-22                aarch64    Fedora® 22 Server (aarch64)
fedora-22                armv7l     Fedora® 22 Server (armv7l)
fedora-22                i686       Fedora® 22 Server (i686)
fedora-22                x86_64     Fedora® 22 Server
fedora-23                aarch64    Fedora® 23 Server (aarch64)
fedora-23                armv7l     Fedora® 23 Server (armv7l)
fedora-23                i686       Fedora® 23 Server (i686)
fedora-23                ppc64      Fedora® 23 Server (ppc64)
fedora-23                ppc64le    Fedora® 23 Server (ppc64le)
fedora-23                x86_64     Fedora® 23 Server
fedora-24                aarch64    Fedora® 24 Server (aarch64)
fedora-24                armv7l     Fedora® 24 Server (armv7l)
fedora-24                i686       Fedora® 24 Server (i686)
fedora-24                x86_64     Fedora® 24 Server
fedora-25                aarch64    Fedora® 25 Server (aarch64)
fedora-25                armv7l     Fedora® 25 Server (armv7l)
fedora-25                i686       Fedora® 25 Server (i686)
fedora-25                ppc64      Fedora® 25 Server (ppc64)
fedora-25                ppc64le    Fedora® 25 Server (ppc64le)
fedora-25                x86_64     Fedora® 25 Server
fedora-26                aarch64    Fedora® 26 Server (aarch64)
fedora-26                armv7l     Fedora® 26 Server (armv7l)
fedora-26                i686       Fedora® 26 Server (i686)
fedora-26                ppc64      Fedora® 26 Server (ppc64)
fedora-26                ppc64le    Fedora® 26 Server (ppc64le)
fedora-26                x86_64     Fedora® 26 Server
fedora-27                aarch64    Fedora® 27 Server (aarch64)
fedora-27                armv7l     Fedora® 27 Server (armv7l)
fedora-27                i686       Fedora® 27 Server (i686)
fedora-27                ppc64      Fedora® 27 Server (ppc64)
fedora-27                ppc64le    Fedora® 27 Server (ppc64le)
fedora-27                x86_64     Fedora® 27 Server
fedora-28                i686       Fedora® 28 Server (i686)
fedora-28                x86_64     Fedora® 28 Server
fedora-29                aarch64    Fedora® 29 Server (aarch64)
fedora-29                i686       Fedora® 29 Server (i686)
fedora-29                ppc64le    Fedora® 29 Server (ppc64le)
fedora-29                x86_64     Fedora® 29 Server
fedora-30                aarch64    Fedora® 30 Server (aarch64)
fedora-30                i686       Fedora® 30 Server (i686)
fedora-30                x86_64     Fedora® 30 Server
fedora-31                x86_64     Fedora® 31 Server
fedora-32                x86_64     Fedora® 32 Server
fedora-33                x86_64     Fedora® 33 Server
fedora-34                x86_64     Fedora® 34 Server
fedora-34                armv7l     Fedora® 34 Server (armv7l)
fedora-35                x86_64     Fedora® 35 Server
fedora-35                aarch64    Fedora® 35 Server (aarch64)
fedora-36                x86_64     Fedora® 36 Server
fedora-37                x86_64     Fedora® 37 Server
fedora-38                x86_64     Fedora® 38 Server
fedora-38                aarch64    Fedora® 38 Server (aarch64)
fedora-39                x86_64     Fedora® 39 Server
fedora-40                x86_64     Fedora® 40 Server
ubuntu@rhel10:~$
```

</details>

-> 最も近い、centosstream-9 は選べず
```
ubuntu@rhel10:~$ sudo virt-p2v-make-disk --output /tmp/virt-p2v.iso centosstream-9
[sudo] ubuntu のパスワード:
virt-p2v-make-disk: internal error: could not work out the Linux distro from 'centosstream-9'
```

fedora42を選ぶ
```
ubuntu@rhel10:~$ sudo virt-p2v-make-disk --output /tmp/virt-p2v.iso fedora-42
[   3.5] Downloading: https://builder.libguestfs.org/fedora-42.xz
################################################################################################################# 100.0%
[ 226.2] Planning how to build this image
[ 226.2] Uncompressing
[ 230.2] Opening the new disk
[ 250.5] Setting a random seed
[ 250.5] Setting the hostname: p2v.local
[ 250.5] Running: hostname p2v.local
[ 250.6] Updating packages
[ 393.0] Installing packages: pcre2 libxml2 librsvg2 gtk3 dbus-libs /usr/bin/ssh nbdkit-server nbdkit-file-plugin which vim-minimal iscsi-initiator-utils /usr/bin/xinit /usr/bin/Xorg xorg-x11-drivers xorg-x11-fonts-Type1 dejavu-sans-fonts dejavu-sans-mono-fonts mesa-dri-drivers metacity NetworkManager nm-connection-editor network-manager-applet dbus-x11 net-tools @hardware-support shim-x64 grub2-efi-x64-cdboot curl ethtool gawk lsscsi pciutils usbutils util-linux xterm less hwdata hdparm smartmontools
[ 449.6] Uploading: /usr/share/virt-p2v/issue to /etc/issue
[ 449.6] Uploading: /usr/share/virt-p2v/issue to /etc/issue.net
[ 449.7] Making directory: /usr/bin
[ 449.7] Uploading: /tmp/tmp.1OFoOWtmQx/virt-p2v to /usr/bin/virt-p2v
[ 449.7] Changing permissions of /usr/bin/virt-p2v to 0755
[ 449.7] Uploading: /usr/share/virt-p2v/launch-virt-p2v to /usr/bin/
[ 449.7] Changing permissions of /usr/bin/launch-virt-p2v to 0755
[ 449.7] Uploading: /usr/share/virt-p2v/p2v.service to /etc/systemd/system/
[ 449.7] Making directory: /etc/systemd/system/multi-user.target.wants
[ 449.7] Linking: /etc/systemd/system/multi-user.target.wants/p2v.service -> /etc/systemd/system/p2v.service
[ 449.7] Editing: /lib/systemd/system/getty@.service
[ 449.8] Copying (in image): /usr/lib/systemd/logind.conf to /etc/systemd/logind.conf
[ 449.8] Editing: /etc/systemd/logind.conf
[ 449.8] Uploading: /tmp/tmp.1OFoOWtmQx/p2v.conf to /etc/dracut.conf.d/
[ 449.8] Running: /tmp/tmp.1OFoOWtmQx/post-install
[ 468.1] Setting passwords
[ 468.8] SELinux relabelling
[ 470.5] Finishing off
                   Output file: /tmp/virt-p2v.iso
                   Output size: 6.0G
                 Output format: raw
            Total usable space: 5.9G
                    Free space: 2.5G (42%)
ubuntu@rhel10:~$
```

ProxmoxにUSB

RHEL10 VMを選んで、画像の通りUSBを認識させる

<img width="874" height="720" alt="image" src="https://github.com/user-attachments/assets/d9ed07c8-d80d-4a2a-a2b6-c7febe29f133" />

追加できた

<img width="773" height="411" alt="image" src="https://github.com/user-attachments/assets/f205cad3-35ac-40a5-abdf-e55a61bb9330" />

/dev/sdb としてUSBが追加されている
```
ubuntu@rhel10:~$ lsblk -o NAME,SIZE,MODEL,TRAN,TYPE,MOUNTPOINT
NAME           SIZE MODEL          TRAN   TYPE MOUNTPOINT
sda             80G QEMU HARDDISK         disk
├─sda1         600M                       part /boot/efi
├─sda2           1G                       part /boot
└─sda3        78.4G                       part
  ├─rhel-root 47.4G                       lvm  /
  ├─rhel-swap  7.9G                       lvm  [SWAP]
  └─rhel-home 23.1G                       lvm  /home
sdb           28.9G USB Flash Disk usb    disk
└─sdb1        28.9G                       part
sr0            7.9G QEMU DVD-ROM   sata   rom
ubuntu@rhel10:~$
```

念のためマウント解除
```
ubuntu@rhel10:~$ lsblk -f /dev/sdb
NAME   FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sdb
└─sdb1 vfat   FAT32       4703-6FEC
ubuntu@rhel10:~$ sudo umount /dev/sdb1 2>/dev/null || true
[sudo] ubuntu のパスワード:
ubuntu@rhel10:~$
```
書き込み
```
ubuntu@rhel10:~$ sudo dd if=/tmp/virt-p2v.iso of=/dev/sdb bs=4M status=progress conv=fsync
6438256640 bytes (6.4 GB, 6.0 GiB) copied, 590 s, 10.9 MB/s6442450944 bytes (6.4 GB, 6.0 GiB) copied, 590.915 s, 10.9 MB/s

1536+0 records in
1536+0 records out
6442450944 bytes (6.4 GB, 6.0 GiB) copied, 769.571 s, 8.4 MB/s
ubuntu@rhel10:~$ 
```
簡易チェック
```
ubuntu@rhel10:~$  sudo cmp -n 64M /tmp/virt-p2v.iso /dev/sdb && echo "OK: first 64MB match"
OK: first 64MB match
ubuntu@rhel10:~$
```
抜く準備　VMからマウントポイントなし　
```
ubuntu@rhel10:~$ sync
ubuntu@rhel10:~$ lsblk -o NAME,MOUNTPOINT /dev/sdb
NAME   MOUNTPOINT
sdb
├─sdb1
├─sdb2
└─sdb3
ubuntu@rhel10:~$
```

VMからUSBをデタッチ

<img width="782" height="548" alt="image" src="https://github.com/user-attachments/assets/bdbece62-dfec-4d1b-89f9-bca605b9a45d" />

USBをCentOS5物理マシンにさして、F12でGRUB起動　USBを選択して起動

ログインできたが、GUI launch-virt-p2v が起動しない　

CPU ISA level lower than required とある　物理マシンのCPUが、Fedora42 にとって古すぎるようです

centos-7.8 でISO作り直す　→　こちらもエラー `mirrorlist.centos.org` が使えないらしい
```
ubuntu@rhel10:~$ sudo virt-p2v-make-disk --output /tmp/virt-p2v.iso centos-7.8
[   4.2] Downloading: https://builder.libguestfs.org/centos-7.8.xz
################################################################################################################# 100.0%
[  77.7] Planning how to build this image
[  77.7] Uncompressing
[  80.8] Opening the new disk
[  95.1] Setting a random seed
[  95.2] Setting the hostname: p2v.local
[  95.2] Running: hostname p2v.local
[  95.2] Updating packages
Loaded plugins: fastestmirror
Determining fastest mirrors
Could not retrieve mirrorlist http://mirrorlist.centos.org/?release=7&arch=x86_64&repo=os&infra=stock error was
14: curl#6 - "Could not resolve host: mirrorlist.centos.org; Unknown error"

```
ubuntu-20.04もエラー
```
ubuntu@rhel10:~$ sudo virt-p2v-make-disk --output /tmp/virt-p2v.iso ubuntu-20.04
...
[ 279.6] Editing: /lib/systemd/system/getty@.service
[ 279.6] Copying (in image): /usr/lib/systemd/logind.conf to /etc/systemd/logind.conf
virt-builder: error: libguestfs error: cp_a: cp: cannot stat
'/sysroot/usr/lib/systemd/logind.conf': No such file or directory

If reporting bugs, run virt-builder with debugging enabled and include the
complete output:

  virt-builder -v -x [...]
```
debian-11 もエラー
```
ubuntu@rhel10:~$ sudo virt-p2v-make-disk --output /tmp/virt-p2v.iso debian-12
...
Processing triggers for ca-certificates (20230311+deb12u1) ...
Updating certificates in /etc/ssl/certs...
0 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d...
done.
Errors were encountered while processing:
 grub-pc
E: Sub-process /usr/bin/dpkg returned an error code (1)
virt-builder: error:
      export DEBIAN_FRONTEND=noninteractive
      apt_opts='-q -y -o Dpkg::Options::=--force-confnew'
      apt-get $apt_opts update
      apt-get $apt_opts upgrade
    : command exited with an error

If reporting bugs, run virt-builder with debugging enabled and include the
complete output:

  virt-builder -v -x [...]
```

fedora40を選ぶ
```

```

## パフォーマンステスト

### p2v前

### 書き込みテスト
```
sync
time dd if=/dev/zero of=/root/dd_test.bin bs=1M count=8192 oflag=direct conv=fdatasync
rm -f /root/dd_test.bin
```
※解説
```
if=/dev/zero: 入力元。中身がゼロの無限データを読み込みます。
of=/root/dd_test.bin: 出力先。/root ディレクトリにテスト用の大きなファイルを作ります。
bs=1M: ブロックサイズ。1回につき 1MB ずつ書き込みます。
count=1024: 回数。8192回繰り返すので、合計 8GB のファイルを書き込むことになります。
conv=fdatasync: 重要。 OSのキャッシュ（メモリ）に書き込んだ時点で「終わり」とせず、物理的なディスクにデータが書き込まれるまで待機します。これにより、本当のディスク性能が測れます。
```
※sync -> OSがメモリ上に溜めている「未書き込みデータ（dirty page）」をディスクへ書き出させる

<img width="1204" height="726" alt="image" src="https://github.com/user-attachments/assets/88866108-4441-419d-8c75-875d78aced43" />
<img width="1202" height="483" alt="image" src="https://github.com/user-attachments/assets/e570b58f-f263-4aa6-936b-4ab7b33a8b98" />

### 読み取りテスト
```
sync
echo 3 > /proc/sys/vm/drop_caches
time dd if=/root/dd_test.bin of=/dev/null bs=1M iflag=direct status=progress
```
※`echo 3 > /proc/sys/vm/drop_caches` -> Linuxのページキャッシュ等（読み込みキャッシュ）を捨てる

<img width="1146" height="723" alt="image" src="https://github.com/user-attachments/assets/d7f419d3-5aca-42d7-858f-ac4d67612212" />
<img width="920" height="483" alt="image" src="https://github.com/user-attachments/assets/a7b25e2a-dbde-4747-90d0-199690711d18" />

## 内容テスト

マーカー作成

<img width="855" height="78" alt="image" src="https://github.com/user-attachments/assets/7cc3f85a-dabe-4702-a675-82939e7509e0" />
<img width="1021" height="128" alt="image" src="https://github.com/user-attachments/assets/eef401a2-2645-4f1e-87bd-0bf12da5f8e1" />



CentOS5を停止

<img width="383" height="28" alt="image" src="https://github.com/user-attachments/assets/d4e45600-4a82-4632-b9dc-c1908b776824" />


<img width="465" height="372" alt="image" src="https://github.com/user-attachments/assets/2d3fbff4-2286-4504-a658-5699ab6854d0" />
