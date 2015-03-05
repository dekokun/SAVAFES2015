# SAVAFES2015

あなたのマシンへは以下のコマンドでログインしてください。
ssh root@192.168.186.29
パスワードはQQ1igIW4です。
パスワードは変更していただいても構いませんが全角/半角スペースは使用しないでください

ベンチマークの取得は以下のサイトから行えます
https://2015spring.serverfesta.info/ranking/benchmark-form.html
マシンのIPアドレス、Linuxのrootのパスワード、構築したDBのバージョンを入力するとベンチマークがスタートします。

DBサーバ構築手順の一例をご案内いたします（あくまで一例となりますので、ご参考まで）。

「MySQLのパッケージインストール用ディレクトリを作り、移動します」
1.# mkdir mysql-stable
2.# cd mysql-stable

「MySQLパッケージをダウンロードします」
3.# wget http://dev.mysql.com/get/Downloads/MySQL-5.5/MySQL-5.5.42-1.el6.x86_64.rpm-bundle.tar

「ダウンロードしたパッケージを解凍します」
4.#tar xvf MySQL-5.5.42-1.el6.x86_64.rpm-bundle.tar

「解凍したパッケージからMySQLをインストールします」
5.#yum localinstall MySQL-client* MySQL-server* MySQL-shared-compat
これで、MYSQLのクライアントとサーバがダウンロードとインストールが
されます。

「設定を変更し、DBをioDrive上に構築する前準備をします」
6.# vim /etc/my.cnf

以下の内容に、ファイルの中身を上書きします。

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


「競技用のDBデータへアクセスするため、セキュリティの一部を停止します。」
7.# service iptables stop
8.# setenforce 0

「競技用DBデータにアクセスするためのプログラム群をインストールします」
9.# yum install nfs-utils

「これで、ベンチマークをする準備ができました」
10.# service mysql start

※/nfs_mountのディレクトリは使用しないでください。ベンチマークが取得できなくなることがあります。
