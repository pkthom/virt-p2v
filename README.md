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



### P2V対象側()チェック

- ✅KVMホスト(CentOS6.10)にパスワードなしでSSHできる

<img width="649" height="267" alt="image" src="https://github.com/user-attachments/assets/c8a07bef-5eaf-41a8-b7cd-012f263f2180" />


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
