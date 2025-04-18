---
title: "Proxmox に Elasticsearch, Kibana をインストールし、Suricata のログを閲覧する"
emoji: "🤖"
type: "tech"
topics: ["Proxmox", "Elasticsearch", "Suricata"]
published: false
---

# 本記事の目的
* Suricata によるパケット監視機能付き自作ルーターへ Elasticsearch と Kibana をインストールし、ログの視覚化を行う方法の紹介

# 操作サマリー
1. Proxmox 管理用 Web 画面を開く
1. Elasticsearch 稼働用コンテナの作成
1. Elasticsearch・Kibana インストールと設定・Proxmox コンソールを開く
    1. Elasticsearch コンテナのコンソールを開く
        1. wget, gpg, apt-transport-https インストール
        1. Elasticsearch インストール
        1. /etc/elasticsearch/elasticsearch.yml を編集
        1. /etc/elasticsearch/jvm.options を編集
        1. Elasticsearch 起動
        1. Kibana インストール
        1. /etc/kibana/kibana.yml を編集
        1. Kibana 起動
        1. Kibana 起動時のメッセージを確認
        1. Kibana 用 Elasticsearch アクセストークン生成
        1. Kibana Web 画面を開く
            * アクセストークン入力・Kibana セットアップを実行
    1. Elasticsearch コンテナのコンソールを閉じる
    1. Suricata コンテナのコンソールを開く
        1. wget, apt-transport-https インストール
        1. Filebeat インストール
        1. /etc/filebeat/filebeat.yml を編集
        1. /etc/filebeat/modules.d/suricata.yml を編集
        1. Filebeat Suricata 用モジュールを有効化
        1. Filebeat 自動起動設定
        1. Filebeat セットアップ
        1. Filebeat 起動
    1. Suricata コンテナのコンソールを閉じる
1. Proxmox コンソールを閉じる

## Elasticsearch コンテナでの作業

### Elasticsearch 稼働用コンテナ
* Suricata コンテナと一緒に LAN へ接続

![](/images/how_to_show_suricata_log_on_elasticsearch/1_network_connections.png)

### Elasticsearch インストール
* リポジトリのアドレスがそれなりに頻繁に変わる。毎回[こちら](https://www.elastic.co/docs/deploy-manage/deploy/self-managed/install-elasticsearch-with-debian-package)で使用するリポジトリが正しいか確認したほうがいい
```sh
> wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
> echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/9.x/apt stable main" | tee /etc/apt/sources.list.d/elastic-9.x.list
> apt update && apt install elasticsearch
```
* Elasticsearch のインストールが完了すると、コンソールへ以下のメッセージが表示される。スーパーユーザーパスワードは後で使うため控えておく
```sh
--------------------------- Security autoconfiguration information ------------------------------

Authentication and authorization are enabled.
TLS for the transport and HTTP layers is enabled and configured.

The generated password for the elastic built-in superuser is : fm-BCPUhcaprzaRIHZGI # ← このパスワードを使う

If this node should join an existing cluster, you can reconfigure this with
'/usr/share/elasticsearch/bin/elasticsearch-reconfigure-node --enrollment-token <token-here>'
after creating an enrollment token on your existing cluster.

You can complete the following actions at any time:

Reset the password of the elastic built-in superuser with 
'/usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic'.

Generate an enrollment token for Kibana instances with 
 '/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana'.

Generate an enrollment token for Elasticsearch nodes with 
'/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s node'.

-------------------------------------------------------------------------------------------------
### NOT starting on installation, please execute the following statements to configure elasticsearch service to start automatically using systemd
 sudo systemctl daemon-reload
 sudo systemctl enable elasticsearch.service
### You can start elasticsearch service by executing
 sudo systemctl start elasticsearch.service
```

### /etc/elasticsearch/elasticsearch.yml を編集
* 以下 4 箇所を修正
```sh
# Use a descriptive name for your cluster:
#
cluster.name: elastic # 任意の名前を指定する。こちらはシングルノード運用でも名前をつけたほうがいい
```
```sh
# By default Elasticsearch is only accessible on localhost. Set a different
# address here to expose this node on the network:
#
network.host: 0.0.0.0 # Elasticsearch コンテナ ens0 の IP を指定するエントリ・0.0.0.0 にしておけば、DHCP で IP が変わっても問題ない
```
```sh
# By default Elasticsearch listens for HTTP traffic on the first free port it
# finds starting at 9200. Set a specific HTTP port here:
#
http.port: 9200 # Elasticsearch API ポートの指定・アンコメントしなくても 9200 番でリクエストを受け付けていたが念の為
```
```sh
# Allow other nodes to join the cluster from anywhere
# Connections are encrypted and mutually authenticated
transport.host: 0.0.0.0 # シングルノード運用であればこの設定は不要
```

### /etc/elasticsearch/jvm.options を編集
* デフォルトではコンテナに割り当てたメモリを全て使い果たしてしまう。2 GB だけ余裕をもたせてサイズの指定を行う
* Kibana をどう使うか ( 何人で使うか, ダッシュボードの設定など ) によって開けておくメモリサイズを調整する。1 〜 数人で使う場合は Kibana 用のメモリは 512 MB でいい
* Xms と Xmx のサイズが異なるとエラーコード 78 でプロセスが終了してしまう。最大ヒープサイズと最小ヒープサイズは同じ値にする
```sh
################################################################
## IMPORTANT: JVM heap size
################################################################
##
## The heap size is automatically configured by Elasticsearch
## based on the available memory in your system and the roles
## each node is configured to fulfill. If specifying heap is
## required, it should be done through a file in jvm.options.d,
## which should be named with .options suffix, and the min and
## max should be set to the same value. For example, to set the
## heap to 4 GB, create a new file in the jvm.options.d
## directory containing these lines:
##
-Xms6g # この環境では elastic コンテナに 8 GB のメモリを割り当てた・Elasticsearch には 6 GB まで使用を許可する
-Xmx6g
```

### Elasticsearch 起動
```sh
systemctl daemon-reload
systemctl enable elasticsearch
systemctl start elasticsearch
```

### Kibana インストール
* Elasticsearch インストール時に追加したリポジトリで Kibana のインストールもできる

### /etc/kibana/kibana.yml を編集
```sh
# Specifies the address to which the Kibana server will bind. IP addresses and host names are both valid values.
# The default is 'localhost', which usually means remote machines will not be able to connect.
# To allow connections from remote users, set this parameter to a non-loopback address.
server.host: 0.0.0.0 # Kibana コンテナ ens0 の IP を指定するエントリ・0.0.0.0 にしておけば、DHCP で IP が変わっても問題ない。FQDN でも指定可能
```

### Kibana 起動
```sh
systemctl daemon-reload
systemctl enable kibana
systemctl start kibana
```

### Kibana 起動時のメッセージを確認
* Kibana 初回起動時に Web 画面で Elasticsearch アクセストークンを伝える必要がある。その画面を開くためのアドレスが出力される
```sh
systemctl status kibana
```
* 出力例
```sh
Apr 17 00:55:27 elastic kibana[698]: Go to http://0.0.0.0:5601/?code=900675 to get started. # クエリパラメータ code=900675 を知る必要がある
```

### Kibana 用 Elasticsearch アクセストークン生成
```sh
/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
```
* 出力例
```sh
eyJ2ZXIiOiI4LjE0LjAiLCJhZHIiOlsiMTkyLjE2OC42MS4xNzM6OTIwMCJdLCJmZ3IiOiJlNDhkZDdlMWMxZjUxYjYwMWYxNjQ2YzMwMjRlMzRlMjQ0NTNkNjQxMDQ0MzUxY2M1MmE0NmFjNDk2MjA4MWFjIiwia2V5IjoicEpkRVFaWUJTYWlRUVpJLTJjbEY6aWdSWDJpbmMxYXROYllFaF8taWVZQSJ9
```

### Kibana Web 画面を開く
![](/images/how_to_show_suricata_log_on_elasticsearch/2_register_access_token.png)
* Configure Elastic をクリックし、セットアップが完了した後、elastic ユーザーでログインが可能になる。elastic ユーザーのパスワードは手順 3-1-2 で得られた fm-BCPUhcaprzaRIHZGI

## ここから下は Suricata コンテナでの作業

### Filebeat インストール
* こちらもリポジトリのアドレスがそれなりに頻繁に変わる。[こちら](https://www.elastic.co/docs/reference/beats/filebeat/setup-repositories)で使用するリポジトリが正しいか確認したほうがいい
* たまにリポジトリのアドレスが間違えている。その場合は Elasticsearch のリポジトリをそのまま使う
```sh
> wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
> echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/9.x/apt stable main" | tee /etc/apt/sources.list.d/elastic-9.x.list
> apt update && apt install filebeat
```

### /etc/filebeat/filebeat.yml を編集
```sh
  # Kibana Host
  # Scheme and port can be left out and will be set to the default (http and 5601)
  # In case you specify and additional path, the scheme is required: http://localhost:5601/path
  # IPv6 addresses should always be defined as: https://[2001:db8::1]:5601
  host: "192.168.61.173:5601" # Kibana ( Elasticsearch ) コンテナ ens0 の IP アドレスを指定・FQDN でも可能
```
```sh
output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["https://192.168.61.173:9200"] # Elasticsearch コンテナ ens0 の IP アドレスを指定・FQDN でも可能

  # Performance preset - one of "balanced", "throughput", "scale",
  # "latency", or "custom".
  preset: balanced

  # Protocol - either `http` (default) or `https`.
  #protocol: "https"

  # Authentication credentials - either API key or username/password.
  #api_key: "id:api_key"
  username: "elastic" # アンコメント
  password: "fm-BCPUhcaprzaRIHZGI" # 手順 3-1-2 で得られた elastic ユーザー用パスワードを指定
  ssl.verification_mode: "none" # 行追加・自己署名証明書でのアクセスが可能なように設定
```

### /etc/filebeat/modules.d/suricata.yml を編集
* Elasticsearch へ送信するファイルのパスを指定する
```sh
- module: suricata
  # All logs
  eve:
    enabled: true # false → true へ変更

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    var.paths: ["/var/log/suricata/eve.json"] # アンコメント・Suricata が出力する eve.json のファイルパスを記入
```

### Filebeat Suricata 用モジュールを有効化
```sh
filebeat modules enable suricata
```

### Filebeat 自動起動設定・Filebeat セットアップ・Filebeat 起動
```sh
systemctl enable filebeat
filebeat setup
systemctl start filebeat
```
