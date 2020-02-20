今天在阿里云ECS上布置了一个redis服务，结果发现连不上，顿时很尴尬，找解决方案也挺浪费时间的，就记录一下，防止童鞋们继续踩坑
# 是不是防火墙问题
![image.png](8350955-582189409c671cb5.png)

在telnet失败之后，我直接去看防火墙的问题了，但是我装的是centos7+了，所以我直接看了下firewalld的状态，发现是关闭的，这个时候就很郁闷，怎么会连不上呢
# 查看redis的配置
![image.png](8350955-cff7f6372dd51765.png)

我又去查看了一下redis的配置，但是redis的配置已经将bind 127.0.0.1注释掉了尴尬，网上还有人说是需要改成0.0.0.0的但是仍旧没用
# 阿里云的安全组配置
这个时候我想到了我这个虚拟机不是自己搭建的而是阿里云的ECS，所以我到控制台去看了一下（因为原来我配置mysql的时候踩过坑，所以轻车熟路了，不然指不定坑多少时间呢）。然后迫不及待的配置了安全组之后我又发现，还是连不上。
# 最坑爹的iptables
![image.png](8350955-b4f5913d02c8ad71.png)

原来ECS上还有iptables，真的是太无语了，转了一圈还是回到了防火墙了，也不知道是我什么时候装的iptables还是自动给我装上的，反正需要配置一下端口
``` shell
 vim /etc/sysconfig/iptables
# 加入如下代码
-A INPUT -m state --state NEW -m tcp -p tcp --dport 6379 -j ACCEPT

#最后重启iptables
service iptables restart
```
终于能连上redis了。
要是到这里还不能解决，请轻喷，哈哈