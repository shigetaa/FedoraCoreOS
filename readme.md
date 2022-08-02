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
