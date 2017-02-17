---
title: How to build this site
updated: 2016-10-21 01:24
---

To my honor, I just set this site, and i'll write down all the things about how to build this web by digitalocean, docker, gogs, jekyll, nginx, namecheap.

Let's start.

## Create a VPS
First of all, we need a VPS(Virtual Personal Server). I used use Aliyun Service, but for some policy, we have to register our website to the government, otherwise our web site will be block with a Slogan "According some layws and rules, Please record your website". Without record, you can't visit the website by domain. so, we choose some service abroad, like digitalocean or Linode.

i create a Droplet(VPS) use Centos, so you should create a droplet and follow the tutorial like following link, of course you can use other linux distribution, like ubuntu or debian, maybe you can choose CoreOS, which support docker in operating system level.

> 
[initial-server-setup-with-centos-7](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-centos-7 "Initial Server Setup with CentOS 7")

> [additional-recommended-steps-for-new-centos-7-servers](https://www.digitalocean.com/community/tutorials/additional-recommended-steps-for-new-centos-7-servers "Additional Recommended Steps for New CentOS 7 Servers")

That all about digitalocean. remember your droplet ip address and your sudo user password.

> caution: we use docker for nginx and gogs, but docker requires a 64-bit OS and version 3.10 or higher of the Linux kernel, So before create VPS (but digitalocean support Droplet change kernel online), caution the kernel and 64-bit.

## Install Docker
because my platform is Centos 7, so i follow the [CentOS install docker](https://docs.docker.com/engine/installation/linux/centos/) .

if you choose other OS, you can google how to install docker in your OS . After this, I recommend you take a Snapshot for your VPS, if you don't know how to do , google how to take a snapshot in digitalocean( or other service provider )

## Get a domain

next, you need a domain name. visit [namecheap](https://www.namecheap.com/) or [name](https://www.name.com/) , purchase a domain, maybe .me, .club, or .online is cheaper.

At this time, after purchase,  visit the Dashboard, try to manager your domain. first, check the Domain nameservers, at namecheap, choose "Namecheap BasicDNS", and then modify the Host Recrods.

This is my setting:


| Type        | host           | value|
| ------------- |:-------------:| -----:|
| A Record      |       www    |  {your VPS server IP address}|
| A Record      |      api     | {your VPS server IP address} |
| A Record      |     blog     |  {your VPS server IP address}|
| A Record      |     @        |  {your VPS server IP address}|

All the subdomains , like www.example.com, blog.example.com are directed to your server indifferent , but of course the different subdomain should show different content. So we need NGINX Reverse Proxy.

## Get Certificate
before Set nginx reverse proxy, we have something else to do. use let's encrypt to encrypt our website connection. This part i do get important use from this artical [LETS ENCRYPT WITH DOCKER NGINX PROXY](https://zettabyte.me/lets-encrypt-with-docker-nginx-proxy/) you also can reference this.

but i just just use his method to get certificate, and the next, i'll explanation the nginx-reverse proxy and other website docker.

> Caution:  The certificate Let's encrypt provided Only three months validity,  you should re encrypt after three months late.


## Nginx Reverse Proxy
Because we install docker, and we are ready for the certificate, so we can do this easy for one command.

with my method, create  `/nimo_nginx_confAndcontent/html` `/nimo_nginx_confAndcontent/conf.d` for our proxy-nginx local volume, maybe you should read some document about what is docker.  

```
docker run -d -p 80:80 -p 443:443 --name=proxy --restart=always -v /var/local/nginx/certs:/etc/nginx/certs -v /etc/letsencrypt:/etc/letsencrypt -v /var/local/proxy-confs:/etc/nginx/vhost.d:ro -v /var/run/docker.sock:/tmp/docker.sock:ro -v /nimo_nginx_confAndcontent/html:/usr/share/nginx/html -v /nimo_nginx_confAndcontent/conf.d:/etc/nginx/conf.d nginx
```

> caution: in my method, I put the certificate in `/etc/letsencrypt/live/` , and i create a soft link in `/var/local/nginx/certs/`

and then is the nginx config: cd `/nimo_nginx_confAndcontent/conf.d/` set a config file `default.conf` like this:


```
# If we receive X-Forwarded-Proto, pass it through; otherwise, pass along the
# scheme used to connect to this server
map $http_x_forwarded_proto $proxy_x_forwarded_proto {
  default $http_x_forwarded_proto;
  ''      $scheme;
}
# If we receive Upgrade, set Connection to "upgrade"; otherwise, delete any
# Connection header that may have been passed to this server
map $http_upgrade $proxy_connection {
  default upgrade;
  '' close;
}
gzip_types text/plain text/css application/javascript application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;
log_format vhost '$host $remote_addr - $remote_user [$time_local] '
                 '"$request" $status $body_bytes_sent '
                 '"$http_referer" "$http_user_agent"';
access_log off;
# HTTP 1.1 support
proxy_http_version 1.1;
proxy_buffering off;
proxy_set_header Host $http_host;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $proxy_connection;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $proxy_x_forwarded_proto;
# Mitigate httpoxy attack (see README for details)
proxy_set_header Proxy "";
server {
    server_name _; # This is just an invalid value which will never trigger on a real hostname.
    listen 80;
    access_log /var/log/nginx/access.log vhost;
    return 503;
}
upstream website{
            # nginx (this is www.nimohunter.com and nimohunter.com nginx-web-docker-ip)
            # !!! need modify !!!
            server 172.17.0.3:80; 
}

server {
    server_name nimohunter.com www.nimohunter.com;
    listen 80 ;
    access_log /var/log/nginx/access.log vhost;
    return 301 https://$host$request_uri;
}

server {
	# !!! need modify !!! modify to your domain
    server_name example.com www.example.com;
    listen 443 ssl http2 ;
    access_log /var/log/nginx/access.log vhost;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers 'ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS';
    ssl_prefer_server_ciphers on;
    ssl_session_timeout 5m;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;
    ssl_certificate /etc/nginx/certs/shared.crt;
    ssl_certificate_key /etc/nginx/certs/shared.key;
    add_header Strict-Transport-Security "max-age=31536000";
    location / {
        proxy_pass http://website;
    }
}
```

focus on something like `!!! need modify !!!` and we should modify this in next part.

## Nginx Web Docker
After the nginx-reverse-proxy, we can create our website finally! you can use apache2 or nginx for your website. in this case, i still use nginx.

Create a folder in `/nimo_www/html`   (of course you can use another folder), and then run docker command

```
docker run -d --restart=always --name=web -v /nimo_www/html:/usr/share/nginx/html  nginx
```


and then, get the this docker's ip use :

```
docker inspect --format='{{.NetworkSettings.IPAddress}}' $CONTAINER_ID
```

get the IP address like `172.17.0.3` modify these in `default.conf` 

---
After these, you can visit your domain , you can get a security SSL https about a page "Welcome use nginx." but what is our blog?  you should read the follow. 

let's continue.

## Use Jekyll !
read some document about [jekyll](http://jekyllrb.com/) , our website need is `_site` folder. For a easy way , visit [jekyll theme](http://jekyllthemes.org/), clone a theme you like and modify.

> read  [jekyll Official website ](http://jekyllrb.com/)  for help, this is a lot help for you.

After you see your jekyll website in local machine. your can put all the file in `_site` to your VPS's `/nimo_www/html`, then you can see your blog.

But, This is stupid, This means you should put your website manually, how can i update my website automatic refreshing ? 
use gogs.

## use Gogs！
you can think Gogs as a private github, and with docker, you can install gogs in one  command.

```
mkdir /var/data/gogs
docker run -d --name=gogs -p 10022:22 -p 3000:3000 -v /var/data/gogs:/data gogs/gogs
```

that is all, visit your Ip address:3000 , make some initialization. i get some useful info from [使用Docker和Hexo打造DevOps个人静态博客](http://www.onlyeric.com/2016/05/03/%E4%BD%BF%E7%94%A8Docker%E5%92%8CHexo%E6%89%93%E9%80%A0DevOps%E4%B8%AA%E4%BA%BA%E9%9D%99%E6%80%81%E5%8D%9A%E5%AE%A2-%E4%B8%89-%E6%9C%8D%E5%8A%A1%E7%AB%AF%E7%AF%87/)


```
Database Type==>SQLite3
Path==>data/gogs.db
Application Name==>Gogs:Go Git Service
Repository Root Path==>/data/git/gogs-repositories
Run User==>git
#domain can use your domain, or IP address
Domain==>126.154.12.45
SSH Port==>10022
HTTP Port==>3000
Application URL==>http://126.154.12.45
Log Path==>/app/gogs/log
Username==> 
Passoword==> 
Confirm Password==> 
Admin Email==>
```



and then, set git hooks. Git hook is a script for Git server , some detail you can visit [8.3 Customizing Git - Git Hooks](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks)

at this time, we use the function which git provided `post-receive` , which will be call after a new push. that's what we need.

```
docker-enter gogs
```

> docker-enter is a small tool for docker, much better than docker attach.
> intall tools:
```
wget -P ~ https://github.com/yeasy/docker_practice/raw/master/_local/.bashrc_docker;
echo “[ -f ~/.bashrc_docker ] && . ~/.bashrc_docker” >> ~/.bashrc; source ~/.bashrc
```

then, into gogs-docker, add a script

```
su - git
vi deploy.sh
chmod u+x deploy.sh
```

here is `deploy.sh` :
```
#!/bin/bash
cd /data/git
#modify!!! git chone from your gogs blog
git clone ssh://git@gogs.onlyeric.com:10022/ericcheung/blog.git
# scp the blog content to nginx-web content directory
scp -r blog/_site/* user@example.com:/var/data/www
rm -rf blog
```

Because we use `scp`, so we need add key trust between docker-gogs and local VPS host, do these follow:
```
docker-enter gogs
ssh-keygen -t rsa
ssh-copy-id -i ~/??Modify?/id_rsa.pub  user@yourdomain
#maybe you will use: ssh-copy-id -i ~/??Modify?/id_rsa.pub -p 2451 user@yourdomain
```

after this , you can try ssh to local VPS in docker container-gogs. test it, if the result is ok, so this `scp` can work, and the `deploy.sh` can work, then all the thing is done.





 
