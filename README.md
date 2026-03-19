# Web Controller Chroot模块

## chrootc
一些软件包
v1对应的v1.x.x的模块
v2对应的v2.x.x的模块

## 使用方法
下载vx.x.x标签内的模块，通过magisk应用刷入即可

## 介绍
web controller是一个magisk模块，基于busybox chroot命令开启的chroot容器的模块，该模块可以极大简化chroot的安装流程，配置了UI环境，适合移动端，手机端去使用。一键启动，快速使用。

### 模块的安装与使用

**请确保9527端口上，没有其他的服务占用**

1. **安装**
使用magisk 30.6版本，通过本地模块刷入，后续可以通过magisk内的模块板页升级
2. **使用**
  安装模块重启后，通过局域网 192.168.0.0/16 网段可以直接访问（白名单）
  如果需要自定义网段访问，可以修改已安装模块目录下的`httpd.conf`文件，添加你要允许访问网页的网段，然后在通过本地机器127.0.0.1:9527访问，重启模块的web服务即可

3. **容器的创建与下载**
   通过网页管理的界面的模板管理板块，点击浏览仓库即可看见你当前架构支持的模块
   点击下载，等待下载完成，等待创建按钮出现（不要立马操作，可以选择刷新下页面）
   然后点击，创建，根据需要修改下容器名和端口号，创建即可
   创建后的容器在容器管理版本，点击启动，就可以根据端口号使用 root/123456 登录到ssh
  > 记得修改ssh密码，由于登录到ssh后，这个端口会暴露到IPV6上，用的还是弱密码，第一步一定要去修改密码


### 自定义模板

如果你想创建你自己模板，可以看这里

我这里推荐去proot直接下他们做好的基础rootfs，然后，你再根据这个模板去创建属于自己的模板

[proot地址](https://github.com/termux/proot-distro/releases)

下载模板，上传到你的手机，例如 /sdcard/Download

然后把它解压到 /data/local/mnt
> tips: 解压时，记得保留原信息不变，例如 tar pxJvf xxx.tar.xz

**示例：**
在安卓shell上
```sh
# 设置环境变量
BUSYBOX=/data/adb/magisk/busybox
ROOTFSPATH="/data/local/mnt/debian13"

# 重新挂载/data
$BUSYBOX mount -o remount,dev,suid /data

挂载必要的目录
$BUSYBOX mount --bind /dev $ROOTFSPATH/dev
$BUSYBOX mount --bind /sys $ROOTFSPATH/sys
$BUSYBOX mount --bind /proc $ROOTFSPATH/proc
$BUSYBOX mount -t devpts devpts $ROOTFSPATH/dev/pts

# 进入chroot系统，最后的命令根据你选的rootfs来进入
$BUSYBOX chroot $ROOTFSPATH /bin/bash 

# chroot环境内
export PATH=$PATH:/bin

cat > /usr/sbin/policy-rc.d << 'EOF'
#!/bin/sh
exit 101
EOF

chmod +x /usr/sbin/policy-rc.d

apt update && apt remove --purge systemd-sysv --allow-remove-essential

apt install openssh-server runit -y

# 安装openssh-server可能会失败，需要执行下面的程序修复
apt --fix-broken install

# 清除缓存
apt autoremove && apt autoclean && apt clean

# 修改root密码
passwd

# 修改ssh配置文件，允许root登录
# 创建/etc/init 和 /etc/exit脚本，记得赋予执行权限

# 添加 SVDIR环境修复sv找不到服务目录的问题
echo "export SVDIR=/var/service" >> /etc/profile

```

> 部分rootfs需要你手动创建/etc/sv下的服务目录

**init脚本参考**

```sh
#!/bin/bash -e

# 该PATH需要你去/etc/profile中去看它提供的路径，然后在这里粘过来，否则，runsv 命令无法启动
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# 挂载服务目录
SOURCE_DIR="/etc/sv"
SERVICE_DIR="/var/service"
mkdir -p "$SERVICE_DIR"

for service_path in "$SOURCE_DIR"/*; do
    if [ -d "$service_path" ]; then
        service_name=$(basename "$service_path")
        
        # 使用case进行排除匹配
        case "$service_name" in
            default-syslog|svlogd|dhcpcd|samba)
                ;;
            *)
                ln -sfn "$service_path" "$SERVICE_DIR/$service_name"
                ;;
        esac
    fi
done

/usr/bin/runsvdir /var/service &

RUNSVDIR_PID=$!

echo $RUNSVDIR_PID > /var/main.pid

tail -f /dev/null &

INIT_PID=$!

echo $INIT_PID > /var/chroot.pid

wait $INIT_PID

```

> INIT_PID会由cgi-bin/stop进行停止，然后，之前通过cgi-bin/start启动的后台init程序会停止，最后卸载unshare --bind中的挂载

**exit脚本参考**

```sh
#!/bin/bash -e

export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

SERVICE_DIR="/var/service"

sv force-stop $SERVICE_DIR/*

rm -rf $SERVICE_DIR/*

# 等待5s让runsv进程自动停止
sleep 5

RUNSVDIR_PID=$(cat /var/main.pid)

# 如果RUNSVDIR_PID存在则kill
if [ -n "$RUNSVDIR_PID" ]; then
  kill -9 "$RUNSVDIR_PID"
fi

echo "" > /var/main.pid

```

> 这是一个补充，添加ssh的服务需要看rootfs中，安装openssh-server有没有给你提供这个脚本，没有的话就得自己创建
> mkdir -p /etc/sv/ssh
> nano /etc/sv/ssh/run ... 这里添加你的启动脚本（不会就问 AI）
> chmod +x /etc/sv/ssh/run

### 配置为模板

创建目录：/data/local/templates/{模板名}/lower
移动模板文件: mv -p $ROOTFSPATH/* /data/local/templates/{模板名}/lower/
写入完成状态: echo "3" > /data/local/templates/{模板名}/status

前端刷新下，你的模板就出来了










