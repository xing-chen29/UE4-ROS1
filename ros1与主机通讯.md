# VMware
网卡改为“桥接”模式
# Ununtu
## 安装 rosbridge
```
sudo apt update
sudo apt install ros-noetic-rosbridge-server
```
## 启动roscore
~~~
roscore
~~~
## 启动rosbridge
```
source /opt/ros/noetic/setup.bash
roslaunch rosbridge_server rosbridge_tcp.launch bson_only_mode:=True
```

## 监听端口
~~~
ss -lntp | grep 9090
~~~

# 主机PowerShell
~~~
Test-NetConnection 虚拟机IPv4地址 -Port 9090
~~~