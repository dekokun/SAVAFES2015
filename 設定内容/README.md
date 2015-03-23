# サバフェス設定手順

## 参加メンバー
* @gotyoooo
 * 言い出しっぺ
 * 社会人2年目
 * 普段は基本的にはインフラ担当(主にミドルウェア)
 * CIが好き・自動化好き
 * 好きなツールはChef
* Mさん(twitterやってない)
 * gotyoooがいつもMySQLで困ったら相談する方
 * データベースエンジニア
 * 涼しい顔で火を吹いてるDBを助けている姿をよく見る
* @dekokun
 * gotyooooの尊敬する先輩
 * 社会人7年目?(未確認)
 * 本職はサーバサイドプログラム(主にPHP)
 * あとCIが好き

* * *

## 戦いの履歴
### 事前準備編
* 設定ファイルはGitで管理できるようにしようと決めていた
* あとは以下の流れでチューニングしようと方針だけ決めた
 * MySQL,Maria,Perconaの5.6系は全てインストールしてある程度チューニングが出来た状態でどれを使うか決める
 * fusion-ioはxfsで(ネット上でそれがいいって書いてあるし)
* 最初の作業担当を割り振った
 * OS/ディスクの設定 -> @gotyoooo
 * 各DBインストール・初期設定 -> Mさん
 * gitやci環境を整備 -> @dekokun
* あとは当日まで色々サイトを眺める

### 第一陣編
### 延長戦編

* * *

## 環境準備
* 履歴を残すために以下の様な構成でGitを利用し設定を保存できるようにした(延長戦では設定せず)
* DBサーバが外部へssh通信/https通信ができなかったので以下のような設定にしていた
 1. 踏み台サーバにGitHubリポジトリからclone
 1. DBサーバの/root以下にGitリポジトリを作る
 1. Gitリポジトリ内に各設定ファイル(my.cnfやlimits.confなど)置きそこにシンボリックリンクを貼る
 1. 各ファイル設定を変更したらコミット
 1. 踏み台サーバはclonで1分に一回DBサーバリポジトリからpull
 1. 踏み台サーバはclonで1分に一回GitHubリポジトリへpush

* * *

## OS設定

### 不要デーモンを止める
* 以下デーモンをstopし,chkconfigをoffにした
 * iptables
 * ip6tables
 * postfix
 * acpid
 * auditd
 * cpuspeed
 * haldaemon
 * kdump
 * messagebus
 * quota_nld
 * rdisc

### mysqlユーザ作成
* /nfs_mountに置かれているファイルのパーミッションにあわせ以下手順で作成
```
# groupadd -g 499 mysql
# useradd -d /var/lib/mysql -u 498 -g mysql mysql
```

### limits.conf
* 以下を追記
```
mysql soft nproc 32768
mysql hard nproc 32768
mysql soft nofile 65536
mysql hard nofile 65536
```

### sysctl.conf
* 以下の状態
```
# Kernel sysctl configuration file for Red Hat Linux
#
# For binary values, 0 is disabled, 1 is enabled.  See sysctl(8) and
# sysctl.conf(5) for more details.

# Controls IP packet forwarding
net.ipv4.ip_forward = 0

# Controls source route verification
net.ipv4.conf.default.rp_filter = 1

# Do not accept source routing
net.ipv4.conf.default.accept_source_route = 0

# Controls the System Request debugging functionality of the kernel
kernel.sysrq = 0

# Controls whether core dumps will append the PID to the core filename.
# Useful for debugging multi-threaded applications.
kernel.core_uses_pid = 1

# Controls the use of TCP syncookies
net.ipv4.tcp_syncookies = 1

# Disable netfilter on bridges.
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-arptables = 0

# Controls the default maxmimum size of a mesage queue
kernel.msgmnb = 65536

# Controls the maximum size of a message, in bytes
kernel.msgmax = 65536

# Controls the maximum shared segment size, in bytes
kernel.shmmax = 68719476736

# Controls the maximum number of shared memory segments, in pages
kernel.shmall = 4294967296

# MySQL parameter
kernel.sem = 250 32000 100 500
vm.overcommit_memory=1
vm.overcommit_ratio=99
vm.swappiness=0
kernel.msgmni=1024

# Network
net.core.somaxconn = 1024
net.core.rmem_max = 8388608
net.core.wmem_max = 6553600
net.ipv4.tcp_rmem = 4096 87380 8388608
net.ipv4.tcp_wmem = 4096 65536 6553600
net.ipv4.tcp_fin_timeout = 5
net.ipv4.ip_local_port_range = 1024 65535

# ipv6 disable
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1

# 非同期I/Oリクエスト数上限設定(デフォルト:65536)
fs.aio-max-nr = 1048576
```

* 反映
```
# sysctl -p
```

### /etc/profile
* 以下追記
```
if [ $USER = "mysql" ]; then
       if [ $SHELL = "/bin/ksh" ]; then
            ulimit -p 32768
            ulimit -n 65536
       else
            ulimit -u 32768 -n 65536
       fi
fi
```

### ディスクマウント
* もう一つ利用されていないディスクがあったのでマウント
```
mkfs.xfs -s size=4096 -b size=4096 /dev/sdb1 -f
mount -o defaults,noatime,nobarrier -t xfs /dev/sdb1 /var/lib/mysql
```

## fusion-io設定手順
* fioフォーマット
```
# umount /dev/fioa
# fio-detach /dev/fct0
# fio-format --block-size 4096 /dev/fct0
# fio-attach /dev/fct0
```

* xfsファイルシステム構築・マウント
```
# mkfs.xfs -s size=4096 -b size=4096 /dev/fioa -f
# mount -o defaults,noatime,nobarrier -t xfs /dev/fioa /fioa
```

* ext4ファイルシステム構築・マウント
 * inode領域を広げている
 * journalを無効化している
```
# mkfs.ext4 -b 4096 -f 4096 -I 256 /dev/fioa 
# tune2fs -o journal_data_writeback /dev/fioa
# tune2fs -O ^has_journal /dev/fioa
# e2fsck -f /dev/fioa
# debugfs -R features /dev/fioa
# mount -o defaults,noatime,nodiratime,nobarrier,data=writeback -t ext4 /dev/fioa /fioa
```

* * *

## DB設定

### mariadb10.0インストール

#### 必要パッケージインストール
* 以下パッケージをインストール
 * bzr > 2.0
 * gunzip
 * GNU tar
 * gcc 2.95.2 or later
 * gcc-c++
 * GNU make 3.75 or later
 * libtool 1.5.24 or later
 * bison (2.0 for MariaDB 5.5)
 * libncurses
 * zlib-dev
 * GNU automake
 * GNU autoconf
 * cmake
 * jemalloc
* 確認
```
# rpm -qa | egrep 'bzr|^gcc|gcc-c++|make|libtool|bison|ncurses-devel|zlib-devel|automake|autoconf|cmake|jemalloc'
# yum install .....
# rpm -ivh jemalloc-3.6.0-1.el6.x86_64.rpm
# rpm -qa | egrep 'bzr|^gcc|gcc-c++|make|libtool|bison|ncurses-devel|zlib-devel|automake|autoconf|cmake|jemalloc'
# yum install libaio-devel
```

#### インストールから初回起動
* /usr/local/srcにDLしたmariadb-10.0.17.tar.gzを配置
```
# cd /usr/local/src
# tar xzf mariadb-10.0.17.tar.gz
# chown -R mysql:mysql mariadb-10.0.17
# su - mysql
$ cd /usr/local/src/mariadb-10.0.17
$ cmake . -DCMAKE_INSTALL_PREFIX:PATH=/var/lib/mysql -DMYSQL_DATADIR:PATH=/fioa/mysql
$ make
$ make test
$ make install
$ cd /var/lib/mysql
$ mkdir conf binlog log tmp data/iblog
$ cd
$ vi .bash_profile
$ source .bash_profile
$ cd /var/lib/mysql/conf
$ vi my.cnf
$ cd ~/scripts
$ ./mysql_install_db --defaults-file=/var/lib/mysql/conf/my.cnf --basedir=/var/lib/mysql --user=maria --datadir=/fioa/mysql
$ mysqld_safe --defaults-file=/var/lib/mysql/conf/my.cnf --user=maria10 --socket=/tmp/mysql.sock &
```

### percona56インストール
#### ソースダウンロード
```
# cd /usr/loca/src
# wget http://www.percona.com/downloads/Percona-Server-5.6/Percona-Server-5.6.22-72.0/source/tarball/percona-server-5.6.22-72.0.tar.gz
```

#### 必要パッケージインストール
* 以下パッケージをインストール
 * bzr > 2.0
 * gunzip
 * GNU tar
 * gcc 2.95.2 or later
 * gcc-c++
 * GNU make 3.75 or later
 * libtool 1.5.24 or later
 * bison (2.0 for MariaDB 5.5)
 * libncurses
 * zlib-dev
 * GNU automake
 * GNU autoconf
 * cmake
 * jemalloc
* 確認
```
# rpm -qa | egrep 'bzr|^gcc|gcc-c++|make|libtool|bison|ncurses-devel|zlib-devel|automake|autoconf|cmake|jemalloc'
# yum install .....
# rpm -ivh jemalloc-3.6.0-1.el6.x86_64.rpm
# rpm -qa | egrep 'bzr|^gcc|gcc-c++|make|libtool|bison|ncurses-devel|zlib-devel|automake|autoconf|cmake|jemalloc'
# yum install libaio-devel
```

#### インストールから初回起動
```
# cd /usr/local/src
# tar xzf percona-server-5.6.22-72.0.tar.gz
# chown -R mysql:mysql percona-server-5.6.22-72.0
# su - mysql
$ cd /usr/local/src/percona-server-5.6.22-72.0
$ cmake . -DCMAKE_INSTALL_PREFIX:PATH=/var/lib/mysql -DMYSQL_DATADIR:PATH=/fioa/mysql
$ make -j20
$ make test
$ make install
$ cd /var/lib/mysql
$ mkdir conf binlog log tmp
$ exit
# cd /fioa
# mkdir mysql
# mkdir tmp
# chown mysql:mysql mysql
# chwon mysql:mysql tmp
# su - mysql
$ cd /var/lib/mysql/conf
$ vi my.cnf.02
$ vi ~/.bash_profile
$ cd ~/scripts
$ ./mysql_install_db --defaults-file=/var/lib/mysql/conf/my.cnf.02 --basedir=/var/lib/mysql --user=mysql --datadir=/fioa/mysql
$ mysqld_safe --defaults-file=/var/lib/mysql/conf/my.cnf.02 --user=mysql --socket=/tmp/mysql.sock &
```

### initファイル設置
* 自作のinitファイル
```
#!/bin/sh

# chkconfig: 2345 64 00
# description: mysql script.

basedir=/var/lib/mysql
datadir=/fioa/mysql
mysql_owner=mysql

start_mysql() {
    sleep 5
    if [ -e /fioa/mysql/ib_logfile0 ];
    then
        rm -rf /fioa/mysql/ib_logfile0
    fi
    if [ -e /fioa/mysql/ib_logfile1 ];
    then
        rm -rf /fioa/mysql/ib_logfile1
    fi
    if [ -e /var/lib/mysql/data/ib_logfile0 ];
    then
        rm -rf /var/lib/mysql/data/ib_logfile0
    fi
    if [ -e /var/lib/mysql/data/ib_logfile1 ];
    then
        rm -rf /var/lib/mysql/data/ib_logfile1
    fi

    su - $mysql_owner -c "$basedir/bin/mysqld_safe --defaults-file=/var/lib/mysql/conf/my.cnf.02 --user=mysql --socket=/tmp/mysql.sock --numa-interleave=1 >/dev/null 2>&1 &"
    sleep 60
}

stop_mysql() {
    sleep 15
    su - $mysql_owner -c "$basedir/bin/mysqladmin shutdown -uroot"
    sleep 15
}

case "$1" in
        start)
                # Start the Mysql databases:
                start_mysql

                touch /var/lock/subsys/mysql
                ;;

        stop)
                stop_mysql

                rm -f /var/lock/subsys/mysql
                ;;
        restart)
                $0 stop
                $0 start
                ;;
        status)
                if [ -f /var/lock/subsys/mysql ]; then
                echo $0 started.
                else
                echo $0 stopped.
                fi
                ;;
        *)
                echo "usage: mysql {start|stop|restart|status}"
                exit 1
esac

exit 0
```

### 最終のmy.cnf
```
# For advice on how to change settings please see
# http://dev.mysql.com/doc/refman/5.6/en/server-configuration-defaults.html

[mysqld]

# ---------------------------------------------------------
# Base
# ---------------------------------------------------------
basedir                                 = /var/lib/mysql
port                                    = 3306
autocommit                              = off
user                                    = mysql
#foreign_key_checks                      = off
#unique_checks                           = off
transaction-isolation                   = READ-UNCOMMITTED
bind_address                = 0.0.0.0

# ---------------------------------------------------------
# Binary Log
# ---------------------------------------------------------
#binlog_format                           = mixed
expire_logs_days                        = 7
#max_binlog_size                         = 512M
sync_binlog                             = 0


# ---------------------------------------------------------
# Connection Management
# ---------------------------------------------------------
max_allowed_packet                      = 4M
max_connect_errors                      = 4294967295
max_connections                         = 200
net_buffer_length                       = 1M
open_files_limit                        = 65536
table_open_cache                        = 10000
#table_open_cache_instances              = 16


# ---------------------------------------------------------
# File Location
# ---------------------------------------------------------
datadir                                 = /fioa/mysql
innodb_log_group_home_dir               = /fioa/mysql
#innodb_log_group_home_dir               = /var/lib/mysql/data
innodb_data_home_dir                    = /fioa/mysql
#log_bin                                 = /var/lib/mysql/binlog/mysql-bin
log_error                               = /var/lib/mysql/log/mysql_error.log
relay_log_recovery                      = on
#slow_query_log_file                     = /var/lib/mysql/log/mysql-slowqueries.log
tmpdir                                  = /fioa/tmp
#tmpdir                 = /var/lib/mysql/tmp

# ---------------------------------------------------------
# Innodb
# ---------------------------------------------------------
innodb_additional_mem_pool_size         = 8M
innodb_adaptive_flushing                = on
innodb_buffer_pool_size                 = 27G
innodb_buffer_pool_dump_at_shutdown     = on
innodb_buffer_pool_load_at_startup      = on
innodb_buffer_pool_instances            = 8
innodb_change_buffer_max_size           = 50
innodb_change_buffering                 = changes
innodb_data_file_path                   = ibdata1:76M:autoextend
innodb_file_per_table                   = on
innodb_flush_method                     = O_DIRECT
innodb_flush_neighbors                  = 0
innodb_io_capacity                      = 100000
innodb_io_capacity_max                  = 250000
innodb_log_buffer_size                  = 32M
innodb_log_block_size                   = 4k
innodb_log_file_size                    = 4G
innodb_log_files_in_group       = 1
innodb_open_files                       = 15000
#innodb_page_size                        = 4k
innodb_print_all_deadlocks              = off
innodb_read_ahead_threshold             = 0
innodb_read_io_threads                  = 12
innodb_sort_buffer_size                 = 16M
innodb_thread_concurrency               = 0
innodb_write_io_threads                 = 24

innodb_sync_array_size          = 62
innodb_support_xa           = off
innodb_rollback_segments        = 32

# ---------------------------------------------------------
# Myisam
# ---------------------------------------------------------
key_buffer_size                         = 64M


# ---------------------------------------------------------
# Memory
# ---------------------------------------------------------
binlog_stmt_cache_size                  = 65536
max_allowed_packet                      = 4M
max_heap_table_size                     = 512M
tmp_table_size                          = 512M
query_cache_size                        = 0
query_cache_type                        = 0
thread_cache_size                       = 64


# ---------------------------------------------------------
# Memory Allocation per Connection
# ---------------------------------------------------------
binlog_cache_size                       = 65536
join_buffer_size                        = 2M
read_buffer_size                        = 2M
read_rnd_buffer_size                    = 8M
sort_buffer_size                        = 8M


# ---------------------------------------------------------
# Replecation
# 1.read_only: (master=off, slave=on)
# 2.server_id: (master1=1, slave1=2, slave2=3)
# 3.apply the slave db only
# 4.apply the slave db only
# ---------------------------------------------------------
read_only                               = off
server_id                               = 1
#relay-log                              = mysql-relay-bin
#relay-log-index                        = mysql-relay-bin

# ---------------------------------------------------------
# Slow Query Log
# ---------------------------------------------------------
#long_query_time                         = 0.5
slow_query_log                          = off


# ---------------------------------------------------------
# Thread Pool
# ---------------------------------------------------------
thread_handling                         = pool-of-threads
thread_pool_size                        = 32
thread_pool_oversubscribe               = 32


# ---------------------------------------------------------
# SQL Behavior
# ---------------------------------------------------------
character_set_server                    = utf8
explicit_defaults_for_timestamp         = on
max_error_count                         = 10000
performance_schema                      = off
skip-name-resolve

innodb_doublewrite
innodb_flush_log_at_trx_commit = 1
innodb_fast_shutdown = 0

```
