---
title: LetsEncrypt
date: 2021-10-04 22:30:00
categories: LetsEncrypt
description: 免费的SSL/TLS证书 3个月有效期 每2个月申请一次
tags:
- LetsEncrypt
---

# 注意事项    测试时务必使用staging api
```
申请证书频率的限制? 
  同一个主域下一周只能申请 50 张证书 (例如 www.example.com 的主域是 example.com) 
  每个账户每个域名每小时申请验证失败的次数为 5 次 
  每周只能创建 5 个重复的证书，即使是通过不同账户创建 
  每个账户同一个 IPv4 地址每 3 小时最多可创建 10 张证书 
  每个多域名(SAN)证书最多包含 100 个子域 
  更新证书没有次数的限制，但是更新证书会受到上述重复证书的限制
```
# 基础知识

**查询证书列表?**

​	可在 [crt.sh](https://crt.sh) 进行查询域名的证书详情

**证书维护**   [tools.letsdebug.net](https://tools.letsdebug.net)

  [证书搜索](https://tools.letsdebug.net/cert-search)

  [证书撤销](https://tools.letsdebug.net/cert-revoke)

**Let’s Encrypt官网**

​	[letsencrypt.org](https://letsencrypt.org/zh-cn/getting-started)

**Let's Encrypt 速率限制**

​	注意：如需调试脚本 请使用测试环境API 生产环境申请次数限制比较严格 触发后比较麻烦

​	[letsencrypt rate-limits](https://letsencrypt.org/zh-cn/docs/rate-limits)

**ACME 客户端**

​	[Certbot](https://certbot.eff.org/) 

​	centos系统安装存在依赖问题 使用docker方式 可移植性更强

​	[certbot running-with-docker](https://certbot.eff.org/docs/install.html#running-with-docker)

**Certbot 命令行参数**

​	[certbot-command-line-options](https://certbot.eff.org/docs/using.html#certbot-command-line-options)


[关于通配符证书(泛域名证书)的支持](https://community.letsencrypt.org/t/acme-v2-production-environment-wildcards/55578)

​	已支持，不过需要使用 DNS 记录进行域名所有权的验证。

[Let's Encrypt 证书的兼容性](https://letsencrypt.org/docs/certificate-compatibility/)



# 准备文件

```bash
# ls
aly.py  certbot.sh  Dockerfile  pushcert.sh
```

```bash
# cat Dockerfile 
FROM certbot/certbot:1.20.0
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories \
    && apk add --no-cache --virtual .build-deps gcc \
        linux-headers \
        openssl-dev \
        musl-dev \
        libffi-dev \
        bash \
    && pip install -i https://mirrors.aliyun.com/pypi/simple aliyun-python-sdk-alidns aliyun-python-sdk-core --no-cache-dir \
    && apk del .build-deps
##安装 bash 默认镜像中无sh bash 等命令
##安装 阿里云sdk 申请证书过程需要操作阿里dns
##gcc等其他依赖 是pip安装aliyun模块的时候需要 否则安装无法通过
```

```bash
# cat certbot.sh 
#!/bin/bash
## docker pull certbot/certbot
## docker build -t registry.example.com/certbot:alydns-v0.0.0-20211009 . --memory=8g --memory-swap=8g

work_dir=$(cd `dirname $0`;pwd)
cert_domain="*.dev.example.com"
main_domain="example.com"
host_prefix="_acme-challenge.dev"
staging_server="https://acme-staging-v02.api.letsencrypt.org/directory"    #调试使用
prod_server="https://acme-v02.api.letsencrypt.org/directory"    #正式申请使用
alydns_image="registry.example.com/certbot:alydns-v0.0.0-20211009"

if [[ "$1" == "adddns" ]];then
    python ${work_dir}/aly.py add ${main_domain} ${host_prefix} ${CERTBOT_VALIDATION}
    sleep 5
elif [[ "$1" == "cleandns" ]];then
    python ${work_dir}/aly.py clean ${main_domain} ${host_prefix} ${CERTBOT_VALIDATION}
else
    config_dir=$1
    docker run --network host --rm --name certbot \
    -v "${config_dir}:/etc/letsencrypt" \
    -v "${work_dir}:${work_dir}" \
    ${alydns_image} certonly --manual --agree-tos \
    --server ${prod_server} \
    --register-unsafely-without-email \
    --preferred-challenges dns-01 \
    --manual-public-ip-logging-ok \
    --manual-auth-hook "/bin/sh ${work_dir}/certbot.sh adddns" \
    --manual-cleanup-hook "/bin/sh ${work_dir}/certbot.sh cleandns" \
    --preferred-chain "ISRG Root X1" \
    -d ${cert_domain}
fi
##使用 --preferred-chain "ISRG Root X1" 指定首选证书链解决DST Root CA X3证书9月30过期问题
```

```bash
# cat aly.py 
#!/usr/bin/env python
#coding=utf-8

## pip install aliyun-python-sdk-alidns aliyun-python-sdk-core

import sys , json
from aliyunsdkcore.client import AcsClient
from aliyunsdkalidns.request.v20150109.AddDomainRecordRequest import AddDomainRecordRequest
from aliyunsdkalidns.request.v20150109.DescribeSubDomainRecordsRequest import DescribeSubDomainRecordsRequest
from aliyunsdkalidns.request.v20150109.DeleteSubDomainRecordsRequest import DeleteSubDomainRecordsRequest

class AliDns:
    def __init__(self):
        self.client = AcsClient('access_keyId', 'access_keySecret', 'cn-hangzhou')

    def domain_record(self, action_name, **domain_config):
        if action_name == "describe":
            request = DescribeSubDomainRecordsRequest()
            request.set_accept_format('json')
            request.set_SubDomain(domain_config.get("SubDomain"))
        elif action_name == "delete":
            request = DeleteSubDomainRecordsRequest()
            request.set_accept_format('json')
            request.set_Type("TXT")
            request.set_RR(domain_config.get("RR"))
            request.set_DomainName(domain_config.get("DomainName"))
        elif action_name == "add":
            request = AddDomainRecordRequest()
            request.set_accept_format('json')
            request.set_Type("TXT")
            request.set_TTL("1")
            request.set_RR(domain_config.get("RR"))
            request.set_Value(domain_config.get("Value"))
            request.set_DomainName(domain_config.get("DomainName"))
        response = self.client.do_action_with_exception(request)
        # python2:  print(response)
        # result = json.loads(response)
        
        # python3
        response_str = (str(response, encoding='utf-8'))
        result = json.loads(response_str)

        return result

if __name__ == '__main__':
    # 第一个参数是 action，代表 (add/clean)
    # 第二个参数是域名
    # 第三个参数是主机名（第三个参数+第二个参数组合起来就是要添加的 TXT 记录）
    # 第四个参数是 TXT 记录值
    # sys.exit(0)

    print(sys.argv)
    file_name, cmd, domain_name, host_prefix, certbot_validation = sys.argv
    domain_config = dict(
        DomainName=domain_name,
        RR=host_prefix,
        SubDomain=host_prefix + "." + domain_name,
        Value=certbot_validation,
    )

    domain = AliDns()
    # cmd = "clean"
    if cmd == "add":
        data = domain.domain_record(action_name="describe", **domain_config)
        if data.get("TotalCount") == 0:
            domain.domain_record(action_name="add", **domain_config)
            print("add aly dns TXT record  finlished")
        else:
            print("======> aly dns record is already exist, please check !!!")
    elif cmd == "clean":
        data = domain.domain_record(action_name="describe", **domain_config)
        if data.get("TotalCount") == 0:
            print("======> aly dns record is not exist, please know !!!")
            sys.exit(0)
        domain.domain_record(action_name="delete", **domain_config)
        print("clean aly dns TXT record finlished")
```

```bash
# cat pushcert.sh 
#!/bin/bash
#author: xiedeng1024
#date: 20211009
## 编写脚本 推送证书文件

WORKSPACE=$1
SSLDIR=${WORKSPACE}/archive/dev.example.com    #申请的证书存放位置

function syncssl() {
case $1 in
qa)
    cd ${SSLDIR} && cat fullchain1.pem privkey1.pem > ssl-dev.example.com.pem   #haproxy证书需要fullchain
    [ $? -eq 0 ] && rsync -acvz ${SSLDIR}/ssl-dev.example.com.pem 172.17.6.19:/opt/haproxy/conf/ssl-dev.example.com.pem
    [ $? -eq 0 ] && ssh 172.17.6.19 "/opt/haproxy/sbin/haproxy -c -f /opt/haproxy/conf/haproxy.cfg && systemctl restart haproxy && systemctl status haproxy"
    [ $? -eq 0 ] && echo "haproxy6-19 ssl is ok" || echo "======> haproxy6-19 ssl is failed ,please check"

    nginxdir="/data/nginx-ops-qa-configfiles"
    gitpath="https://gitlab.dev.example.com/OPS/nginx-ops-qa-configfiles.git"
    ;;
office)
    nginxdir="/data/nginx-ops-office-configfiles"
    gitpath="https://gitlab.dev.example.com/OPS/nginx-ops-office-configfiles.git"
    ;;
*)
    echo "the env is not support"
    exit -1
    ;;
esac

letsencrypt_path="conf.d/templates/letsencrypt"    # 上传证书到gitlab指定位置
##cd /data && [ ! -d ${nginxdir} ] && git clone ${gitpath}
cd ${nginxdir} && git pull
[ $? -eq 0 ] && rsync -acvz --delete ${SSLDIR}/ --exclude=.git* ${nginxdir}/${letsencrypt_path}/|sed '/\/$/d'
[ $? -eq 0 ] && tagname="master-v0.0.0-`date +%Y%m%d%H%M%S`-letsencrypt" && git add --all && git commit -m "${tagname}"
[ $? -eq 0 ] && git push origin master && git tag -a ${tagname} -m "${tagname}" && git push origin ${tagname}
}

syncssl qa
syncssl office
```

# 封装新镜像

* 下载指定版本certbot镜像
    
    https://hub.docker.com/r/certbot/certbot/tags
    ```docker pull certbot/certbot:v1.20.0```
* build 本地镜像

  ```bash
  docker build -t registry.example.com/certbot:alydns-v0.0.0-20211009 . --memory=8g --memory-swap=8g
  docker push registry.example.com/certbot:alydns-v0.0.0-20211009
  ```

# 使用 acme-staging-v02.api.letsencrypt.org 测试

```bash
docker run --network host --rm --name certbot -v /root/.jenkins/workspace/ops-nginx/cron-ops-letsencrypt:/etc/letsencrypt -v /data/k8s-ops-deploy-file/scripts/letsencrypt:/data/k8s-ops-deploy-file/scripts/letsencrypt registry.example.com/certbot:alydns-v0.0.0-20211009 certonly --manual --agree-tos --server https://acme-staging-v02.api.letsencrypt.org/directory --register-unsafely-without-email --preferred-challenges dns-01 --manual-public-ip-logging-ok --manual-auth-hook '/bin/sh /data/k8s-ops-deploy-file/scripts/letsencrypt/certbot.sh adddns' --manual-cleanup-hook '/bin/sh /data/k8s-ops-deploy-file/scripts/letsencrypt/certbot.sh cleandns' --preferred-chain 'ISRG Root X1' -d '*.dev.example.com'
-----------------------------------------------------------------------------------------------------------------
Account registered.
Requesting a certificate for *.dev.example.com
Hook '--manual-auth-hook' for dev.example.com ran with output:
 ['/data/k8s-ops-deploy-file/scripts/letsencrypt/aly.py', 'add', 'example.com', '_acme-challenge.dev', 'lAsyMNcAZOEh2TYxzGHt9Ss9cNOHleUxKib26xqhhP0']
 add aly dns TXT record  finlished
Hook '--manual-cleanup-hook' for dev.example.com ran with output:
 ['/data/k8s-ops-deploy-file/scripts/letsencrypt/aly.py', 'clean', 'example.com', '_acme-challenge.dev', 'lAsyMNcAZOEh2TYxzGHt9Ss9cNOHleUxKib26xqhhP0']
 clean aly dns TXT record finlished

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/dev.example.com/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/dev.example.com/privkey.pem
This certificate expires on 2022-01-06.
These files will be updated when the certificate renews.
NEXT STEPS:
- The certificate will need to be renewed before it expires. Certbot can automatically renew the certificate in the background, but you may need to take steps to enable that functionality. See https://certbot.org/renewal-setup for instructions.
Saving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```



# Jenkinsfile

```groovy
# cat jenkinsfile/JenkinsfileLetsencrypt
pipeline {
    options { skipDefaultCheckout(true) }
    agent {label Cluster }
    stages {
        stage('build user') {
          steps {
               script {
                   echo 'workspace cleanup !!!'
                   deleteDir() /* clean up our workspace */
               }
          }
        }
        stage('ExecCertbot') {
            steps {
                script {
                    sh "sh -x /data/k8s-ops-deploy-file/scripts/letsencrypt/certbot.sh $WORKSPACE"
                }
            }
        }
        stage('PushCerts') {
            steps {
                script {
                    sh "sh -x /data/k8s-ops-deploy-file/scripts/letsencrypt/pushcert.sh $WORKSPACE"
                }
            }
        }
    }
    post {
        always {
            script {
                env.BUILD_REAL_URL = sh (script: '''
                    set +x && echo -n ${BUILD_URL} | sed "s#securityRealm/finishLogin&renew=true/##"
                ''', returnStdout: true).trim()
                emailext (
                    body:'''
                        (本邮件是程序自动下发的，请勿回复！)<br/>
                        项目名称：$JOB_BASE_NAME<br/>
                        构建编号：$BUILD_NUMBER<br/>
                        构建地址：''' + env.BUILD_REAL_URL + ''' <br/>
                        构建日志地址：'''  + env.BUILD_REAL_URL + '''console <br/> ''',
                    subject: "构建通知:$JOB_BASE_NAME - Build # $BUILD_NUMBER !",
                    to:'xiedeng1024@example.com.cn'
                )
            }
        }
        failure {
            script {
                env.BUILD_REAL_URL = sh (script: '''
                    set +x && echo -n ${BUILD_URL} | sed "s#securityRealm/finishLogin&renew=true/##"
                ''', returnStdout: true).trim()
                emailext (
                    body:'''
                        (本邮件是程序自动下发的，请勿回复！)<br/>
                        项目名称：$JOB_BASE_NAME<br/>
                        构建编号：$BUILD_NUMBER<br/>
                        构建地址：''' + env.BUILD_REAL_URL + ''' <br/>
                        构建日志地址：'''  + env.BUILD_REAL_URL + '''console <br/> ''',
                    subject: "构建失败通知:$JOB_BASE_NAME - Build # $BUILD_NUMBER !",
                    to:'notice@group.example.com.cn'
                )
            }
        }
    }
}
```

# 问题

**DST Root CA X3证书 9月30 过期问题**

使用 --preferred-chain "ISRG Root X1" 指定首选证书链解决

修复前证书链

```
---
Certificate chain
 0 s:/CN=*.dev.example.com
   i:/C=US/O=Let's Encrypt/CN=R3
 1 s:/C=US/O=Let's Encrypt/CN=R3
   i:/C=US/O=Internet Security Research Group/CN=ISRG Root X1
 2 s:/C=US/O=Internet Security Research Group/CN=ISRG Root X1
   i:/O=Digital Signature Trust Co./CN=DST Root CA X3
---
```

修复后证书链

```
---
Certificate chain
 0 s:CN = *.dev.example.com
   i:C = US, O = Let's Encrypt, CN = R3
 1 s:C = US, O = Let's Encrypt, CN = R3
   i:C = US, O = Internet Security Research Group, CN = ISRG Root X1
---
```

**客户端更新证书链**

```
windows 管理员命令行:
    certutil -verifyCTL AuthRootWU

linux 平台：
   更新系统根证书
    yum install ca-certificates -y    

mac 平台：
    brew install openssl@1.1
```

