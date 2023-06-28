# How to build gitlab using docker

## OS

NAME="Ubuntu"
VERSION_ID="22.04"
VERSION="22.04.2 LTS (Jammy Jellyfish)"

## os install後の事前準備

```shell
sudo apt update -y
sudo apt upgrade -y
```

## 古いdockerのremoveと新しいdockerのinstall

[official](https://docs.docker.com/engine/install/ubuntu/)を参照

```shell
# 古いdockerのremove

for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done

# install前の準備

sudo apt install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update -y

# install

sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable docker
sudo systemctl start docker

# dockerをsudoなしで実行するための手順
sudo groupadd docker
sudo usermod -aG docker $USER
```

## 証明書の作成

### CA局開設

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

### SV証明書のCSR作成

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
Common Name (e.g. server FQDN or YOUR name) []:*.apc.itoclab.jp
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

### rootCAでSV証明書の発行

```shell
# subjectAltNameを設定するためのfileを作成
sudo sh -c "echo 'subjectAltName=DNS:*.apc.itoclab.jp' > demosan.ext"

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
            commonName                = *.apc.itoclab.jp
        X509v3 extensions:
            X509v3 Subject Alternative Name: 
                DNS:*.apc.itoclab.jp
Certificate is to be certified until May 28 08:54:01 2123 GMT (36500 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated

# SV証明書をtrustoutする（Gitlabで使用するために必要な作業）
cd sv_certs
sudo openssl x509 -outform pem -in sv.pem -out sv_trustout.crt -trustout

```

## gitlab_homeの作成と証明書の配置

```shell
# gitlab_homeのdirectoryを作成

mkdir /home/mbs_gitlab/gitlab_home

# 証明書配置用のdirectoryを作成
mkdir /home/mbs_gitlab/gitlab_home/ssl

sudo cp /etc/ssl/sv_certs/sv_nopass.key /home/mbs_gitlab/gitlab_home/ssl/<url>.key
sudo cp /etc/ssl/sv_certs/sv_trustout.crt /home/mbs_gitlab/gitlab_home/ssl/<url>.crt

```

## docker-compose.ymlと.env fileの作成

docker-compose.yml

```yaml
---
version: '3.6'
services:
  web:
    image: 'gitlab/gitlab-ee:latest'
    restart: always
    hostname: 'gitlab-01'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://${GITLAB_IP}'
        gitlab_rails['gitlab_shell_ssh_port'] = 10022
        gitlab_rails['time_zone'] = 'Asia/Tokyo'
        gitlab_rails['registry_enabled'] = true
        gitlab_rails['registry_api_url'] = 'https://${GITLAB_URL}:5050'
        nginx['ssl_certificate'] = '/etc/gitlab/ssl/${GITLAB_URL}.crt'
        nginx['ssl_certificate_key'] = '/etc/gitlab/ssl/${GITLAB_URL}.key'
        registry_external_url 'https://${GITLAB_URL}:5050'
        letsencrypt['enable'] = false
        # Add any other gitlab.rb configuration here, each on its own line
    ports:
      - '5050:5050'
      - '4443:443'
      - '10022:22'
    volumes:
      - '$GITLAB_HOME/config:/etc/gitlab'
      - '$GITLAB_HOME/logs:/var/log/gitlab'
      - '$GITLAB_HOME/data:/var/opt/gitlab'
```

.env

```shell
GITLAB_URL=gitlab-01.apc.itoclab.jp
GITLAB_IP=192.168.79.9
GITLAB_HOME=/home/mbs_gitlab/gitlab_home
```

## docker compose up

```shell
# 実行後10分くらい待つ
docker compose up -d

# コンテナ内部のファイルを編集
docker exec --env_file ./.env mbs_gitlab-web-1 sh -c 'sed -i -e "s/$GITLAB_IP\/jwt\/auth/$GITLAB_URL:4443\/jwt\/auth/" /var/opt/gitlab/registry/config.yml; gitlab-ctl restart'

# rootのpassword確認(fileが24hで消えるらしいので注意)
docker exec mbs_gitlab-web-1 cat /etc/gitlab/initial_root_password | grep Password:

```

## WebUIでUserを作成

root userは24h後にはpasswordを確認する方法がなくなるので、userを作成しておく

1. https://\<IP\> からrootと前段で確認したpasswordでlogin
2. configure Gitlab
3. 左ペインでuser -> 右上のNew user ボタンをクリック
4. Administrator で作成
5. 作成したUserをEditしてpasswordを設定（すぐに再設定させられるので、本当に使いたいPWは設定しないが吉）
6. logoutして作成したuserでlogin
7. passwordの再設定
