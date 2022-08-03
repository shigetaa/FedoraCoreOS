# Remote Container を FedoraCoreOS に接続してコンテナ環境を利用する
ローカル開発環境でDocker デーモンを動かさず
Docker 専用ホストにてコンテナを動作環境を構築

コンテナを動かすだけのOSとして、Fedora CoreOS を使用
開発IDEとしてVisual Studio Code を使用して、拡張機能 Remote Contaners を使用して
Docker 専用ホスト(FedoraCoreOS)上でコンテナを使用する。

## Fedora Core OS をインストール
### インストール前に

大抵のLinux系OSインストーラはネットワークやユーザ設定等を対話式に進めていきますが、Fedora CoreOSは違います。
設定情報を予めIgnitionファイルと呼ばれるものに記述しておき、インストール時はIgnitionファイルを参照してセットアップが行われます

IgnitionファイルはJSON形式ですが、手で書くのはつらいので、YAML形式のFCCファイルからIgnitionファイルを生成する、FCCTというツールが公式で提供されています。

また今回はVMware+OVAでセットアップを行うため、isoインストーラを使用する場合と少し手順が異なります。

### OSセットアップ手順
流れとしてはざっくりこんなかんじです。
1. YAML 形式の Butane構成を生成します
2. Butane を実行して、YAML ファイルをIgnitionファイルの生成
3. 仮想マシン作成, 起動

以下の設定を行うのが今回の目標です。

- デフォルトで用意されているユーザ coreにSSHログインするための公開鍵設定
- ホスト名をfcos1にする
- 静的ネットワーク設定
  - NIC: ens192
  - IP: 192.168.220.77/24
  - Gateway: 192.168.220.1
  - Nameserver: 192.168.220.251

#### 1. YAML 形式の Butane 構成を生成します
ユーザーパスワードを Docker コンテナにて `password_hash` を生成して、YAML ファイルに記述すうる。
```bash
docker run -ti --rm quay.io/coreos/mkpasswd --method=yescrypt
```

- ホスト名を設定
- ネットワーク関連を設定
- SSH接続許可を設定
- ローカルタイムを設定
- docker を有効化する
- docker-compose をインストール

`config.bu`ファイル名として下記の様に記述する
```yml
variant: fcos
version: 1.4.0
passwd:
  users:
    - name: core
      groups:
        - docker
      password_hash: $y$j9T$VpN.6h9/DB/sIwiM3tj2W/$Ifv3gW23jPphG1JnYl/yzURqvg0NO4YOttcoRjV/fGD
storage:
  links:
    - path: /etc/localtime
      target: ../usr/share/zoneinfo/Japan
  files:
    - path: /etc/hostname
      mode: 0644
      contents:
        inline: fcos1
    - path: /etc/NetworkManager/system-connections/ens33.nmconnection
      mode: 0600
      contents:
        inline: |
          [connection]
          id=ens33
          type=ethernet
          interface-name=ens33
          [ipv4]
          address1=192.168.220.77/24,192.168.220.1
          dns=192.168.220.251
          dns-search=
          may-fail=false
          method=manual
    - path: /etc/ssh/sshd_config.d/20-enable-passwords.conf
      mode: 0644
      contents:
        inline: |
          # Fedora CoreOS disables SSH password login by default.
          # Enable it.
          # This file must sort before 40-disable-passwords.conf.
          PasswordAuthentication yes
systemd:
  units:
    - name: docker.service
      enabled: true
    - name: install-docker-compose.service
      enabled: true
      contents: |
        [Unit]
        Description=Install docker-compose on first boot
        DefaultDependencies=no
        Conflicts=shutdown.target
        After=network-online.target
        Before=shutdown.target
        ConditionFirstBoot=yes

        [Service]
        Type=oneshot
        RemainAfterExit=no
        ExecStart=/usr/bin/rpm-ostree install -r docker-compose

        [Install]
        WantedBy=network-online.target
```
#### 2. Butane を実行して、YAML ファイルをIgnitionファイルの生成
先述の通り、ButaneでButane→Ignitionに変換します。
Butaneは実行バイナリとコンテナイメージが配布されています。今回はDockerコンテナでいきます。

先程作成したFCCファイルを`config.bu`として保存し、以下のコマンドを実行します。
```bash
docker run -i --rm --security-opt label=disable --volume ${PWD}:/pwd --workdir /pwd quay.io/coreos/butane:release --pretty --strict < config.bu > install.ign
```
これでinstall.ignにIgnitionファイルが出力されます。
（isoインストーラを使用してOSインストールを行う場合は、このIgnitionファイルをHTTPで公開する必要があります。）



#### 3. 仮想マシン作成, 起動
ダウンロードページからVMware用のISOファイルを落とします。

Download Fedora CoreOS
https://getfedora.org/coreos/download?tab=metal_virtualized&stream=stable


httpsの場合
```bash
sudo coreos-installer install /dev/sda --ignition-url https://xxx/xxxx.ign
```
httpの場合
```bash
sudo coreos-installer install /dev/sda --ignition-url http://xxx/xxxx.ign  --insecure-ignition
```

## Visual Studio Code の設定
VSCode Remote Containers で開発環境を使用する場合は以下の2つをインストールが必要です。

- Visual Studio Code
- Docker Decktop for Windows or Mac

インストールを済ませると、`Visual Studio Code` を起動して、拡張機能の `Remote Development` をインストールします。

### SSH 接続設定
SSH接続する際には、`公開鍵`と`秘密鍵`があるとパスワード無しで接続できますので
まずは、公開鍵と秘密鍵を作成していきます。

リモート ホストに SSH キー ベースの認証を設定するには
最初にローカルで鍵ペアを作成し、公開鍵をホストにコピーします。

ローカルマシンに SSH キーが既にあるかどうかを確認します。
macOS / Linux では `.ssh`にあり。
Windows ではユーザー プロファイル フォルダーのディレクトリにあります (例: C:\Users\your-user\.ssh\id_ed25519.pub)

キーがない場合は、ローカルターミナル/PowerShell で次のコマンドを実行して、SSH キー ペアを生成します。
```bash
ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/home/core/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/core/.ssh/id_rsa.
Your public key has been saved in /home/core/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:DW0pky/X/Bn625RfDt3QDXWiS2zSQEebTD0noaQW4QM core@localhost
The key's randomart image is:
+---[RSA 2048]----+
|        E.=o=.o.o|
|         = % =+oo|
|        = X @ .+ |
|         O B . o.|
|        S + + o o|
|         o   o =o|
|            . +.=|
|             . =o|
|              o.+|
+----[SHA256]-----+
```

ローカル接続の承認する
ローカルターミナルウィンドウで次のコマンドのいずれかを実行し、
必要に応じてユーザー名とホスト名を置き換えて、ローカル公開キー`id_rsa.pub`を SSH ホストにコピーします。
```bash
$USER_AT_HOST="core@192.168.220.77"
$PUBKEYPATH="$HOME\.ssh\id_rsa.pub"

$pubKey=(Get-Content "$PUBKEYPATH" | Out-String); ssh "$USER_AT_HOST" "mkdir -p ~/.ssh && chmod 700 ~/.ssh && echo '${pubKey}' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```
以下の様に表示されるので
yes と入力して、パスワードを入力してください。
```bash
The authenticity of host '192.168.220.77 (192.168.220.77)' can't be established.
ECDSA key fingerprint is SHA256:xxxxxxxxxxxxxxxxxxxxxxxxxxx
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.220.77' (ECDSA) to the list of known hosts.
core@192.168.220.77's password:
```
ローカルターミナルで以下の様に入力してパスワード無しに接続出来たら設定完了です。
```bash
ssh $USER_AT_HOST
```
```bash
Fedora CoreOS 36.20220716.3.1
Tracker: https://github.com/coreos/fedora-coreos-tracker
Discuss: https://discussion.fedoraproject.org/tag/coreos

Last login: Tue Aug  2 16:22:34 2022 from 192.168.220.13
[core@fcos1 ~]$
```
### VSCode SSH設定
左下`><`をクリックして、`ホストに接続する`をクリックして`新規SSHホストを追加する`を実行して
`~/.ssh/config`に設定を保存する。

### VSCode 使用方法
左下`><`をクリックして、`ホストに接続する`をクリックして 上記で登録したホストを選択する。
別ウィンドウが表示されリモートの VSCode Server 上で　`Remote Containers` の拡張機能が利用できる。

#### Remote Containers 機能
- Open Folder in Container...
- Clone Repository in Container Volume..
- Attach to Running Container...
- Add Development Container Configuration Files...


### リモートホストのDocker Container 起動
起動例
```bash
docker run --secrity-opt label=disable --rm -p 8080:80 --name php8 -v $PWD:/var/www/html php:8.0-apache
```
`--secrity-opt label=disable` Option を使用しないと ボリュームをマウント時パーミッションエラーでボリュームを操作出来ない。

起動したコンテナに対して、Visual Studio Code で操作するには
Remote Containers の機能
`Attach to Running Container...` にて操作する事が可能になります。

### コードにてコンテナ構成管理
コードにてコンテナの構成、起動、等を管理する為
最小構成で以下の様なディレクトリ構成ファイルになります。
<pre>
.
├── .devcontainer
│   ├── devcontainer.json
│   └── docker-compose.yml
└── public
    └── index.html
</pre>

#### .devcontainer/devcontainer.json
Visual Studio Code のコンテナ開発の設定を行います。
```json
{
	"name": "PHP",
	"dockerComposeFile": [
		"docker-compose.yml"
	],
	"service": "php",
	"workspaceFolder": "/var/www/html",
	"extensions": [
		"felixfbecker.php-debug",
		"felixfbecker.php-intellisense"
	]
}
```
`service` に、接続するサービス名を指定します。
`dockerComposeFile` に、接続コンテナの設定ファイルを指定します。
`workspaceFolder` に、接続コンテナの 作業フォルダを指定します。
`extensions` に、接続コンテナにインストールする拡張機能を指定します。


#### .devcontainer/docker-compose.yml
開発に利用するコンテナを設定をおこないます。
```yml
version: '3'

services:
  php:
    image: php:8.0-apache
    ports:
      - '8080:80'
    volumes:
      - ../public:/var/www/html:Z

```
`services` は、作成するコンテナを定義します。
`php` は、コンテナ名としてphp（名前自由）を定義してます。
`image` は、コンテナイメージ
`ports` は、公開ポート
`volumes` は、ボリュームを指定します。ここでは、行末に `:Z` を記述してますがSELinuxの兼ね合いで記述してます。

#### public/index.html
公開するファイルなど
```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta http-equiv="X-UA-Compatible" content="IE=edge">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>test</title>
</head>
<body>
	<h1>test</h1>
</body>
</html>
```

#### コンテナに接続する
左下`><`をクリックして、`Reopen in Container`をクリックすると、コンテナを生成して、コンテナに接続して `workspaceFolder` にて指定したフォルダを Visual Studio Code 上で操作出来ます。