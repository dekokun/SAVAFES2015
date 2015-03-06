# SAVAFES2015

## 接続情報

* あなたのマシンへは以下のコマンドでログインしてください。

```
ssh root@192.168.186.29
```

* パスワードはQQ1igIW4です。
* パスワードは変更していただいても構いませんが全角/半角スペースは使用しないでください

* ベンチマークの取得は以下のサイトから行えます
- https://2015spring.serverfesta.info/ranking/benchmark-form.html
- マシンのIPアドレス、Linuxのrootのパスワード、構築したDBのバージョンを入力するとベンチマークがスタートします。

## DBサーバ構築手順の一例

* あくまで一例となりますので、ご参考まで

* 「MySQLのパッケージインストール用ディレクトリを作り、移動します」

```
# mkdir mysql-stable
# cd mysql-stable
```

* 「MySQLパッケージをダウンロード・解凍・インストール」
```
# wget http://dev.mysql.com/get/Downloads/MySQL-5.5/MySQL-5.5.42-1.el6.x86_64.rpm-bundle.tar
# tar xvf MySQL-5.5.42-1.el6.x86_64.rpm-bundle.tar
# yum localinstall MySQL-client* MySQL-server* MySQL-shared-compat
```
* 設定を変更し、DBをioDrive上に構築する前準備をします
```
# vim /etc/my.cnf
[mysqld]
datadir=/fio/mysql
user=mysql
max_connections = 1000
thread_cache_size = 1000
innodb_buffer_pool_size = 16G
innodb_log_file_size = 512M
skip-name-resolve
innodb_flush_method=O_DIRECT
innodb_write_io_threads=16
innodb_read_io_threads=8
innodb_io_capacity=10000
socket=/fioa/mysql/mysql.sock

[client]
socket=/fioa/mysql/mysql.sock

[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid

※以下の設定はデフォルト値から変更しないでください（レギュレーション）。
innodb_doublewrite
innodb_flush_log_at_trx_commit = 1
```

* 「競技用のDBデータへアクセスするため、セキュリティの一部を停止します。」
```
# service iptables stop
# setenforce 0
```

* 「競技用DBデータにアクセスするためのプログラム群をインストールします」
```
# yum install nfs-utils
```

* 「これで、ベンチマークをする準備ができました」
```
# service mysql start
```

## 注意
* ※/nfs_mountのディレクトリは使用しないでください。ベンチマークが取得できなくなることがあります。

## コンフィグのGithub同期

DBサーバから22ポートで外部に出られなかったため以下方法をとっている

1. DBサーバの/root/repoディレクトリでコミット
2. 踏み台サーバはcronにより毎分上記からgit pullし、毎分git pushを行うことで毎分githubにsyncしている
