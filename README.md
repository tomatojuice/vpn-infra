# 🏠 Home VPN / Proxy Infrastructure

Rocky Linux 10 上に構築した、自宅向け VPN / Proxy 基盤です。  
WireGuard、Xray、Nginx、dnsmasq、Let's Encrypt、Cloudflare DDNS を組み合わせ、
Ansible で再構築できるようにしています。

This repository contains a self-hosted VPN / proxy infrastructure
running on Rocky Linux 10.

The environment combines WireGuard, Xray, Nginx, dnsmasq,
Let's Encrypt, Cloudflare DDNS, Docker, and Ansible.

---

## 🇯🇵 日本語

### 📌 概要

このプロジェクトでは、自宅の Rocky Linux サーバー上に
VPN / Proxy / DNS / TLS / DDNS 環境を構築しています。

主な構成要素:

- 🔐 WireGuard VPN
- 🧰 wg-easy
- 🌐 Xray
- 📡 VLESS + WebSocket + TLS
- 🔁 Nginx Reverse Proxy
- 🧭 dnsmasq / Split DNS
- 🌍 IPv6 ULA
- 🔒 Let's Encrypt
- ☁️ Cloudflare DNS
- 🔄 Cloudflare DDNS
- 🧱 firewalld
- 🐳 Docker / Docker Compose
- 🤖 Ansible
- ✅ 自動検証 Playbook

主な目的:

- 外出先から自宅ネットワークへ安全に接続
- 自宅回線経由でインターネットへ接続
- WireGuard と Xray を用途別に使い分け
- インフラ構成をコード化
- 再構築・復旧を容易にする
- Linux / Network / Docker / Ansible の学習

---

## 🗺️ Architecture

```text
                    Internet
                        │
                 Cloudflare DNS
                        │
                  Dynamic IPv4
                        │
                    Home Router
                ┌───────┴───────┐
                │               │
           UDP 51820         TCP 443
                │               │
           WireGuard           Nginx
                │          ┌────┴──────────────┐
             wg-easy       │                   │
                         wg-admin.*          xray.*
                            │                   │
                     127.0.0.1:51821      /secret-tunnel
                                                │
                                         127.0.0.1:10080
                                                │
                                               Xray
                                                │
                                             Internet
```

LAN 内では dnsmasq による Split DNS を使用します。

```text
Public DNS
xray.example.com
        │
        └─> Public IPv4

LAN DNS
xray.example.com
        │
        └─> Local Server IP
```

これにより Hairpin NAT に依存せず、
LAN 内外で同じドメイン名を使用できます。

---

# 🔐 WireGuard

WireGuard は `wg-easy` を Docker で実行しています。

### Public Port

```text
UDP 51820
```

### Admin UI

wg-easy 管理画面は外部へ直接公開しません。

```text
127.0.0.1:51821
```

Nginx を経由して HTTPS でアクセスします。

```text
https://wg-admin.example.com
        │
        ▼
      Nginx
        │
        ▼
127.0.0.1:51821
        │
        ▼
     wg-easy
```

管理画面は LAN 内からのみアクセス可能な構成です。

### 主な用途

- 🏠 自宅 LAN へのアクセス
- 🖥️ 自宅サーバーへの接続
- 🌐 Full Tunnel VPN
- 📱 PC / Smartphone / Mac からの接続

---

# 🌐 Xray

Xray は WireGuard とは別系統の Proxy として構築しています。

使用プロトコル:

```text
VLESS
+
WebSocket
+
TLS
```

通信経路:

```text
Client
  │
TCP 443
  │
  ▼
Nginx
  │
  ▼
/secret-tunnel
  │
  ▼
127.0.0.1:10080
  │
  ▼
Xray
  │
  ▼
Internet
```

Xray 自身では TLS 終端を行いません。

```text
TLS        → Nginx
VLESS      → Xray
WebSocket  → Xray
```

Xray の内部ポート `10080` は、
インターネットへ直接公開していません。

### Client UUID

各端末には個別 UUID を割り当てています。

例:

```text
Android
iPhone
Windows
MacBook
```

端末ごとに UUID を分離することで、
特定端末だけアクセスを無効化できます。

---

# 🔁 Nginx

Nginx は Reverse Proxy と TLS 終端を担当します。

同じ TCP 443 を利用しながら、
Host / SNI によってサービスを振り分けます。

```text
wg-admin.example.com
        │
        └─> wg-easy :51821

xray.example.com
        │
        └─> Xray :10080
```

---

# 🧭 Split DNS

LAN 内では dnsmasq を使用して、
公開ドメインを自宅サーバーの LAN IP に直接解決します。

例:

```text
wg-admin.example.com
        → 192.168.x.x

xray.example.com
        → 192.168.x.x

vpn.example.com
        → 192.168.x.x
```

LAN 外では通常の Public DNS が使用されます。

```text
xray.example.com
        → Public IPv4
```

これにより、自宅ルーターの Hairpin NAT に依存しません。

---

# 🌍 IPv6 ULA

LAN 内では IPv6 ULA を利用しています。

例:

```text
fdxx:xxxx:xxxx:1::/64
```

dnsmasq は IPv6 Router Advertisement の補助も行います。

サーバー自身は IPv6 のデフォルトルーターとしては使用しません。

---

# ☁️ Cloudflare DDNS

自宅回線の IPv4 アドレスは動的なため、
Cloudflare DNS を自動更新します。

systemd timer から定期実行します。

```text
Current Public IPv4
        │
        ▼
Cloudflare DNS
        │
        ▼
Compare
        │
   ┌────┴─────┐
   │          │
 Same      Changed
   │          │
 No update   Update
```

対象例:

```text
vpn.example.com
xray.example.com
```

---

# 🔒 TLS / Let's Encrypt

Let's Encrypt 証明書は Certbot で取得します。

Cloudflare DNS-01 Challenge を使用しています。

対象例:

```text
wg-admin.example.com
xray.example.com
```

証明書は Nginx で利用します。

---

# 🧱 Firewall

firewalld を使用して、
外部公開するポートを最小限にしています。

### Public

```text
TCP 443
UDP 51820
```

### Internal Only

```text
TCP 51821   wg-easy Admin
TCP 10080   Xray Backend
```

これらの内部ポートは直接インターネットへ公開しません。

---

# 🤖 Ansible

この構成は Ansible で自動構築できます。

## 📁 Directory Structure

```text
vpn-infra/
├── inventory/
│   ├── hosts.ini
│   └── group_vars/
│       └── all/
│           ├── main.yml
│           └── vault.yml
│
├── playbooks/
│   ├── 01_base.yml
│   ├── 02_network.yml
│   ├── 03_docker.yml
│   ├── 04_wg_easy.yml
│   ├── 05_xray.yml
│   ├── 06_dnsmasq.yml
│   ├── 07_certbot.yml
│   ├── 08_nginx.yml
│   ├── 09_ddns.yml
│   ├── 10_firewalld.yml
│   ├── 11_verify.yml
│   └── site.yml
│
├── templates/
│   ├── cloudflare-ddns.service.j2
│   ├── cloudflare-ddns.sh.j2
│   ├── cloudflare-ddns.timer.j2
│   ├── cloudflare.ini.j2
│   ├── ddns.env.j2
│   ├── dnsmasq.conf.j2
│   ├── wg-easy-compose.yml.j2
│   ├── wg-easy.conf.j2
│   ├── xray-compose.yml.j2
│   ├── xray-config.json.j2
│   └── xray.conf.j2
│
├── .gitignore
└── README.md
```

---

## 📚 Playbooks

| Playbook | Purpose |
|---|---|
| `01_base.yml` | OS 基本設定 |
| `02_network.yml` | Network 設定 |
| `03_docker.yml` | Docker 導入 |
| `04_wg_easy.yml` | WireGuard / wg-easy |
| `05_xray.yml` | Xray |
| `06_dnsmasq.yml` | Split DNS / IPv6 RA |
| `07_certbot.yml` | Let's Encrypt |
| `08_nginx.yml` | Reverse Proxy / TLS |
| `09_ddns.yml` | Cloudflare DDNS |
| `10_firewalld.yml` | Firewall |
| `11_verify.yml` | 構築後の確認 |
| `site.yml` | 全 Playbook 実行 |

---

## ▶️ Deploy

全体実行:

```bash
ansible-playbook \
  -i inventory/hosts.ini \
  playbooks/site.yml
```

個別実行例:

```bash
ansible-playbook \
  -i inventory/hosts.ini \
  playbooks/05_xray.yml
```

---

# ♻️ Idempotency

Ansible は冪等性を意識して作成しています。

設定変更後の初回実行では:

```text
changed > 0
```

変更がない状態で再実行した場合:

```text
changed=0
failed=0
```

となることを確認しています。

---

# ✅ Verification

`11_verify.yml` では、
主に以下を確認します。

- WireGuard Container
- Xray Container
- Nginx
- dnsmasq
- firewalld
- Cloudflare DDNS Timer
- WireGuard Port
- wg-easy Admin Port
- Xray Backend Port
- DNS
- TLS Certificates
- IPv4 Forwarding

---

# 🔑 Secrets

このリポジトリには秘密情報を含めません。

`.gitignore` で以下を除外しています。

```text
vault.yml
*.vault
*.key
*.pem
*.env
cloudflare.ini
wg0.conf
*.conf
*.db
```

GitHub に公開してはいけない情報:

- 🔐 Cloudflare API Token
- 🔐 WireGuard Private Key
- 🔐 WireGuard Preshared Key
- 🔐 Xray UUID
- 🔐 TLS Private Key
- 🔐 wg-easy Database

秘密情報は Ansible Vault 等で管理します。

```text
inventory/group_vars/all/vault.yml
```

---

# 🛠️ Tested Environment

```text
OS:
Rocky Linux 10

Architecture:
x86_64

Platform:
Intel N100 Mini PC

Container:
Docker
Docker Compose

Automation:
Ansible
```

---

# 📖 Learning Topics

このプロジェクトでは以下を実践しています。

- Linux Server Administration
- Network Configuration
- IPv4 / IPv6
- VPN
- Reverse Proxy
- TLS
- DNS
- Split DNS
- Dynamic DNS
- Docker
- Docker Compose
- Ansible
- Infrastructure as Code
- Firewall
- Service Monitoring

---

# 🇺🇸 English

## 📌 Overview

This project provides a self-hosted VPN and proxy infrastructure
running on Rocky Linux 10.

The stack includes:

- WireGuard
- wg-easy
- Xray
- VLESS
- WebSocket
- TLS
- Nginx
- dnsmasq
- Split DNS
- IPv6 ULA
- Let's Encrypt
- Cloudflare DNS
- Cloudflare DDNS
- firewalld
- Docker
- Docker Compose
- Ansible

The infrastructure is fully reproducible with Ansible.

---

## 🎯 Goals

The main goals of this project are:

- Secure remote access to the home network
- Internet access through the home connection
- Separate WireGuard and Xray use cases
- Infrastructure as Code
- Easy rebuilding and recovery
- Practical Linux and networking education

---

## 🔐 WireGuard

WireGuard is managed using wg-easy running inside Docker.

Public access:

```text
UDP 51820
```

The administrative interface is bound only to:

```text
127.0.0.1:51821
```

and accessed through Nginx.

---

## 🌐 Xray

Xray runs in a separate Docker Compose project.

Protocol stack:

```text
VLESS
+
WebSocket
+
TLS
```

Traffic flow:

```text
Client
  │
TCP 443
  │
Nginx
  │
WebSocket
  │
Xray
  │
Internet
```

TLS termination is handled by Nginx.

Each client uses an individual UUID,
making it possible to revoke access per device.

---

## 🧭 Split DNS

dnsmasq provides internal DNS overrides.

Inside LAN:

```text
xray.example.com
        → Local IP
```

Outside LAN:

```text
xray.example.com
        → Public IPv4
```

This removes the dependency on router Hairpin NAT support.

---

## ☁️ Dynamic DNS

A custom Cloudflare DDNS script updates DNS records
whenever the public IPv4 address changes.

The script is executed periodically using systemd timer.

---

## 🔒 TLS

Let's Encrypt certificates are obtained with Certbot
using the Cloudflare DNS-01 challenge.

Nginx uses these certificates for HTTPS
and Xray TLS termination.

---

## 🧱 Firewall

Only required services are publicly exposed.

Public:

```text
TCP 443
UDP 51820
```

Internal backends:

```text
TCP 51821
TCP 10080
```

Internal backend ports are not directly exposed to the Internet.

---

## 🤖 Ansible

The infrastructure is fully managed using Ansible.

Run the complete deployment:

```bash
ansible-playbook \
  -i inventory/hosts.ini \
  playbooks/site.yml
```

Run a single component:

```bash
ansible-playbook \
  -i inventory/hosts.ini \
  playbooks/05_xray.yml
```

---

## ♻️ Idempotency

The playbooks are designed to be idempotent.

When no changes are required:

```text
changed=0
failed=0
```

---

## 🔑 Secret Management

Sensitive data is intentionally excluded from Git.

Examples:

- Cloudflare API Tokens
- WireGuard Private Keys
- WireGuard Preshared Keys
- Xray UUIDs
- TLS Private Keys
- wg-easy Database

Secrets should be stored in:

```text
inventory/group_vars/all/vault.yml
```

and protected using Ansible Vault.

---

## ⚠️ Disclaimer

This repository is primarily intended for:

- Personal infrastructure management
- Networking practice
- Linux learning
- Ansible automation learning

Use this project at your own risk.

---

## 📜 License

This project is intended primarily for personal and educational use.