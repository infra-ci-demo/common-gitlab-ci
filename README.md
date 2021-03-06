# GitLab CI with container

Prepare the latest docker & docker-compose for CentOS7
```
yum update -y
yum install -y yum-utils device-mapper-persistent-data lvm2 git
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce
```

Configure docker daemon for GitLab container registry.
```
vi /usr/lib/systemd/system/docker.service
---
ExecStart=/usr/bin/dockerd -H unix://
  ↓
ExecStart=/usr/bin/dockerd --insecure-registry 192.168.33.10:4567 -H unix://
---
```
- It's necessary to change to an ip address that can access from external network.


Restart tour system.
```
systemctl enable docker

reboot
```


Take the configure files for gitlab.
```
cd ~/
git clone https://github.com/infra-ci-demo/common-gitlab-ci.git
cd common-gitlab-ci
```

Edit file to adjust your env.
```
vi volumes/gitlab.rb
```
- It's necessary to change to an ip address that can access from external network again.


Run GitLab & CI Runner containers.
```
docker run -d \
           -p 80:80 -p 443:443 -p 4567:4567 \
           -h gitlab.example.com \
           --name gitlab \
           --restart always \
           -v /root/common-gitlab-ci/volumes/gitlab.rb:/etc/gitlab/gitlab.rb:z \
           -v /root/common-gitlab-ci/volumes/data:/var/opt/gitlab:z \
           -v /root/common-gitlab-ci/volumes/logs:/var/log/gitlab:z \
           gitlab/gitlab-ce:11.5.4-ce.0

docker run -d \
           -h gitlab-runner-centos7.example.com \
           --name gitlab-runner-centos7 \
           --restart always \
           -v /var/run/docker.sock:/var/run/docker.sock \
           gitlab/gitlab-runner:v11.5.1

docker run -d \
           -h gitlab-runner-centos6.example.com \
           --name gitlab-runner-centos6 \
           --restart always \
           -v /var/run/docker.sock:/var/run/docker.sock \
           gitlab/gitlab-runner:v11.5.1
```


Register runner to GitLab.
```
docker exec gitlab-runner-centos7 gitlab-runner register \
      --non-interactive \
      --url http://10.208.81.221 \
      --registration-token token-AABBCCDDEE \
      --tag-list docker-centos7 \
      --executor docker \
      --locked=false \
      --docker-image centos:7 \
      --clone-url http://10.208.81.221/ \
      --docker-volumes /var/run/docker.sock:/var/run/docker.sock \
      --docker-privileged=true

docker exec gitlab-runner-centos6 gitlab-runner register \
      --non-interactive \
      --url http://10.208.81.221 \
      --registration-token token-AABBCCDDEE \
      --tag-list docker-centos6 \
      --executor docker \
      --locked=false \
      --docker-image centos:6 \
      --clone-url http://10.208.81.221/ \
      --docker-volumes /var/run/docker.sock:/var/run/docker.sock \
      --docker-privileged=true
```


Additional Runner
```
docker run -d \
           -h staging.example.com \
           --name gitlab-runner-staging \
           --restart always \
           -v /var/run/docker.sock:/var/run/docker.sock \
           gitlab/gitlab-runner:v11.5.1

docker exec gitlab-runner-staging gitlab-runner register \
      --non-interactive \
      --url http://10.208.81.221 \
      --registration-token token-AABBCCDDEE \
      --tag-list staging \
      --executor docker \
      --locked=false \
      --docker-image centos:7 \
      --clone-url http://10.208.81.221/ \
      --docker-volumes /var/run/docker.sock:/var/run/docker.sock \
      --docker-privileged=true
```
