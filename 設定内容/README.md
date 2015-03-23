# サバフェス設定手順

## 戦いの履歴

* * *

## 環境準備
* 履歴を残すために以下の様な構成でGitを利用し設定を保存できるようにした(延長戦では設定せず)
* DBサーバが外部へssh通信/https通信ができなかったので以下のような設定にしていた
 # 踏み台サーバにGitHubリポジトリからclone
 # DBサーバの/root以下にGitリポジトリを作る
 # Gitリポジトリ内に各設定ファイル(my.cnfやlimits.confなど)を置きそこにシンボリックリンクを貼る
 # 各ファイル設定を変更したらコミット
 # 踏み台サーバはclonで1分に一回DBサーバリポジトリからpull
 # 踏み台サーバはclonで1分に一回GitHubリポジトリへpush

* * *

## OS設定

### mysqlユーザ作成
* /nfs_mountに置かれているファイルのパーミッションにあわせ以下手順で作成
```
# groupadd -g 499 mysql
# useradd -u 498 -g mysql mysql
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


