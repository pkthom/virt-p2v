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
ubuntu@localhost:~$ sudo update-crypto-policies --set LEGACY
[sudo] ubuntu のパスワード:
Setting system policy to LEGACY
Note: System-wide crypto policies are applied on application start-up.
It is recommended to restart the system for the change of policies
to fully take place.
ubuntu@localhost:~$
```
※⚠上記だけでは解決せず、以下が必要になるが、上記がないと、以下をやっても　no kex algが出るので、上記は必要です

まだSSHできない　エラーが変わる　no hostkey alg

<img width="402" height="50" alt="image" src="https://github.com/user-attachments/assets/53363771-5a90-4395-a55e-e404cd1e2d89" />

```
root@localhost:/etc/ssh/sshd_config.d# cat 05-hostkey-legacy.conf
HostKey /etc/ssh/ssh_host_rsa_key
HostKeyAlgorithms +ssh-rsa
root@localhost:/etc/ssh/sshd_config.d# cat 10-centos5-compat.conf
Match Address <<CentOS5.11のIPアドレス>>
    PubkeyAcceptedAlgorithms +ssh-rsa
root@localhost:/etc/ssh/sshd_config.d# sshd -t
root@localhost:/etc/ssh/sshd_config.d# systemctl restart sshd
root@localhost:/etc/ssh/sshd_config.d#
```
-> SSHできるようになるので、鍵を置く

✅パスワードなしでSSHできるようになった

<img width="358" height="30" alt="image" src="https://github.com/user-attachments/assets/f839e7b4-349d-4795-a5a2-19a9210d8371" />


- ✖virt-p2vが入っている -> 入ってない

<img width="900" height="128" alt="image" src="https://github.com/user-attachments/assets/84fd799c-4426-4b8a-b13c-acb733c650e4" />

### virt-v2v実施側チェック(RHEL10)

- ✅KVMホスト(CentOS6.10)にパスワードなしでSSHできる

```
Host centos6-8300
    HostName 10.20.30.40
    User root
    IdentityFile ~/.ssh/id_rsa_centos6
    Port 22
    HostKeyAlgorithms +ssh-rsa
    PubkeyAcceptedAlgorithms +ssh-rsa
    IdentitiesOnly yes
    ServerAliveInterval 60
```
鍵はわざと古い形式で作らないと受け入れてくれない
```
ssh-keygen -t rsa -b 4096 -f ./id_rsa_centos6
```

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
sudo dnf -y install virt-v2v libguestfs-tools-c libvirt-client
```
<img width="722" height="212" alt="image" src="https://github.com/user-attachments/assets/41c7ed7e-9ce9-48ad-b4cc-b6809dda1c70" />


✅インストールできたことを確認
```
virt-v2v --version
guestfish --version
```
<img width="658" height="248" alt="image" src="https://github.com/user-attachments/assets/59fcb7ae-5cbc-4322-9147-8e88d41a43e1" />



