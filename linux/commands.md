# Linux系统相关

## 磁盘

```bash
# 创建新分区然后格式化
fdisk /dev/vdb
mkfs -t xfs /dev/vdb1


# 也可以不分区直接格式化
mkfs.xfs -f /dev/vdb  # 或者 mkfs -t xfs /dev/vdb

# 创建挂载
echo `blkid /dev/vdb | awk '{print $2}' | sed 's/\"//g'` /data xfs defaults 0 0 >> /etc/fstab
mkdir /data
mount -a
```

磁盘扩容

```bash
# 磁盘扩容后，这是对于整个盘都挂载到一个目录得情况
xfs_growfs /dev/vdb   # 针对xfs，可能需要安装 yum install xfsprogs
resize2fs /dev/xvda1  # 针对ext2 ext3 ext4
```

如果盘做了分区，且只有一个区，并扩容该分区为整个盘得容量

```bash
#NAME         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
#/dev/xvda    202:0    0   25G  0 disk
#└─/dev/xvda1 202:1    0   20G  0 part /

lsblk # 查看
growpart /dev/xvda 1     # 就是上面得格式  xvda 是设备  1 是分区
growpart /dev/nvme0n1 1  #
# 执行完后可能还需要执行 xfs_growfs /dev/vdb 或 resize2fs /dev/xvda1 这要取决于具体文件系统
```

查看

```bash
# 查看除去 proc 目录的其他目录磁盘占用情况
du -sh /* --exclude="proc"

# 查看磁盘UUID
blkid
ls -l /dev/disk/by-uuid
```

查看文件占用

```bash
# 查看删除但没有释放的文件
lsof |grep deleted | grep /var/log/kucoin      # lsof -D /var/log/kucoin
# 一般需要让程序自己释放，比如重启程序，但线上服务有时候不方便重启，可以找到这个进程的FD来清空
# 上面执行之后可以看到进程号(第二列)
cd /proc/PID/fd/   # 已经删除的文件带有 deleted 标记，然后 ll | grep deleted


# 删除问号文件
ls -i
find ./ -inum 问号文件ID -delete
```

## 定时任务

```bash
+1 表示 1 * 24 +24小时以外..
+0 表示 0 * 24 +24小时以外
1 表示 1 * 24 + 24 到 24 之间..
0 表示 0 * 24 + 24 到 0 之间..
-1 表示 0 * 24 +24 内,甚至为未来时间...
# atime 是访问时间，查看也算，但是查看不属于修改时间
# -mtime +1  2天前也就是48小时以前，  1 * 24 + 24 = 48
# -mtime +0  1天前也就是24小时以前，  0 * 24 + 24 = 24
# -mtime 1   当前时间48小时前 到 当前时间24小时前 这段时间
# -mtime 0.125   当前时间27小时前 到 当前时间24小时前 这段时间
# -mtime -1  最近24小时
# -mtime -0.5  最近12小时
# -mtime -0.25 最近6小时  -mtime +0.25  6小时前
# 删除文件2天前的
find /var/log/kucoin/ -type f -mtime +1 -exec rm -f {} \;

# 24小时前 或者叫做 1天前
find /var/log/kucoin/*/* -type f -name '*.[0-9][0-9][0-9][0-9]*' -mtime +0 -exec rm -f {} \;

find /var/log/kucoin/*/* -type f -name '*.[0-9][0-9][0-9][0-9]*' -mtime 0.125 -exec truncate -s 0 {} \;

0 */1 * * * find /var/log/kucoin/ -type f -name common-biz.log.* -exec rm -f {} \;
0 */1 * * * find /var/log/kucoin/ -type f -name common-default.log.* -exec rm -f {} \;
0 */1 * * * find /var/log/kucoin/ -type f -name common-biz.log.* -exec rm -f {} \;

# 3 小时前
0 1 * * * find /var/log/kucoin/ -type f -mtime +3 -exec rm -f {} \;
# 30分钟执行一次
*/30 * * * * find /var/log/kucoin/ -type f -exec truncate -s 0 {} \;
# 每3小时执行一次
0 */3 * * * find /var/log/kucoin/ -type f -atime +1 -exec rm -rf {} \;


# 清空文件
find /var/log/kucoin/ -type f -mtime +2 -exec truncate -s 0 {} \;

00 */1 * * * find /var/log/kucoin/ -type f -mtime +1 -exec rm -f {} \;
```


## 内核

```bash
# 程序读文件是把数据加载到pagecache（默认4K），多个程序读相同文件其实是共享pagecache，当然每个程序对相同文件读取的文件描述符不同，
# 文件描述符里记录的该程序读取的偏移量(lsof -op PID)，如果某个程序要修改内容那么就通过写时复制产生新的pagecache并标记它为脏页（脏页就是说明有修改）
# 写文件也是先写到pagecache，由OS决定什么时候落盘（时间维度和空间维度），pagecache查看工具 https://github.com/tobert/pcstat

# 脏页占物理内存70%的时候OS才会做异步flush操作把脏页写入磁盘，这时候不会阻塞程序，刷盘速度要比程序写入的速度要快才行
vm.dirty_background_ratio = 70
# 这个和上面的作用是一样的，只不过是按照具体大小来设置，当启用比例的话，这个就设置为0
vm.dirty_background_bytes = 0
# 脏页占物理内存90%的时候OS才会做同步flush操作把脏页写入磁盘，这时候会阻塞程序，如果程序还继续写，那么就触发OS的LRU，进行内存淘汰，但前提是脏页
# 不能直接淘汰，这个值通常比上面的值大，对于写压力比较大的建议把这个参数调大。
vm.dirty_ratio = 90
# 这个和上面的作用是一样的，只不过是按照具体大小来设置，当启用比例的话，这个就设置为0
vm.dirty_bytes = 0
# 把脏页写入磁盘的频率，间隔多久写一次，单位厘秒，实际内核使用时会 * 10，500 * 10 也就是5s
vm.dirty_writeback_centisecs = 500
# 配置脏页能存活多久，默认3000，也就是30s
vm.dirty_expire_centisecs = 3000
```

## 网络

```bash
# 查看连接状况
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'

# 查看访问这个IP走哪里
ip route get [IP]  

route add -host 10.1.20.11 dev eth0
sudo route del -host 10.1.20.11 dev eth0
```

## 错误排查

### ubuntu 下遇到curl不能用

```bash
# 遇到如下错误
curl: /lib/x86_64-linux-gnu/libcurl.so.4: no version information available (required by curl)
curl: symbol lookup error: curl: undefined symbol: curl_url_set, version CURL_OPENSSL_4

# 可尝试重新安装
sudo apt-get remove libcurl4 libcurl4-openssl-dev -y
sudo apt-get install libcurl4 libcurl4-openssl-dev -y
sudo apt-get curl
```
