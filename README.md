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



### P2V対象側チェック

- ✅KVMホスト(CentOS6.10)にパスワードなしでSSHできる

- ✖virt-p2vが入っている -> 入ってない

<img width="900" height="128" alt="image" src="https://github.com/user-attachments/assets/84fd799c-4426-4b8a-b13c-acb733c650e4" />
