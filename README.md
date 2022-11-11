# fixiosopenssh

修复iOS越狱后openssh无法连接

* 环境: iOS10 + h3lix越狱 + openssh

## 分析

在mterminal手动执行/usr/bin/sshd提示ssh_host*key不存在，因此结论openssh这个包对于h3lix越狱环境配置错误  
错误的配置文件:/Library/LaunchDaemons/com.openssh.sshd.plist  
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.openssh.sshd</string>
    <key>Program</key>
    <string>/usr/libexec/sshd-keygen-wrapper</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/sbin/sshd</string>
        <string>-i</string>
    </array>
    <key>SessionCreate</key>
    <true/>
    <key>Sockets</key>
    <dict>
        <key>Listeners</key>
        <dict>
            <key>SockServiceName</key>
            <string>ssh</string>
        </dict>
    </dict>
    <key>StandardErrorPath</key>
    <string>/dev/null</string>
    <key>inetdCompatibility</key>
    <dict>
        <key>Wait</key>
        <false/>
    </dict>
</dict>
</plist>
```

相关文件:/usr/libexec/sshd-keygen-wrapper  
```bash
#!/bin/sh
[ ! -f /etc/ssh/ssh_host_key ]     && ssh-keygen -q -t rsa1 -f /etc/ssh/ssh_host_key     -N "" -C "" < /dev/null > /dev/null 2> /dev/null
[ ! -f /etc/ssh/ssh_host_rsa_key ] && ssh-keygen -q -t rsa  -f /etc/ssh/ssh_host_rsa_key -N "" -C "" < /dev/null > /dev/null 2> /dev/null
[ ! -f /etc/ssh/ssh_host_dsa_key ] && ssh-keygen -q -t dsa  -f /etc/ssh/ssh_host_dsa_key -N "" -C "" < /dev/null > /dev/null 2> /dev/null
exec /usr/sbin/sshd $@
```

### 技巧

sshd日志保留: `/usr/sbin/sshd  -E /tmp/sshd_log`， 将此参数加入sshd-keygen-wrapper即可分析失败原因

### 问题1: 为何未生成/etc/ssh/ssh_host*_key
sshd-keygen-wrapper本来自动生成/etc/ssh/ssh_host*_key，可是由于com.openssh.sshd.plist配置错误导致未执行此脚本   
其中ProgramArguments首参从sshd改为sshd-keygen-wrapper即可  

### 问题2: 为何提示22端口占用
在上一步操作后执行sshd提示22端口占用，此是因为com.openssh.sshd.plist设置Listeners导致，看了下更高版本的配置也有该项， 
launchd通过该字段分配端口，然后通过/usr/sbin/sshd -i并后续给sshd传入标识完成sshd服务创建。然而在10系统上可能未对-i支持故失败,  
这里我们单纯去掉Sockets字段及其子项即可成功完成sshd服务创建

修复后

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.openssh.sshd</string>
    <key>Program</key>
    <string>/usr/libexec/sshd-keygen-wrapper</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/libexec/sshd-keygen-wrapper</string>
    </array>
    <key>SessionCreate</key>
    <true/>
    <key>StandardErrorPath</key>
    <string>/dev/null</string>
    <key>inetdCompatibility</key>
    <dict>
        <key>Wait</key>
        <false/>
    </dict>
</dict>
</plist>
```

### 重启服务器即可远程ssh连接

```bash
launchctl stop com.openssh.sshd
launchctl unload /Library/LaunchDaemons/com.openssh.sshd.plist
launchctl load /Library/LaunchDaemons/com.openssh.sshd.plist
launchctl start com.openssh.sshd
```

## 后记

iOS10上远程ssh在使用中比较卡，应该是系统设计缺陷。使用tcprelay或iproxy映射端口到本地后则非常流畅

```bash
nohup python2 tcprelay.py -t 22:22  &
```

