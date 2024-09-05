# KeyclaokをSystemdのサービスとして起動する方法

Red Hat build of Keycloak (RHBK) 22 or 24 を使用するが、アーカイブやそれを解凍したディレクトリ名が異なるだけでコミュニティ版のKeycloakでも同様。

## HTTPSの使用について

Keycloakを開発モードで使うだけなら極めて簡単である。
すなわち解凍してできたディレクトリに移動して `bin/kc.sh start-dev` で起動し http://localhost:8080/ にアクセスするだけである（Java 17 or 21はインストール済みとする）。

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

公式ドキュメント: https://docs.redhat.com/ja/documentation/red_hat_build_of_keycloak/22.0/html/server_guide/enabletls-#enabletls-

## プロダクション環境に近い設定での起動方法

ここではアーカイブ解凍直後の状態から以下の設定で起動するまでの手順をしめす。

- HTTPSを使用（上記参照）
  - キーストアもしくはPEM形式で証明書を持っていること
- DBとしてPostgreSQLを使用
  - 既に起動済みでユーザ名、パスワード、JDBC URLの接続情報を持っていること
  - ローカル環境ならDocker or Podmanで `docker run -e POSTGRES_USER=keycloak -e POSTGRES_PASSWORD=password -e POSTGRES_DB=keycloak --name kcpostgres -p 5432:5432 -d docker.io/library/postgres:15` のように用意することも可能
- Infinispanキャッシュのクラスタリングの設定としてJDBC_PINGを使用
  - デフォルトではUDPマルチキャストを使うが、この場合マルチキャストの届く範囲で起動したKeycloakは全て一つのクラスタに参加しようとしてしまう
  - UDPマルチキャストの代わりにJDBC_PINGを設定し、上記で用意した同じPostgreSQLを使うよう設定する。この場合は同じDBを設定したKeycloakがクラスタメンバーとなる
- Systemdのサービスとして起動
- Keycloakの設定は環境変数を使用
  - Keycloakの設定項目はコマンドラインオプション、環境変数、conf/keycloak.confの順で優先順位を持つ（最初の方が優先で後の方の設定を上書きできる）
  - せっかくSystemdのユニットファイルを用意するので、必要な設定をそこで環境変数にて設定する
  - JDBC_PINGの設定を行うキャッシュ定義のXMLファイルはInfinispanの設定であってKeycloakの設定ではないのでコマンドラインオプションやkeycloak.confの内容を見ることはできないが、環境変数なら見れる
  - conf/keycloak.confを使っていけない訳では全くない
    - Systemd管理外で起動する場合に使うオプションを conf/keycloak.conf に書いておき、上書き設定したいものをユニットファイルの環境変数で設定するといった使い分けが可能
- レルムの自動インポートも可能
  - このレポジトリでは使っていないが、 data/import/ 内にレルムのJSONファイルを置くことで起動時に自動インポートさせることも可能
    - https://docs.redhat.com/ja/documentation/red_hat_build_of_keycloak/24.0/html-single/server_guide/index#importExport-importing-a-realm-during-startup

```shell
# root権限で作業する
$ sudo -i 

# SELinuxの制限回避のため /opt 以下を使用
$ mkdir /opt/keycloak
$ cd /opt/keycloak

# バージョン22やコミュニティ版でも可
$ unzip -q /path/to/rhbk-24.0.6.zip
$ ls /opt/keycloak/rhbk-24.0.6
bin  conf  lib  LICENSE.txt  providers  README.md  themes  version.txt

# ここがいわゆる KC_HOME になる。アップグレード時には /opt/keycloak 以下にKC_HOMEを増やしていくとよい
$ cd /opt/keycloak/rhbk-24.0.6

# 本リポジトリのファイルをコピー
$ cp conf/cache-ispn-jdbcping.xml conf/
$ cp conf/quarkus.properties conf/
$ cp keycloak.service .

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
Keycloak側はプロキシモードを "edge" もしくは "reencrypt" にする必要がある。
リバースプロキシとKeycloak間がHTTPになるのが "edge"、HTTPSになるのが "reencrypt" である。

- バージョン22で使用していた `proxy` オプションは24以降では非推奨になった
  - https://docs.redhat.com/ja/documentation/red_hat_build_of_keycloak/24.0/html/upgrading_guide/deprecated_and_removed_features#deprecated_literal_proxy_literal_option
- `health-enabled` および `metrics-enabled` を有効にしないなら /health, /metrics は機能しないのでブロックする必要はない
  - Keycloak 25からは management port が導入され8080ではなく9000でこれらのエンドポイントが提供されポート番号ベースのアクセス制限が可能
    - https://www.keycloak.org/docs/latest/release_notes/index.html#management-port-for-metrics-and-health-endpoints
- 管理コンソール (/admin) に対してはホスト名ベースのアクセス制限なら `hostname-admin` オプションで設定可能
  - https://docs.redhat.com/ja/documentation/red_hat_build_of_keycloak/24.0/html/server_guide/hostname-#hostname-administration-console

