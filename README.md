# virt-p2v

CentOS5.11[(こちらで構築)](https://github.com/pkthom/centos_5.11)の物理マシンを、CentOS6.10(KVMホスト)[(こちらで構築)](https://github.com/pkthom/centos_6.10)にp2vする

## 事前チェック

<details>
  <summary>p2v前パフォーマンステスト</summary>

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

</details>

## ISO作成機 (RHEL10)を準備

バイナリをバックアップ（今から差し替えるため）
```
ubuntu@rhel10:~$ cd /usr/lib64/virt-p2v/
ubuntu@rhel10:/usr/lib64/virt-p2v$ ls
virt-p2v.xz
ubuntu@rhel10:/usr/lib64/virt-p2v$ sudo cp -a virt-p2v.xz virt-p2v.xz.bak
```
Rocky 8 の virt-p2v-maker（v2ではない）をダウンロード
```
ubuntu@rhel10:/usr/lib64/virt-p2v$ mkdir -p ~/p2v_el8 && cd ~/p2v_el8
ubuntu@rhel10:~/p2v_el8$ curl -L -o virt-p2v-maker.el8.rpm \
  https://dl.rockylinux.org/pub/rocky/8/AppStream/x86_64/os/Packages/v/virt-p2v-maker-1.42.0-5.el8.x86_64.rpm
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  235k  100  235k    0     0   315k      0 --:--:-- --:--:-- --:--:--  315k
```
バイナリを差し替え
```
ubuntu@rhel10:~/p2v_el8$ sudo cp -a ~/p2v_el8/usr/lib64/virt-p2v/virt-p2v.xz /usr/lib64/virt-p2v/virt-p2v.xz
ubuntu@rhel10:~/p2v_el8$ sudo ls -l /usr/lib64/virt-p2v/virt-p2v.xz /usr/lib64/virt-p2v/virt-p2v.xz.bak
-rw-r--r--. 1 ubuntu ubuntu 188856  4月 12  2021 /usr/lib64/virt-p2v/virt-p2v.xz
-rw-r--r--. 1 root   root   189308 11月  5  2024 /usr/lib64/virt-p2v/virt-p2v.xz.bak
```
バイナリが差し変わった状態で、fedora42でISOを作成
```
ubuntu@rhel10:~$ sudo virt-p2v-make-disk --output /tmp/virt-p2v.iso fedora-42
```
ISOができたら、USBに書き込み
```
ubuntu@rhel10:~/p2v_el8$ sudo umount /dev/sdb1 2>/dev/null || true
[sudo] ubuntu のパスワード:
ubuntu@rhel10:~/p2v_el8$ sudo dd if=/tmp/virt-p2v.iso of=/dev/sdb bs=4M status=progress conv=fsync
6442450944 bytes (6.4 GB, 6.0 GiB) copied, 627 s, 10.3 MB/s
1536+0 records in
1536+0 records out
6442450944 bytes (6.4 GB, 6.0 GiB) copied, 821.622 s, 7.8 MB/s
ubuntu@rhel10:~/p2v_el8$ sync
ubuntu@rhel10:~/p2v_el8$ sudo cmp -n 64M /tmp/virt-p2v.iso /dev/sdb && echo "OK"
[sudo] ubuntu のパスワード:
OK
```
※　単に `virt-p2v-make-disk --output /tmp/virt-p2v.iso fedora-42` だけだと、以下のエラーが出たため、上記の改造をしてクリアした

![IMG_1108](https://github.com/user-attachments/assets/c360269c-398b-48db-b88d-3e4166c78a42)

## virt-v2vホスト(almalinux8)

### VMを作成

[こちら](https://mirror.nishi.network/almalinux/8.10/isos/x86_64/)_からISOを取得

<img width="688" height="363" alt="image" src="https://github.com/user-attachments/assets/082debed-3d85-4713-9a5f-efff2e006509" />

ProxmoxにてISOからVMを作成

起動後、IPが付いておらずネットと疎通できないため、リンクを上げる

<img width="879" height="79" alt="image" src="https://github.com/user-attachments/assets/13a20a62-f91f-4f38-ad11-a70305c47e28" />

上記でDHCPでIPが付くため、同VLANからパスワードSSHできるようになる

以下で固定IP付与
```
[root@alma8 ~]# CON=enp6s18
[root@alma8 ~]# sudo nmcli con mod "$CON" ipv4.method manual \
>   ipv4.addresses 10.20.30.43/24 \
>   ipv4.gateway 10.20.30.1 \
>   ipv4.dns "10.20.30.1 8.8.8.8" \
>   ipv4.ignore-auto-dns yes
[root@alma8 ~]# sudo nmcli con mod "$CON" connection.autoconnect yes
[root@alma8 ~]# sudo nmcli con up "$CON"
```

### CentOS6にパスワードなしでSSHできるようにしておく

```
[root@alma8 ~]# cat .ssh/config
Host centos6
    Hostname 10.20.30.40
    User root
    IdentityFile ~/.ssh/id_rsa
```
```
[root@alma8 ~]# ssh-copy-id centos6
```

### virt-v2v等をインストール

```
[root@alma8 ~]# sudo dnf -y install virt-v2v libguestfs-tools-c libvirt-client　virt-p2v-maker
```
```
[root@alma8 ~]# virt-v2v --version
virt-v2v 1.42.0rhel=8,release=22.module_el8.9.0+3659+9c8643f3
```
落ちてるので、あげておく
```
[root@alma8 ~]# systemctl start libvirtd
```
CentOS6へ接続確認　OK
```
[root@alma8 ~]# virsh -c qemu+ssh://root@centos6/system list --all
 Id   名前          状態
----------------------------------
 -    almalinux10   シャットオフ
```


## P2V対象側(CentOS5.11)チェック

### KVMホスト(CentOS6.10)とvirt-v2vホスト(almalinux8)にパスワードなしでSSHできるようにしておく

<img width="465" height="372" alt="image" src="https://github.com/user-attachments/assets/2d3fbff4-2286-4504-a658-5699ab6854d0" />


## 受け入れ側(CentOS6.10)チェック

- ✅KVM/Libvirtは動いている
<img width="629" height="199" alt="image" src="https://github.com/user-attachments/assets/7bd95b2e-327c-4a83-ab23-7b25a2b4b1bf" />

- ✅受け入れられるだけの容量がある

CentOS5.11は、1.6G

<img width="643" height="177" alt="image" src="https://github.com/user-attachments/assets/e1be2093-7394-444b-bcbe-f37ebba125dd" />

CentOS6.10は、/ に40GB残ってる

<img width="657" height="222" alt="image" src="https://github.com/user-attachments/assets/0aa8bad5-d848-4aaf-9e55-e86e936cd961" />



















## centos6-8300 に保存先（ストレージ）を用意
```
[root@centos6-8300 images]# pwd
/var/lib/libvirt/images
[root@centos6-8300 images]# ls
CentOS-6.10-DVD1.iso  almalinux10.img
[root@centos6-8300 images]# mkdir p2v
```

## virt-p2v ISO　からCentOS5を起動

libpcre.so.1 がない　というエラー

<img width="1536" height="1152" alt="image" src="https://github.com/user-attachments/assets/5b8136ff-36a8-4089-9a9c-a6443febac33" />

`pcre` をインストールすれば、`libpcre.so.1` が手に入るらしいので、インストールする

![IMG_1126](https://github.com/user-attachments/assets/836ae5c8-9229-4933-a46d-dd8879cc1337)

インストール後、`launch-virt-p2v` でGUI起動

![IMG_1127](https://github.com/user-attachments/assets/0c37f3be-20de-4e83-bc40-9696c43a2144)

変換サーバ(rhel10)の virt-v2v が新しすぎて弾かれた　→　※なので今回alma8を用意した↑

<img width="640" height="480" alt="image" src="https://github.com/user-attachments/assets/e1b4407c-6f28-43ce-b0ee-8984d054ee64" />

alma8のvirt-v2vには接続成功

<img width="640" height="480" alt="image" src="https://github.com/user-attachments/assets/c0d0b6ae-ac49-4cf3-b3e2-b4c5558286bc" />

Next ボタンクリックで、以下の画面になる

<img width="640" height="480" alt="image" src="https://github.com/user-attachments/assets/3895f666-3fbc-4554-8867-7e188f16fd2e" />

以下にして、Start Conversion をクリック　一部見えないが、Output connは `qemu+ssh://root@centos6/system`

<img width="640" height="480" alt="image" src="https://github.com/user-attachments/assets/99a8f8c8-d6ce-4253-a75b-6d6b9d16c982" />

エラー終了

<img width="1536" height="1152" alt="image" src="https://github.com/user-attachments/assets/d97c457e-4b34-496a-b803-91502ce71adf" />

指定されたエラーログを見ると、以下の文字
```
virt-v2v: error: cannot find libvirt pool ‘default’: ストレージプールが見つかりません: 一致する名前 'default' を持つストレージプールがありません
```
しかし確かにdefaultプールはある
```
[root@alma8 ~]# virsh -c qemu+ssh://root@centos6/system pool-list --all
 名前      状態     自動起動
--------------------------------
 default   動作中   はい (yes)
```

alma8 ローカルにも defaultを作る
```
sudo virsh -c qemu:///system pool-define-as default dir - - - - /var/lib/libvirt/images
sudo virsh -c qemu:///system pool-start default
sudo virsh -c qemu:///system pool-autostart default
sudo virsh -c qemu:///system pool-list --all
```
```
[root@alma8 ~]# sudo virsh -c qemu:///system pool-list --all
 名前      状態     自動起動
--------------------------------
 default   動作中   はい (yes)
```

再度変換すると、成功

<img width="1536" height="1152" alt="image" src="https://github.com/user-attachments/assets/c1bd45e6-4c89-4bbd-b3df-b00609416aeb" />

なぜかalma8側にp2vされていたので、イメージを手動でCentOS6にコピーする
```
[root@alma8 ~]# scp /var/lib/libvirt/images/centos5-p2v-sda root@<<CentOS6のIP>>:/var/lib/libvirt/images/
```

起動
```
[root@centos6-8300 ~]# virsh start centos5-p2v
ドメイン centos5-p2v が起動されました
[root@centos6-8300 ~]#
```
接続する

以下のため、ポートは5900　※１なら5901
```
[root@centos6-8300 ~]# virsh vncdisplay centos5-p2v
:0
```
TigerVNCをダウンロード

https://sourceforge.net/projects/tigervnc/

VMにアクセス

<img width="568" height="218" alt="image" src="https://github.com/user-attachments/assets/8f703082-3656-4fea-bdda-4074e2bb9ac2" />

接続できたが、no bootable device のエラー

<img width="719" height="381" alt="image" src="https://github.com/user-attachments/assets/8a2eecbd-4a5c-4575-8121-cb126ff52f18" />

原因わかりました。CentOS6 側の qemu が “QCOW2 v3” を読めません。
```
[root@centos6-8300 ~]# qemu-img info /var/lib/libvirt/images/centos5-p2v-sda
'image' uses a qcow2 feature which is not supported by this qemu version: QCOW version 3
Could not open '/var/lib/libvirt/images/centos5-p2v-sda': Operation not supported
[root@centos6-8300 ~]#
```
最短の解決策：QCOW2 v2（compat=0.10）に変換して差し替え

VM停止
```
virsh shutdown centos5-p2v || virsh destroy centos5-p2v
```

Alma8側で修理　VMイメージを再度持ってくる
```
[root@alma8 ~]# mkdir -p /root/p2vfix
[root@alma8 ~]# cd p2vfix/
[root@alma8 p2vfix]# scp root@centos6:/var/lib/libvirt/images/centos5-p2v-sda .
centos5-p2v-sda                                                                       100% 1759MB 108.3MB/s   00:16
```
qcow2 v2へ変換
```
[root@alma8 p2vfix]# qemu-img convert -p \
>   -O qcow2 -o compat=0.10 \
>   centos5-p2v-sda centos5-p2v-sda.qcow2v2
    (100.00/100%)
[root@alma8 p2vfix]# qemu-img info centos5-p2v-sda.qcow2v2
image: centos5-p2v-sda.qcow2v2
file format: qcow2
virtual size: 112 GiB (120034123776 bytes)
disk size: 1.72 GiB
cluster_size: 65536
Format specific information:
    compat: 0.10
    compression type: zlib
    refcount bits: 16
[root@alma8 p2vfix]# 
```
修理済みイメージを戻す
```
[root@alma8 p2vfix]# scp centos5-p2v-sda.qcow2v2 root@centos6:/var/lib/libvirt/images/
centos5-p2v-sda.qcow2v2                                                               100% 1759MB 111.7MB/s   00:15
[root@alma8 p2vfix]#
```

CentOS6 側で所有者を合わせる：
```
[root@centos6-8300 ~]# chown qemu:qemu /var/lib/libvirt/images/centos5-p2v-sda.qcow2v2
[root@centos6-8300 ~]# chmod 644 /var/lib/libvirt/images/centos5-p2v-sda.qcow2v2
```
CentOS6側で読み込めているか確認
```
[root@centos6-8300 ~]# qemu-img info /var/lib/libvirt/images/centos5-p2v-sda.qcow2v2
image: /var/lib/libvirt/images/centos5-p2v-sda.qcow2v2
file format: qcow2
virtual size: 112G (120034123776 bytes)
disk size: 1.7G
cluster_size: 65536
[root@centos6-8300 ~]#
```
修正版を参照するように、以下2行だけ変更
```
[root@centos6-8300 ~]# virsh edit centos5-p2v
      <driver name='qemu' type='qcow2' cache='none'/>
      <source file='/var/lib/libvirt/images/centos5-p2v-sda.qcow2v2'/>
```
```
[root@centos6-8300 ~]# virsh start centos5-p2v
```

アクセスできた

<img width="901" height="538" alt="image" src="https://github.com/user-attachments/assets/d82d8be3-8fa6-461c-8e3b-1427d95b2b96" />
