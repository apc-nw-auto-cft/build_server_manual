# How to build gitlab

- [1. OS](#1-os)
- [2. os install後の事前準備](#2-os-install後の事前準備)
- [3. 証明書の作成](#3-証明書の作成)
  - [3.1. CA局開設](#31-ca局開設)
  - [3.2. SV証明書のCSR作成](#32-sv証明書のcsr作成)
  - [3.3. rootCAでSV証明書の発行](#33-rootcaでsv証明書の発行)
- [4. gitlabのinstall](#4-gitlabのinstall)
- [5. gitlab.rbの編集](#5-gitlabrbの編集)
- [6. gitlab\_homeの作成と証明書の配置](#6-gitlab_homeの作成と証明書の配置)
- [7. 設定の反映](#7-設定の反映)
- [8. fqdnでdocker loginできない現象の回避策](#8-fqdnでdocker-loginできない現象の回避策)
- [9. WebUIでUserを作成](#9-webuiでuserを作成)

## 1. OS

NAME="Ubuntu"
VERSION_ID="22.04"
VERSION="22.04.2 LTS (Jammy Jellyfish)"

## 2. os install後の事前準備

```shell
sudo apt update -y
sudo apt upgrade -y
```

## 3. 証明書の作成

### 3.1. CA局開設

```shell
# openssl.cnfで定義されているdirectory名をそのまま使用
cd /etc/ssl
mkdir demoCA
cd demoCA
pwd # /etc/ssl/demoCA
mkdir private

# まずはCA用の秘密鍵を作成
sudo openssl genrsa -aes256 -out ./private/cakey.pem 2048

# CA用の秘密鍵を元にCSRを作成
sudo openssl req -new -key ./private/cakey.pem -out cacert.csr
Enter pass phrase for cakey.pem:  # <=P...!で作成

You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:JP
State or Province Name (full name) [Some-State]:Tokyo
Locality Name (eg, city) []:Chiyoda-ku
Organization Name (eg, company) [Internet Widgits Pty Ltd]:AP-Communications
Organizational Unit Name (eg, section) []:iTOC
Common Name (e.g. server FQDN or YOUR name) []:AP-Communications
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
mbs_gitlab@mbsgitlab:/etc/ssl/demoCA$ 

# CA用の鍵で自己署名したCA証明書を作成

sudo openssl x509 -days 36500 -in cacert.csr -req -signkey ./private/cakey.pem -out cacert.pem
Enter pass phrase for cakey.pem:
Certificate request self-signature ok
subject=C = JP, ST = Tokyo, L = Chiyoda-ku, O = AP-Communications, OU = iTOC, CN = AP-Communications

# のちに必要になってくるfileとdirectoryを作成
sudo touch index.txt
sudo sh -c "echo 00 > ./serial"
sudo mkdir newcerts
```

### 3.2. SV証明書のCSR作成

```shell
cd ../
sudo mkdir sv_certs
cd sv_certs
pwd   # /etc/ssl/sv_certs

# SV用の秘密鍵の作成
sudo openssl genrsa -aes256 -out sv.key 2048
Enter PEM pass phrase:

# 作成した秘密鍵からpassphraseを除去した秘密鍵を作成
sudo openssl rsa -in sv.key -out sv_nopass.key
Enter pass phrase for sv.key:

# SV用秘密鍵をもとにCSRを作成
sudo openssl req -new -key sv_nopass.key -out sv.csr

You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:JP
State or Province Name (full name) [Some-State]:Tokyo
Locality Name (eg, city) []:Chiyoda-ku
Organization Name (eg, company) [Internet Widgits Pty Ltd]:AP-Communications
Organizational Unit Name (eg, section) []:iTOC
Common Name (e.g. server FQDN or YOUR name) []:192.168.79.9
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

### 3.3. rootCAでSV証明書の発行

```shell
# subjectAltNameを設定するためのfileを作成
sudo sh -c "echo 'subjectAltName=IP:192.168.79.9' > san.ext"

# openssl.cnfでは./demoCAがrootのため、demoCAの直上のdirectoryで作業する
cd ../
pwd  # /etc/ssl

# SV証明書の発行
sudo openssl ca -in sv_certs/sv.csr -out sv_certs/sv.pem -days 36500 -extfile demoCA/san.ext 
Using configuration from /usr/lib/ssl/openssl.cnf
Enter pass phrase for ./demoCA/private/cakey.pem:
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number: 1 (0x1)
        Validity
            Not Before: Jun 21 08:54:01 2023 GMT
            Not After : May 28 08:54:01 2123 GMT
        Subject:
            countryName               = JP
            stateOrProvinceName       = Tokyo
            organizationName          = AP-Communications
            organizationalUnitName    = iTOC
            commonName                = 192.168.79.9
        X509v3 extensions:
            X509v3 Subject Alternative Name: 
                IP:192.168.79.9
Certificate is to be certified until May 28 08:54:01 2123 GMT (36500 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated

# SV証明書をtrustoutする（Gitlabで使用するために必要な作業）
cd sv_certs
sudo openssl x509 -outform pem -in sv.pem -out sv_trustout.crt -trustout

```

## 4. gitlabのinstall

[official](https://about.gitlab.com/install/#ubuntu)とは少々異なる手順でinstallした。

```shell

sudo apt-get update
sudo apt-get install -y curl openssh-server ca-certificates tzdata perl

sudo apt-get install -y postfix

curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.deb.sh | sudo bash

sudo apt-get install gitlab-ee

```

## 5. gitlab.rbの編集

```shell
external_url 'https://192.168.79.9'
gitlab_rails['time_zone'] = 'Asia/Tokyo'
gitlab_rails['gitlab_shell_ssh_port'] = 10022

registry_external_url 'https://192.168.79.9:5050'
gitlab_rails['registry_enabled'] = true
gitlab_rails['registry_host'] = "192.168.79.9"
gitlab_rails['registry_port'] = "5050"
nginx['ssl_certificate'] = "/etc/gitlab/ssl/192.168.79.9.crt"
nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/192.168.79.9.key"
letsencrypt['enable'] = false
```

## 6. gitlab_homeの作成と証明書の配置

```shell
# 証明書配置用のdirectoryを作成
sudo mkdir /etc/gitlab/ssl

# 証明書と鍵をcp
sudo cp /etc/ssl/sv_certs/sv_nopass.key /etc/gitlab/ssl/192.168.79.9.key
sudo cp /etc/ssl/sv_certs/sv_trustout.crt /etc/gitlab/ssl/192.168.79.9.crt

```

## 7. 設定の反映

```shell
sudo gitlab-ctl reconfigure
```

## 8. fqdnでdocker loginできない現象の回避策

結局やらなくなった作業

```shell
sudo vi /var/opt/gitlab/registry/config.yml

# これを
auth:
  token:
    realm: https://192.168.79.9/jwt/auth

# これに変更

auth:
  token:
    realm: https://gitlab-01.apc.itoclab.jp/jwt/auth

sudo gitlab-ctl restart

```

## 9. WebUIでUserを作成

root userは24h後にはpasswordを確認する方法がなくなるので、userを作成しておく

1. https://\<IP\> からrootと前段で確認したpasswordでlogin
2. configure Gitlab
3. 左ペインでuser -> 右上のNew user ボタンをクリック
4. Administrator で作成
5. 作成したUserをEditしてpasswordを設定（すぐに再設定させられるので、本当に使いたいPWは設定しないが吉）
6. logoutして作成したuserでlogin
7. passwordの再設定
