---
title: Proxmox に OpenWRT をインストールし、ルーターとして使う
private: false
tags:
  - Proxmox
  - OpenWRT
  - Open vSwitch
updated_at: '2025-04-16T14:16:24.517Z'
id: null
organization_url_name: null
slide: false
---

# 本記事の目的
* 新品 PC へ Proxmox, Open vSwitch をインストールし、OpenWRT コンテナを稼働させ自作ルーターとする手順の紹介
* 以下の状態がゴール

![](https://raw.githubusercontent.com/YuhichYOC/web-articles/main/images/openwrt-in-proxmox/4_goal.png)
※ハードウェアの品番は動作確認を含めて併記しています

* 本記事は次の記事「[Proxmox で自作したルーターのパケット監視に Suricata を使う](https://zenn.dev/yuhichyoc/articles/suricata-in-proxmox)」へ続く

# 操作サマリー
1. Proxmox をインストールするマシンを仮の LAN へ接続
1. Proxmox セットアップ
1. Proxmox 初期設定, Open vSwitch インストール, OpenWRT コンテナ作成・Proxmox 管理用 Web 画面を開く
    1. Proxmox コンソールを開く
        * Open vSwitch インストール
    1. Proxmox コンソールを閉じる
    1. OVS ブリッジネットワーク作成, 名前 = ovsbr_wan0
    1. Linux ブリッジネットワーク作成, 名前 = vmbr_lan0
    1. Proxmox コンソールを開く
        * pct create コマンドにてコンテナ作成・まだ起動しない
    1. Proxmox コンソールを閉じる
    1. ovsbr_wan0 を名前 = eth0 で OpenWRT と接続させる
    1. vmbr_lan0 を名前 = lan0 で OpenWRT と接続させる
    1. OpenWRT コンテナ起動
    1. Proxmox コンソールを開く
        1. OpenWRT のコンソールを開く
            * /etc/config/network を編集
        1. OpenWRT のコンソールを閉じる
    1. OpenWRT の LAN 側 NIC へケーブルを接続
    1. コンテナ再起動
1. OpenWRT 初期設定・OpenWRT 管理用 Web 画面を開く
    1. 管理者パスワード設定
    1. SSH 公開鍵設定
    1. SSH 応答 NIC の設定
    1. SSH パスワード認証の無効化
    1. Network / Interfaces 設定ページを開く
    1. Network / DHCP and DNS 設定ページを開く
        1. General・ドメイン名設定
        1. Devices & Ports・Listen NIC 設定
        1. Devices & Ports・Exclude NIC 設定
        1. Forwards・DNS サーバーの追加
        1. Hostnames・ホスト設定
    1. System / System 設定ページを開く
        * General Settings・タイムゾーンの設定
1. OpenWRT 管理用 Web ページを閉じる
1. ルーター稼働用の Proxmox 設定・Proxmox 管理用 Web ページへ戻る
    1. Proxmox コンソールを開く
        1. OpenWRT コンソールを開く
            * /etc/config/network を編集
        1. OpenWRT コンソールを閉じる
        1. /etc/network/interfaces を編集
            * 管理用 NIC の固定 IP 設定を変更, OpenWRT の lan0 へ接続できるよう修正
        1. /etc/hosts を編集
            * 新ネットワークの IP アドレスを Proxmox へ割り当てる
    1. Proxmox コンソールを閉じる
    1. OpenWRT コンテナ自動起動設定
1. Proxmox VE 電源 OFF
1. LAN ケーブル接続変更
1. Proxmox VE 電源 ON

## 初期状態のケーブル接続
![](https://raw.githubusercontent.com/YuhichYOC/web-articles/main/images/openwrt-in-proxmox/1_initial.png)

## ovsbr_wan0 ブリッジネットワーク
以下の図での ens2 を含む仮想ブリッジが ovsbr_wan0 です  
WAN へ接続する NIC として使います
![](https://raw.githubusercontent.com/YuhichYOC/web-articles/main/images/openwrt-in-proxmox/2_create_openwrt_container.png)

## vmbr_lan0 ブリッジネットワーク
上の図での ens3, ens4, ens5 を含む仮想ブリッジが vmbr_lan0 です  
LAN 内の各機器と接続するためのものです

## OpenWRT コンテナ作成 ( pct create コマンドにてコンテナ作成 )
```sh
pct create <コンテナ ID> /var/lib/vz/template/cache/<Proxmox へ保存した OpenWRT のイメージ> --arch <ホストマシンのアーキテクチャ> --hostname <コンテナ名> --rootfs local-lvm:<使用するディスクサイズ> --memory <割り当てるメモリサイズ> --cores <割り当てる CPU コア数> --ostype unmanaged --unprivileged 1
```
Proxmox 管理用 Web 画面から作成することはできません。コミュニティ版のリポジトリには OpenWRT のイメージが無いためです  
非特権コンテナを作成します ( --unprivileged に 1 を渡す )。デフォルト値は 0 ( 特権コンテナ作成 ) です  
ovsbr_wan0 と vmbr_lan0 を接続した後起動します。この時点では起動しません

## OpenWRT の NIC : eth0
OpenWRT のデフォルト設定で WAN 側 NIC の名前は eth0 です。この設定を活かしてセットアップを進めます  
名前を変更するとインターネットへ接続するために設定を変更しなければなりません。かなり面倒 & eth0 から名前を変更する理由がないためこちらの名前は eth0 とします

## OpenWRT の NIC : lan0
こちらの名前は自由に設定できます  
/etc/config/network へ書き込む名前です

## OpenWRT の /etc/config/network 最初の記述 ( 手順 3-10-1 )
記入例は以下の通り・LAN 側 NIC の設定を追記する必要があります  
この時点ではセットアップ用仮 LAN へ接続するよう記入します
```
config interface 'lan'
        option proto 'dhcp'
        option netmask '255.255.255.0'
        option device 'lan0'
```

## OpenWRT 再起動前・ケーブル接続
OpenWRT 管理用 Web 画面を開くために OpenWRT LAN 側 NIC と接続します
![](https://raw.githubusercontent.com/YuhichYOC/web-articles/main/images/openwrt-in-proxmox/3_openwrt_setting.png)

## OpenWRT 管理用 Web ページ・Network / Interfaces ( 手順 4-5 )
開くだけで OK です。「設定を更新しますか？」と聞かれるので「YES」ボタンをクリックしましょう

## OpenWRT の /etc/config/network ルーター稼働前の記述 ( 手順 6-1-1 )
記入例は以下の通り・ゲートウェイとしての IP 設定を記入します
```
config interface 'lan'
        option proto 'static'
        option netmask '255.255.255.0'
        option ipaddr '192.168.1.1'
        option device 'lan0'
```

## Proxmox の /etc/network/interfaces
Proxmox 管理用 NIC ( ens1 ) の固定 IP 設定を変更する必要があります  
OpenWRT の lan0 へ接続できるよう修正しましょう

## Proxmox の /etc/hosts
/etc/network/interfaces の修正に合わせてこちらも修正する必要があります

## Proxmox 再起動前・ケーブル接続変更 ( 手順 8 )
以下の状態へケーブルを接続し直しましょう
![](https://raw.githubusercontent.com/YuhichYOC/web-articles/main/images/openwrt-in-proxmox/4_goal.png)
