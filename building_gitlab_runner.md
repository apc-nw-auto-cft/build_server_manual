# How to build gitlab runner

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

## gitlab runnerのinstall

```shell
sudo sh -c 'curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | bash'
sudo apt install -y gitlab-runner
```

## gitlabのrootCA証明書の配置

```shell
scp mbs_gitlab@gitlab-01.apc.itoclab.jp:~/certs/cacert.pem .
mv cacert.pem /usr/share/ca-certificates/cacert.crt
sudo update-ca-certificates
```

## dockerでrootCAを使用するために、証明書のsymbolic linkを作成

```shell
sudo mkdir /etc/docker/certs.d
sudo mkdir /etc/docker/certs.d/192.168.79.9:5050
sudo ln -s /etc/ssl/certs/ca_apc.itoclab.jp.pem /etc/docker/certs.d/192.168.79.9\:5050/ca_apc.itoclab.jp.crt
```

## docker loginできることを確認

```shell
docker login gitlab-01.apc.itoclab.jp:5050 -u shi_ono
```

## runnerのregister

```shell
sudo gitlab-runner register

Enter the GitLab instance URL (for example, https://gitlab.com/):
https://192.168.79.9
Enter the registration token:
glrt-7HzyY4X7uz8WyguK8z4_
Verifying runner... is valid                        runner=7HzyY4X7u
Enter a name for the runner. This is stored only in the local config.toml file:
[mbsgitlabrunner]: 
Enter an executor: docker, ssh, virtualbox, instance, custom, docker-windows, parallels, shell, docker-autoscaler, docker+machine, kubernetes:
docker
Enter the default Docker image (for example, ruby:2.7):
ubuntu:latest
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!
 
Configuration (with the authentication token) was saved in "/etc/gitlab-runner/config.toml"
```
