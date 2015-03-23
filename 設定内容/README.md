# サバフェス設定手順

## 戦いの履歴

## 環境準備

### 

### mariadb10.0インストール

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
