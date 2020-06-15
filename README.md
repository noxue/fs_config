# 配置 freeswitch 教程

安装部分参考网址 https://www.cnblogs.com/whf191/p/12320482.html

## 所有命令：
===================

```bash
yum update update  -y
yum groupinstall "开发工具"  -y
yum install zlib-de* libjpeg-de*  sqlite*   epel-re*  libcurl* pcre-de*  speex* libldns*  libedit* openssl* lua*  libsndfile*  yasm* -y
wget https://files.freeswitch.org/releases/freeswitch/freeswitch-1.6.2.tar.gz
tar xvf  freeswitch-1.6.2.tar.gz 
cd freeswitch-1.6.2

#打开modules.conf 注释掉这些模块 mod_enum mod_fsv mod_opus mod_sndfile mod_vpx ，当然如果你需要这些模块，需下载对应版本的库，手动编译安装.

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


