# KeyclaokをSystemdのサービスとして起動する方法

Red Hat build of Keycloak (RHBK) 26.0 を使用するが、アーカイブやそれを解凍したディレクトリ名が異なるだけでコミュニティ版のKeycloakでも同様。
なお、Keycloak 26 からバージョニングが変わり、約3ヶ月毎にマイナーバージョンが上がるようになった([Backwards compatibility in Keycloak releases](https://www.keycloak.org/2024/10/release-updates))。
製品版のRHBKは偶数マイナーバージョンが採用され、26.0の次が26.2である。

## HTTPSの使用について

Keycloakを開発モードで使うだけなら極めて簡単である。
すなわち解凍してできたディレクトリに移動して `bin/kc.sh start-dev` で起動し http://localhost:8080/ にアクセスするだけである
（Java (OpenJDKもしくはTemurinが[supported](https://access.redhat.com/articles/7033107)) 17 or 21 はインストール済みとする）。

しかしプロダクションモード（`start-dev`の代わりに`start`を使う）の場合はHTTPSを使おうとするため事前に準備が必要である。
HTTPSではなくHTTPを使うようにコマンドラインオプションを追加すれば、解凍直後の状態でもプロダクションモードで起動することは一応可能である。

```shell
bin/kc.sh start --http-enabled=true --hostname-strict=false
```

これで開発モードと同様 http://localhost:8080/ でアクセスできるが、プロダクションモードではInfinispanのキャッシュのタイプが
localからリモート（すなわちdistributed-cacheかreplicated-cache）になるという違いがある。

HTTPSを使用する場合は証明書を用意する必要があるが、自己署名証明書でいいなら以下のコマンドで簡単に作成できる。

```shell
# JDK付属のkeytoolを使用して証明書を作成し、キーストア conf/server.keystore にパスワード password で格納
keytool -genkeypair -storepass password -storetype PKCS12 -keyalg RSA -keysize 2048 -dname "CN=server" -alias server -ext "SAN:c=DNS:localhost,IP:127.0.0.1" -keystore conf/server.keystore

# 証明書のSAN (Subject Alt Name) に設定したホスト名を指定して起動
bin/kc.sh start --hostname=localhost
```

これで https://localhost:8443/ にてHTTPSでアクセスできるようになる。
Javaのキーストアのファイルパスとそのパスワードをデフォルト値で作成しているので `hostname` の他の設定は何も必要ない。

キーストアではなくPEM形式で証明書を持っているのなら、対応する秘密鍵とともに明示的な設定が必要である。

```shell
bin/kc.sh start --https-certificate-file=/path/to/certfile.pem --https-certificate-key-file=/path/to/keyfile.pem --hostname=<証明書のSANに設定したホスト名>
```

公式ドキュメント: https://docs.redhat.com/ja/documentation/red_hat_build_of_keycloak/26.0/html-single/server_configuration_guide/index#enabletls-

## プロダクション環境に近い設定での起動方法

ここではアーカイブ解凍直後の状態から以下の設定で起動するまでの手順をしめす。

- HTTPSを使用（上記参照）
  - キーストアもしくはPEM形式で証明書を持っていること
- DBとしてPostgreSQLを使用
  - 既に起動済みでユーザ名、パスワード、JDBC URLの接続情報を持っていること
  - ローカル環境ならDocker or Podmanで `docker run -e POSTGRES_USER=keycloak -e POSTGRES_PASSWORD=password -e POSTGRES_DB=keycloak --name kcpostgres -p 5432:5432 -d docker.io/library/postgres:16` のように用意することも可能
- Systemdのサービスとして起動
- Keycloakの設定は keycloak.conf を使用
  - Keycloakの設定項目はコマンドラインオプション、環境変数、conf/keycloak.confの順で優先順位を持つ（最初の方が優先で後の方の設定を上書きできる）
- レルムの自動インポートも可能
  - data/import/ 内にレルムのJSONファイルを置き、`--import-realm` オプションを追加することで起動時に自動インポートさせることも可能
    - https://docs.redhat.com/ja/documentation/red_hat_build_of_keycloak/26.0/html-single/server_configuration_guide/index#importExport-importing-a-realm-during-startup
  - 同名のレルムが存在する場合はインポートしない
  - レルムのJSONファイルは部分的なものでもよく、またパスワードは平文であってもよい
    - 例: https://github.com/keycloak/keycloak/blob/18.0.2/examples/js-console/example-realm.json

```shell
# root権限で作業する
$ sudo -i

# SELinuxの制限回避のため /opt 以下を使用
$ mkdir /opt/keycloak
$ cd /opt/keycloak

# カスタマーポータルよりダウンロードしたRHBKのアーカイブを解凍(コミュニティ版のkeycloak-*.zipでも同様)
$ unzip -q /path/to/rhbk-26.0.9.zip
$ ls /opt/keycloak/rhbk-26.0.9
bin  conf  lib  LICENSE.txt  providers  README.md  themes  version.txt

# ここがいわゆる KC_HOME になる。アップグレード時には /opt/keycloak 以下にKC_HOMEを増やしていくとよい
$ cd /opt/keycloak/rhbk-26.0.9

# 本リポジトリのファイルをコピー
$ cp /path/to/conf/* conf/
$ cp /path/to/keycloak.service .

# カスタマイズがある場合はそれらもコピー
$ cp /path/to/*.jar providers/

# 証明書をコピー
cp /path/to/server.keystore conf/
# 上記はキーストアの場合。PEM形式の場合はconf以下でなくてもよいがkeycloak.serviceを併せて編集する

# ユニットファイルを編集して微調整
$ vi keycloak.service

# 専用のユーザ・グループがあるなら所有権を変更
$ chown -R keycloak:keycloak .

# ユニットファイルをインストール
$ systemctl enable ./keycloak.service

# 起動
$ systemctl start keycloak

# ログを監視
$ journalctl -fu keycloak

# ユニットファイル編集後はリロードが必要
$ vi keycloak.service
$ systemctl daemon-reload
```

## 本当のプロダクション環境での考慮点

Quarkus版のKeycloakはWildFly版にはあったリクエストパスやIPアドレスによるアクセス制限ができないため、
/admin, /health, /metrics といったパスのアクセス制限を行うために前段にリバースプロキシを起き、
HTTPSを一旦ほどいて[パス名によるアクセス制限](https://docs.redhat.com/ja/documentation/red_hat_build_of_keycloak/26.0/html-single/server_configuration_guide/index#reverseproxy-exposed-path-recommendations)を行う必要がある。

リバースプロキシ・Keycloak間をHTTPで済ませるなら `http-enabled=true` を指定してHTTPSではなくHTTPを使うようにする。
この時ブラウザのIPアドレスを伝えるためにリバースプロキシが使用するヘッダーに合わせて `proxy-headers=forwarded` のような
[設定](https://docs.redhat.com/ja/documentation/red_hat_build_of_keycloak/26.0/html-single/server_configuration_guide/index#reverseproxy-)も追加する。

- `health-enabled` および `metrics-enabled` を有効にしないなら /health, /metrics は機能しないのでブロックする必要はない
  - Keycloak 25からは management port が導入され8080ではなく9000でこれらのエンドポイントが提供されるようになったのでポート番号ベースのアクセス制限も可能になった
- 管理コンソール (/admin) に対してはホスト名ベースのアクセス制限なら `hostname-admin` オプションで設定可能
  - https://docs.redhat.com/ja/documentation/red_hat_build_of_keycloak/24.0/html/server_guide/hostname-#hostname-administration-console

## 古いログファイルの削除方法について

KeycloakのベースであるQuarkusが採用しているロギングライブラリであるJBoss Log Managerは
同一プレフィクスのファイル数を制限する機能 (max-backup-size) は持っているが、
（LogbackのmaxHistoryのような）古いファイルを削除する機能は持っていない。

[LOGMGR-139](https://issues.redhat.com/browse/LOGMGR-139) をはじめ古くから機能追加のリクエストは上がっているが、
ファイル名ベースの判断の不安定性やローテート時のロックの問題などを理由にCRONなど他の機構に頼る方がいいとしている。

Systemdの使える環境ではCRONよりもSystemd timerの方が使い勝手がいい（[参考URL](https://opensource.com/article/20/7/systemd-timers)）。
週次ベースで削除を行うユニットファイルを参考として置いてある。

```
systemctl enable ./keycloak-log-delete.service
systemctl enable ./keycloak-log-delete.timer
systemctl start keycloak-log-delete.timer
```
