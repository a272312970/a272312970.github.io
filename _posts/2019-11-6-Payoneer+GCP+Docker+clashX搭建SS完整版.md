---
layout: post
title: "Payoneer+GCP+Docker+clashX搭建SS完整版"
date: 2019-11-6
description: "其他"
tag: 其他 
--- 

## 概述
#### 记录一次薅谷歌羊毛搭建SS的经历，GCP对于新用户可以免费赠送300美金，可以用于搭建自己的服务器等等，但是谷歌鉴于去年开始谷歌的选项里面已经没有中国，所以中国的银行卡账号已经不能免费申请了，所以这里采用Payment造一个美国本土的虚拟银行卡号（其他地区也可以试试）,本文全程使用mac配置
### 1,申请账号

进入 [GCP](https://cloud.google.com/free/) ，单击Try it Free
接受条款 – 同意并继续，
需要谷歌账号，地区没有中国，选择美国，然后，在[Payoneer](https://login.payoneer.com/api/v2/internal/logout/authorize?client_id=b3d186db-4e5d-49c8-8a12-5753136af807&redirect_uri=https%3a%2f%2fmyaccount.brand.domain%2flogin%2flogin.aspx&scope=myaccount%20openid&response_type=code&state=4ef9c2d3-00d4-490f-b189-989cb55b0c20)中申请一张美国银行的储蓄卡，在收款--Global Payment Service 中增加，USD账户，默认是First Century Bank，这个GCP好像不能使用这种银行的，跟客服沟通换成Community Federal Savings Bank银行，

### 2，创建服务器

注册成功后，登陆到后台，即可开始创建服务器。在后台左侧列表选择 Computer Engine => VM实例，进行创建：
![](/images/posts/gcp/chuangjian1)

名称随意填写即可，然后地区，根据需要进行选择，我这里选择 "asia-east1-a",然后机器类型根据自己的需要进行选择，配置越高，当然就越贵。如果仅仅想搭一个科学上网工具，配置不需要太高
![](/images/posts/gcp/chuangjian2.jpg)
不要以为谷歌送的 300 美元很多就随意选一个高配置，由于谷歌云的流量是单独计费的，每G大概 0.23 美元左右。在配置上节省的钱，刚好可以使用更多的流量。高配置费用还是比较昂贵的,这里我把镜像改为 centos7 ，然后在防火墙规则那边勾选允许 http和https 流量

![](/images/posts/gcp/chuangjian3.jpg)
### 3，配置
创建完成之后，这里建立先在gcp上配置ssh,把~/.ssh里面的.pub公钥拿出来，复制粘贴进去，公钥最后空格后面设置好"用户名"chenzhe

```
cat ~/.ssh/id_rsa.pub 
```
最后在你的终端上用ssh连接,ssh 用户名@外部ip地址

```
ssh chenzhe@35.241.109.167 
```
![](/images/posts/gcp/ssh.png)

连接上了之后，使用[docker](https://docs.docker.com/install/linux/docker-ce/centos/)来配置,这里用的centerOS的，

```
1  sudo yum install -y yum-utils   device-mapper-persistent-data   lvm2
2  sudo yum-config-manager     --add-repo     https://download.docker.com/linux/centos/docker-ce.repo
3  sudo yum-config-manager --enable docker-ce-nightly
4  sudo yum-config-manager --enable docker-ce-test
5  sudo yum-config-manager --disable docker-ce-nightly
6  sudo yum install docker-ce docker-ce-cli containerd.io
7  sudo systemctl start docker
```
docker启动之后
运行sudo docker run -dt --name ss -p 外部IP地址:容器IP地址 mritd/shadowsocks -s "-s 0.0.0.0 -p 容器IP地址 -m 
加密算法 -k 密码"

```
sudo docker run -dt --name ss -p 12775:6443 mritd/shadowsocks -s "-s 0.0.0.0 -p 6443 -m 
aes-256-gcm -k ******"
```
这样就算配置好了，可以查看下运行状态下的容器

```
sudo docker ps
```
如果没有的话，容器可能异常断开了

```
sudo docker ps -a
```
查看下所有容器状态，这里查看到容器断开了，
所以直接再启动他
![](/images/posts/gcp/docker.jpg)

```
sudo docker start 5ef44b81ab72
```
这里可以再定义几个策略

```
//1.设置开机启动
sudo systemctl enable docker

//2.退出重启
sudo docker update --restart=unless-stopped ss
```
到这里配置已经完成了


### 4，客户端配置
下载[clashX](https://github.com/yichengchen/clashX)，**release**-**asset**-**ClashX.dmg**，下载完之后打开软件，点击配置-打开配置文件夹，打开config.yaml，在里面配置规则，这里有一份别人总结的[规则](https://github.com/Hackl0us/SS-Rule-Snippet/blob/master/LAZY_RULES/clash.yaml)，判断国外ip全走代理，其余的全不走代理，可以拷贝进来，放进去
，还有[另外一份](https://github.com/a272312970/chenzhe/blob/master/resource/gcp/posts/gcp/config.yaml),是匹配一些特定的网站（youtube，facebook,jcenter等等）会走代理,匹配规则设置好之后，选择出站模式为规则判断，然后退出软件重启一下就可以生效了
![](/images/posts/gcp/clashX.jpg)
![](/images/posts/gcp/clashX2.jpg)

到这里整个薅羊毛经历完毕，这里客户端只针对mac端的配置，手机端的话自己去下载个ss的客户端可以使用




   


