---
title: Proxmox で自作したルーターのパケット監視に Suricata を使う
private: false
tags:
  - Proxmox
  - Suricata
  - OpenWRT
  - Open vSwitch
updated_at: '2025-04-16T14:16:25.022Z'
id: null
organization_url_name: null
slide: false
---

# 本記事の目的
* 自作ルーターとした OpenWRT の WAN 側 NIC に届くパケットを Suricata で監視できるようセットアップする手順の紹介
* 基本的な操作は[こちら](https://youtu.be/UXKbh0jPPpg?si=X9rVSSNYeraROK2W)を参照

https://youtu.be/UXKbh0jPPpg?si=X9rVSSNYeraROK2W

# 操作サマリー
1. Proxmox 管理用 Web 画面を開く
1. Suricata 稼働用コンテナの作成
1. Suricata インストールと設定・Proxmox コンソールを開く
    1. Suricata コンテナのコンソールを開く
        1. Suricata インストール
        1. /etc/suricata/suricata.yaml 修正
        1. Suricata ルールセットの取得
        1. 使用するルールの指定
        1. テストラン
    1. Suricata コンテナのコンソールを閉じる
1. Open vSwitch ポートミラーリング設定
1. Proxmox コンソールを閉じる
1. Suricata コンテナ再起動

## Suricata 稼働用コンテナ
* WAN 側 NIC のファイアーウォール機能は OFF・IP アドレスを取得しない設定にする。詳細は後述
* ネットワーク設定の IPv4, IPv6 を「静的」にすると IP アドレスを取得しない設定となる
* LAN 側 NIC は Suricata セットアップ & ルールの取得に使用する。図では省略

![](https://raw.githubusercontent.com/YuhichYOC/web-articles/main/images/suricata-in-proxmox/1_create_container.png)
![](https://raw.githubusercontent.com/YuhichYOC/web-articles/main/images/suricata-in-proxmox/2_turn_off_firewall.png)

## /etc/suricata/suricata.yaml 修正
* 以下の 5 箇所を修正
```sh
##
## Step 1: Inform Suricata about your network
##

vars:
  # more specific is better for alert accuracy and performance
  address-groups:
    #HOME_NET: "[192.168.0.0/16,10.0.0.0/8,172.16.0.0/12]"
    HOME_NET: "[192.168.1.0/16]" # 任意の IP アドレス範囲を指定
    #HOME_NET: "[10.0.0.0/8]"
    #HOME_NET: "[172.16.0.0/12]"
    #HOME_NET: "any"
```
```sh
      # Community Flow ID
      # Adds a 'community_id' field to EVE records. These are meant to give
      # records a predictable flow ID that can be used to match records to
      # output of other tools such as Zeek (Bro).
      #
      # Takes a 'seed' that needs to be same across sensors and tools
      # to make the id less predictable.

      # enable/disable the community id feature.
      community-id: true # true へ修正
```
```sh
##
## Step 3: Configure common capture settings
##
## See "Advanced Capture Options" below for more options, including Netmap
## and PF_RING.
##

# Linux high speed capture support
af-packet:
  - interface: eth0 # Suricata コンテナの WAN 側 NIC 名を指定
```
```sh
# Cross platform libpcap capture support
pcap:
  - interface: eth0 # Suricata コンテナの WAN 側 NIC 名を指定
```
```sh
# PF_RING configuration: for use with native PF_RING support
# for more info see http://www.ntop.org/products/pf_ring/
pfring:
  - interface: eth0 # Suricata コンテナの WAN 側 NIC 名を指定
```

## Suricata ルールセットの取得
```sh
suricata-update
```

## 使用するルールの指定
### 利用可能なルール一覧の取得
```sh
suricata-update list-sources
```
### 使用するルールの指定
* ライセンスが「Commercial」と表示されるルールは使用できない
```sh
suricata-update enable-source <ソース名>
```
### 再度 suricata-update を実行

## テストラン
* 設定に問題があれば教えてくれるらしい
```sh
suricata -T -c /etc/suricata/suricata.yaml -v
```

## ポートミラーリング設定
* OpenWRT の WAN 側 NIC から Suricata の WAN 側 NIC へ Open vSwitch のポートミラーリングでパケットを転送する。下の画像の赤矢印

![](https://raw.githubusercontent.com/YuhichYOC/web-articles/main/images/suricata-in-proxmox/3_configure.png)

### 方法 1 : コンテナの起動をフックスクリプトでフック

#### スニペットディレクトリ作成
![](https://raw.githubusercontent.com/YuhichYOC/web-articles/main/images/suricata-in-proxmox/4_create_snippets_directory_1.png)
![](https://raw.githubusercontent.com/YuhichYOC/web-articles/main/images/suricata-in-proxmox/4_create_snippets_directory_2.png)

#### フックスクリプト作成
```sh
root@pve:~# cat /var/lib/vz/snippets/snippets/hook_suricata.sh
#!/bin/bash

if [ "$2" == "post-start" ]; then
        sleep 10
        ovs-vsctl -- --id=@src get port ens1 -- --id=@out get port veth102i0 -- --id=@mirror create mirror name=ovsbr_w0m0 select-src-port=@src select-dst-port=@src output-port=@out -- set bridge ovsbr_wan0 mirrors=@mirror
fi
```

#### /var/lib/vz/snippets/snippets/hook_suricata.sh を実行可能にする

#### フックスクリプトを登録
* /etc/pve/lxc/<コンテナ ID>.conf へ以下の行を追記
```sh
hookscript: snippets:snippets/hook_suricata.sh
```

### 方法 2 : udev でインターフェース UP をフック

#### フックスクリプト作成
```sh
root@pve:~# cat /var/lib/vz/snippets/snippets/hook_veth102i0.sh
#!/bin/bash

sleep 10
ovs-vsctl -- --id=@src get port ens1 -- --id=@out get port veth102i0 -- --id=@mirror create mirror name=ovsbr_w0m0 select-src-port=@src select-dst-port=@src output-port=@out -- set bridge ovsbr_wan0 mirrors=@mirror
```

#### 設定を記入
* /etc/udev/rules.d/99-if.rules へ以下を記入
```
ACTION=="add", SUBSYSTEM=="net", KERNEL=="veth102i0", RUN+="/var/lib/vz/snippets/snippets/hook_veth102i0.sh"
```

### 補足 1・なぜ Suricata WAN 側 NIC でファイアウォールを OFF にするのか？
Proxmox コンテナでファイアウォールを ON にすると、Proxmox は仮想ブリッジを自動で作成し、その中にコンテナの仮想 NIC を登録します。下の図のイメージです。  
この場合、Open vSwitch のポートミラーリングで Suricata の WAN 側 NIC を指定できなくなり、パケットを転送しても Suricata コンテナに届かなくなります。  
ですので、ファイアウォール機能は OFF にして NIC の設定を行う必要があります。
![](https://raw.githubusercontent.com/YuhichYOC/web-articles/main/images/suricata-in-proxmox/5_when_firewall_exists.png)

### 補足 2・なぜ Suricata WAN 側 NIC が IP アドレスを取得しない設定にするのか？
IP アドレスのない NIC は、少なくとも能動的に通信することができません。  
補足 1 によりファイアウォールを OFF にする代わりに、通信機能をパケット転送による受信へ絞ることでセキュリティ面の担保を行う狙いです。  
※セキュリティのことはあまり詳しくないので、これが不足か過剰かいまいちわかっていません。

### 補足 3・フックスクリプトが 10 秒の待機時間を用意しているのはなぜか？
コンテナ起動 / インターフェース UP どちらをフックした場合でも、即座にポートミラーリング設定コマンドを実行するとミラー元のポートが存在せずミラーの作成に失敗します。  
見た目は不格好ですが、10 秒待つことで確実にミラー元のポートが UP している状態にすることが狙いです。

### 補足 4・Open vSwitch ポートミラーリング設定コマンドの構文
* コマンドは「--」で区切ることができる
```sh
ovs-vsctl \
    -- --id=@src get port <ミラー元ポート名> \
    -- --id=@out get port <ミラー先ポート名> \
    -- --id=@mirror create mirror name=ovsbr_w0m0 select-src-port=@src select-dst-port=@src output-port=@out \
    -- set bridge ovsbr_wan0 mirrors=@mirror 
```
* コマンドの内訳は以下の通り
1. OpenWRT WAN 側の物理ポート名を変数「@src」へ格納
1. Suricata WAN 側の物理ポート名を変数「@out」へ格納
1. ミラー「ovsbr_w0m0 ( 変数 : @mirror )」の定義・パケット送信元ポート = @src, パケットの宛先ポート = @src, パケット転送先ポート = @out
1. 仮想ブリッジ ovsbr_wan0 へミラー「@mirror」を設定

#### 転送元・転送先ポートの名前を調べるときは
* ovs-vsctl show コマンドを使う
* 戻り値の「101」「102」はコンテナ ID を示す・例の環境では 101 = OpenWRT, 102 = Suricata
```sh
a841c25b-28cb-490f-b4d8-9dc341fc8532
    Bridge ovsbr_wan0
        Port veth102i0
            Interface veth102i0
        Port ovsbr_wan0
            Interface ovsbr_wan0
                type: internal
        Port ens1
            Interface ens1
        Port fwln101o0
            Interface fwln101o0
                type: internal
    ovs_version: "3.1.0"
```
