## 概要
本手順では、ドローン航路システムのAPI Gateway機能を提供するDockerコンテナの構築手順を説明する。

## 手順概要
### 1. 事前準備
本リポジトリを動作させるために必要な実行環境の準備を行う。

### 2. Docker構築
本リポジトリ（drone-proxy）をDockerコンテナとして起動する。

### 3. 動作確認
動作確認を実施する。

## 手順詳細

### 1. 事前準備

#### 1.1 実行環境準備
本リポジトリのコンテナを構築するために必須となるソフトウェアを事前にインストールする。
必要なソフトウェアおよび、動作確認済み実行環境は以下のとおりである。

| 名称 | バージョン | 備考 |
| --- | --- | --- |
| Docker | 25.0.5 |  |

### 2. Docker構築

以下の流れでDockerコンテナの作成、起動を行う。
- nginxの設定ファイルを修正する
- nginxの設定ファイルおよびコンテナイメージを用いてDockerコンテナを作成する

※本手順では、実行ユーザがdockerグループに所属している前提でコマンドを記載しています。
実行ユーザがdockerグループに所属していない場合、dockerコマンドに「sudo」を付与し、sudoが可能なユーザで実行する必要があります。

#### 2.1 nginxの設定ファイルの修正

#### ①nginx用設定ファイルの配置
ビルドに必要なリポジトリ資材を配置する。
本手順では、ホームディレクトリ配下に以下のパスで資材を配置している想定で手順を作成している。
- 本リポジトリ（drone-proxy）の格納先
```bash
~/work/drone-proxy
```

#### ②本リポジトリの「nginx-openresty.conf」に定義されている接続情報の変数を、試験環境に合わせて修正する。
| 変数 | 説明 | 備考 |
| --- | --- | --- |
| APIKEY | トークンイントロスペクション用APIキー |  |
| PROTOCOL | バックエンドサービスに送信するリクエストのプロトコル<br>`http`もしくは`https`を設定してください |  |
| DEGITAL_TWIN_DOMAIN | 空域デジタルツインのドメイン |  |
| AIRWAY_DESIGN_DOMAIN | 航路画定のドメイン |  |
| AIRWAY_RESERVATION_DOMAIN | 航路予約のドメイン |  |
| AIRWAY_RESERVATION_APIKEY  | 航路予約のAPIキー |  |
| SAFETY_DOMAIN | 安全管理のドメイン |  |
| ASSET_DOMAIN | ポート・機体管理のドメイン |  |
| EXTERNAL_DOMAIN | 外部システム連携のドメイン |  |
| USER_AUTH_DOMAIN  | ユーザ認証システムのドメイン |  |

#### 2.2 Dockerコンテナイメージ作成

#### ①ビルド用資材の配置
以下コマンドでnginx(OpenResty)のコンテナイメージを取得する。
```bash
docker pull openresty/openresty:1.27.1.1-0-bullseye-fat
```
※本手順では nginx(OpenResty) バージョン1.27.1.1にて動作を確認しております。

#### 2.3 Dockerコンテナ作成

#### ①Dockerコンテナ作成
```bash
cd ~/work/drone-proxy
docker run --detach --name drone-proxy -v ./nginx-openresty.conf:/usr/local/openresty/nginx/conf/nginx.conf -p 8081:8081 --log-driver local --log-opt max-size=500M --log-opt max-file=5 -d openresty/openresty:1.27.1.1-0-bullseye-fat nginx -g 'daemon off;'
```
※コンテナ起動時、ログはコンテナに出力されます。また、500MBでログのローテートが実行され5世代まで管理されます。

#### ②Dockerコンテナの起動確認
```bash
docker ps -a
```
以下のように、STATUSが「Up」となっていることを確認する。
```bash
CONTAINER ID   IMAGE                                         COMMAND                  CREATED         STATUS        PORTS                                               NAMES
5b6dda7d3c23   openresty/openresty:1.27.1.1-0-bullseye-fat   "nginx -g 'daemon of…"   2 seconds ago   Up 1 second   0.0.0.0:8081->8081/tcp, :::8081->8081/tcp           drone-proxy
```

### 3. 動作確認

#### 3.1 動作確認

前提：
- 外部システム連携コンテナが構築済みであること
- ユーザ認証システムコンテナが構築済みであること

#### ①ユーザ認証システムへのログイン（ユーザ当人認証API実行）

ユーザ認証システムのユーザ当人認証APIを実行し、アクセストークンが返却されることを確認する。

1. 以下のcurlを実行し、正常にレスポンスが返ってくることを確認する。

```bash
curl -X POST "https://{ユーザ認証システムのドメイン}/auth/login" -H "Content-Type: application/json" -H "apiKey: {払いだされたapiKey}" -d '{"operatorAccountId": "{払いだされたID}", "accountPassword": "{払いだされたパスワード}"}'
```

2. 以下のようなレスポンスが返却されることを確認する。
```bash
{"accessToken":"{払いだされたアクセストークン}","refreshToken":"{払いだされたリフレッシュトークン}"}
```

#### ②外部システム連携への疎通確認（DIPSトークン検証APIの実行）

外部システム連携のDIPSトークン検証APIを実行し、API Gatewayでのトークンイントロスペクションとルーティングが適切に行われていることを確認する。

1. 以下のcurlを実行し、正常にレスポンスが返ってくることを確認する。

```bash
curl -v http://{API Gatewayのドメイン}/external/api/v1/dipsTokenVerification -X POST -H "Authorization: Bearer {ユーザ認証システムより払いだされたアクセストークン}"
```

2. 以下のようなレスポンスが返却されることを確認する。
```bash
{"message":"TokenInfo Not Found"}
```
※HTTPステータスコードは400となる。clientIdに紐づくトークン情報が見つからない旨のメッセージが返却されるが、DIPSトークン検証APIが返却するメッセージであり、外部システム連携との疎通確認はできているため問題ない。