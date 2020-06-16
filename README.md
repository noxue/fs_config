# CentOs7 配置 freeswitch1.6.2 教程

## 所有命令：
===================

```bash
yum update update  -y
yum groupinstall "开发工具"  -y
yum install zlib-de* libjpeg-de*  sqlite*   epel-re*  libcurl* pcre-de*  speex* libldns*  libedit* openssl* lua*  libsndfile*  yasm* -y
#这个文件下载很慢，可以迅雷下载之后，上传到服务器，再继续后续命令
wget https://files.freeswitch.org/releases/freeswitch/freeswitch-1.6.2.tar.gz
tar xvf  freeswitch-1.6.2.tar.gz 
cd freeswitch-1.6.2

#打开modules.conf 注释掉这些模块 mod_enum mod_fsv mod_opus mod_sndfile mod_vpx ，当然如果你需要这些模块，需下载对应版本的库，手动编译安装.

sed -i 's/\(.*mod_enum\)/#\1/g' modules.conf
sed -i 's/\(.*mod_fsv\)/#\1/g' modules.conf
sed -i 's/\(.*mod_opus\)/#\1/g' modules.conf
sed -i 's/\(.*mod_sndfile\)/#\1/g' modules.conf
sed -i 's/\(.*mod_vpx\)/#\1/g' modules.conf


#编译
./configure
make
make install

#建立软连接，方便直接运行 freeswitch 命令
ln -sf /usr/local/freeswitch/bin/freeswitch /usr/local/bin/
ln -sf /usr/local/freeswitch/bin/fs_cli /usr/local/bin/

#完成安装之后，设置符号链接：
ln -sf /usr/local/freeswitch/bin/freeswitch /usr/local/bin/
ln -sf /usr/local/freeswitch/bin/fs_cli /usr/local/bin/

#禁用ipv6 
cd /usr/local/freeswitch
mv conf/sip_profiles/internal-ipv6.xml conf/sip_profiles/internal-ipv6.xml.bk
mv conf/sip_profiles/external-ipv6.xml conf/sip_profiles/external-ipv6.xml.bk
#=======================

#安装声音文件(不是必须)
#make cd-sounds-install
#make cd-moh-install

#修改默认密码
sed -i 's/\(default_password=\)[^"]*/\1fs12345/g' conf/vars.xml

#配置卡线拨打
mv conf/dialplan/public.xml conf/dialplan/public.xml.bak
mv conf/dialplan/default.xml conf/dialplan/default.xml.bak



```


## 安装后的配置部分

### 修改密码

 FreeSWITCH的默认密码为1234，客户端使用该密码拨号时，会有10秒的延时。通过修改此默认密码，可以避免这个延时。
 如把新密码设为2345，此时可以把 `conf` 目录下的 `vars.xml` 中的：
 
 `<X-PRE-PROCESS cmd="set" data="default_password=1234"/>`

修改为

 `<X-PRE-PROCESS cmd="set" data="default_password=2345"/> `


## 配置卡线拨打电话

conf\dialplan目录下
public.xml 删除掉 
default.xml 内容替换成下面的内容

```xml
<include>
  <context name="default">
    <extension name="GOIP-call_out">
      <condition field="destination_number" expression="^\d+$"/>
      <condition field="caller_id_number" expression="^1\d{3}$">
        <action application="set" data="effective_caller_id_number=${caller_id_number}"/>
        <action application="set" data="effective_caller_id_name=${caller_id_number}"/>
        <action application="set" data="hangup_after_bridge=true"/>
        <action application="bridge" data="{absolute_codec_string='PCMA'}${regex(${sofia_contact(internal/1000@${domain_name})}|^(.+)sip:(.+)@(.+)|%1sip:${destination_number}@%3)}"/>
        <action application="hangup"/>
      </condition>
    </extension>

    <!-- 配置第二个网关，可以支持多个 -->
    <extension name="GOIP-call_out-2">
      <condition field="destination_number" expression="^\d+$"/>
      <condition field="caller_id_number" expression="^2\d{3}$">
        <action application="set" data="effective_caller_id_number=${caller_id_number}"/>
        <action application="set" data="effective_caller_id_name=${caller_id_number}"/>
        <action application="set" data="hangup_after_bridge=true"/>
        <action application="bridge" data="{absolute_codec_string='PCMA'}${regex(${sofia_contact(internal/2000@${domain_name})}|^(.+)sip:(.+)@(.+)|%1sip:${destination_number}@%3)}"/>
        <action application="hangup"/>
      </condition>
    </extension>
  </context>
</include>
```

## 修改 RTP(如果打通电话没有声音的话)

修改文件
```
/usr/local/freeswitch/conf/sip_profiles/internal.xml
```

大概 `284` 行左右
```xml
<param name="ext-rtp-ip" value="auto-nat"/>
<param name="ext-sip-ip" value="auto-nat"/>
```
修改为
```xml
<param name="ext-rtp-ip" value="fs所在服务器ip"/>
<param name="ext-sip-ip" value="fs所在服务器ip"/>
```




## 添加 SIP 账号


```bash
#添加一个 sip 账号 2000 
cp /usr/local/freeswitch/conf/directory/default/1000.xml /usr/local/freeswitch/conf/directory/default/2000.xml
sed -i 's/1000/2000/g' 2000.xml

#循环添加多个账号，下面是添加账号 2000~2032
for i in $(seq 2000 2032); 
do 
/bin/cp -f /usr/local/freeswitch/conf/directory/default/1000.xml /usr/local/freeswitch/conf/directory/default/$i.xml; 
sed -i "s/1000/$i/g" $i.xml; 
done
```


# 网关配置相关（杭州三汇网关）


## 配置过程

1. 【SIP设置中】首先以整个网关注册
1. 【端口设置】修改端口（注册该端口设置为 否） （接入方式为 静态绑定） （绑定号码为该端口想用那个sip账号来拨打比如 1001） 其他默认
1. 【端口组设置】 添加端口组（注册该端口组为 否） （认证选择方式为 以整个网关注册）其他默认，一个端口组 对应 一个端口
1. 【路由配置-》IP->Tel/IP路由】 添加路由规则（主叫前缀 改为 想用哪个sip账号来拨打 比如 1001）（目标端口组 设置为对应的端口） 其他默认


## 网关整体配置思路

1. 网络电话客户端使用指定的 `sip账号`，比如 `1001` 拨打 `手机号`
1. `freeswitch` 接受到请求，根据 `规则` 转到指定的网关所注册的账号 `1000`，并会带上是谁发起的拨打请求，比如 `1001`
1. 网关接受到拨打请求后根据 `路由规则` 找到对应的 `端口组` 
1. `端口组` 找到对应的 `端口`
1. `端口` 就使用端口中所插的 `手机卡` 来拨打 `1001` 账号多拨打的 `手机号`
1. `freeswitch` 的作用就是 `sip账号` 和 `网关` 之间的一个桥梁，把 `sip账号` 发起的请求转送到 `网关` 上。
1. `网关` 的作用就是负责把 `freeswitch` 发过来的请求根据规则用对应的 `手机卡` 拨打出去， 也就是 `freeswitch` 和 `手机卡` 之间的桥梁。

## 生成网关路由规则

### 下面两个代码作用一样。就是无聊写的。。

```rust
// rust 代码批量生成网关路由规则
fn main() {
    for i in 0..32 {
        println!("* {} * {} 1 default 0 0", 2001 + i, i);
    }
}
```

```bash
#linux命令批量生成网关路由规则
for i in $(seq 0 31); 
do 
echo "* $((2001+i)) * $i 1 default 0 0"
done
```

