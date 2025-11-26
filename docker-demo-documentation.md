update: 25.11.2025  

# Installing Docker to Debian Bookworm64

Lets start the installation of Debian by hand before automation. I started working with clean Debian Bookworm64 virtualmachine that has pre-installed wget, curl, gnup2, micro and Salt-master. I also installed bash-compleation in to virtualmachine. (note to add on Vagrantfile)   

Let's log in to our virtualmachine through Windows cmd with command `vagrant ssh master`. Next command are recommended to be done at order:  

Inside of virtualmachine - vagrant@master  

1. `sudo apt-get update`
2.  `sudo apt install ca-certificates curl`
3.  `sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc`
4.  `sudo chmod a+r /etc/apt/keyrings/docker.asc`
5.  `sudo tee /etc/apt/sources.list.d/docker.sources <<EOF Types: deb URIs: https://download.docker.com/linux/debian Suites: $(. /etc/os-release && echo "$VERSION_CODENAME") Components: stable Signed-By: /etc/apt/keyrings/docker.asc EOF`
6.  `sudo apt-get update`
7.  `VERSION_STRING=5:29.0.4-1~debian.12~bookworm`
8.  `sudo apt install docker-ce=$VERSION_STRING docker-ce-cli=$VERSION_STRING containerd.io docker-buildx-plugin docker-compose-plugin`
9.  `sudo systemctl status docker`  

![kuva1](./img/kuva1.png)  
And we have Docker running!  

10. Next we can try `sudo docker run hello-world` that downloads the test image and runs it in a container. You should get message that tells your installation has worked correctly.
(Dockerdocs)

## Automation with Salt

Lets start by creating directory path: /srv/salt/docker/ by using command `sudo mkdir -p /srv/salt/docker` and make init.sls inside of the docker directory.  

Version_1
```
docker_depencies:
  pkg.installed:
    - pkgs:
      - ca-certificates
      - curl
      - gnupg

/etc/apt/keyrings:
  file.directory:
    - mode: 0755

docker-gpg-key:
  cmd.run:
    - name: |
        curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc && \
        chmod a+r /etc/apt/keyrings/docker.asc
    - unless: test -f /etc/apt/keyrings/docker.asc

/etc/apt/sources.list.d/docker.sources:
  file.managed:
    - contents: |
        Types: deb
        URIs: https://download.docker.com/linux/debian
        Suites: {{ grains['oscodename'] }}
        Components: stable
        Signed-By: /etc/apt/keyrings/docker.asc

docker_apt-update:
  cmd.run:
    - name: sudo apt-get update
    - onchanges:
      - file: /etc/apt/sources.list.d/docker.sources

docker_packages:
  pkg.installed:
    - pkgs:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-buildx-plugin
      - docker-compose-plugin

docker_service:
  service.running:
    - name: docker
    - enable: True
```

Now we can try how it works on our minion-virtualmachine by running command `sudo salt 'minion1' state.apply docker` from master.  

Looks like it worked!  

![kuva2](./img/kuva2.png)  

Lets log in to minion and see if Docker runs there. Lets try command `sudo docker run hello-world` and it gives answer. It seems that our installation is working as planned.  

![kuva3](./img/kuva3.png)  


# Serving static website in Nginx-container

Let's start with docker installed in our Salt-Master. We have already automated installation of Docker so we can run salt command locally. `sudo salt-call --local state.apply docker` and now we have installed Docker in our master.  

Now we make folder to our home directory for docker-compose -file so we can set up multiple containers and we also include our index.html, styles.css and image in that directory. Run command `mkdir -p /home/vagrant/nginx-demo/site/images`.  

Let's create docker-compose.yml -file that creates three nginx web-services.  

```
services:
  web1:
    image: nginx
    container_name: nginx-web1
    ports:
      - "8081:80"
    volumes:
      - ./site:/usr/share/nginx/html:ro

  web2:
    image: nginx
    container_name: nginx-web2
    ports:
      - "8082:80"
    volumes:
      - ./site:/usr/share/nginx/html:ro

  web3:
    image: nginx
    container_name: nginx-web3
    ports:
      - "8083:80"
    volumes:
      - ./site:/usr/share/nginx/html:ro
```

Then we want to add our depencies to the webpage.  

![kuva4](./img/kuva4.png)  

Now we have compelated our nginx-demo directory and its time to try it. Run this command in nginx-demo directory: `sudo docker compose up -d`.  

After that we can run `sudo docker ps` to see what containers are running and `curl localhost:8080` to see if our webpage is up and running.  

![kuva5](./img/kuva5.png)  

As we can see we have three Nginx containers up and running and our webpage responds correctly.  

## Automating nginx with Salt

I started automation by creating new directory nginx-web under /srv/salt/. Firstly I copied all the files that we used before to our module.  

![kuva6](./img/kuva6.png)  

Next I created init.sls -file inside nginx-web -module.  

```
/home/vagrant/nginx-demo:
  file.directory:
    - user: vagrant
    - group: vagrant
    - mode: 755

/home/vagrant/nginx-demo/docker-compose.yml:
  file.managed:
    - source: salt://nginx-web/docker-compose.yml
    - user: vagrant
    - group: vagrant
    - mode: 644
    - require:
      - file: /home/vagrant/nginx-demo

/home/vagrant/nginx-demo/site:
  file.recurse:
    - source: salt://nginx-web/site
    - user: vagrant
    - group: vagrant
    - file_mode: 644
    - dir_mode: 755
    - require:
      - file: /home/vagrant/nginx-demo

nginx_web_up:
  cmd.run:
    - name: docker compose up -d
    - cwd: /home/vagrant/nginx-demo
    - require:
      - file: /home/vagrant/nginx-demo/docker-compose.yml
      - file: /home/vagrant/nginx-demo/site
    - unless: "docker ps --format '{{.Names}}' | grep -q nginx-web1"
```

![kuva7](./img/kuva7.png)  

And it works! Now we can copy this module to our repository.  

## Testing

Lets try to run our modules to minion virtualmachine. Lets run command `sudo salt 'minion1' state.apply` in our master machine.  

![kuva8](./img/kuva8.png)  

Seems to be working. I run the command twice and got succeed 11 and 0 changed, so it seems to be idempotent. Time to log in to our minion and see if we have webpage at localhost:8080.  

![kuva9](./img/kuva9.png)  

Works well!  

# Nginx as proxy and loadbalancer

The idea is to install Nginx as proxy and loadbalancer that servers our three backend web-servers that run in Docker containers.  

Lets start to make this without automation first. Run `sudo apt-get update` `sudo apt-install nginx`. After that we can make new module for salt in /srv/salt/nginx-proxy, so we can save our configuration easily there.   

After installation we want to tweak nginx.conf that is found in /etc/nginx/nginx.conf.

```
worker_processes 1;


events {

}


http {
        include mime.types;

        upstream container_cluster {
                server 127.0.0.1:8081;
                server 127.0.0.1:8082;
                server 127.0.0.1:8083;
        }

        server {

                listen 80;
                server_name localhost;

                location / {

                        proxy_pass http://container_cluster;
                        proxy_set_header Host $host;
                        proxy_set_header X-Real-IP $remote_addr;

                }

        }

}

```

Here we have setup:  
1. Worker processes to one, so only on http-request per worker
2. Upstream acts as load balancer. Default is round robin, so it will pick one server at time to send the requests.
3. Location as reverse proxy. It will get the request from client and forwards it to our servers.
4. Nginx will listen port 80 and serve localhost-address.

## Lets test it!

Firstly we want to shutdown our containers with command `sudo docker compose down` and back to up with `sudo docker compose up -d`. Also we just made changes to /etc/conf-file so lets restart the daemon with `sudo systemctl restart nginx`.  

![kuva10](./img/kuva10.png)  

Localhost answers with desired webpage in every port. `curl localhost` also delivers same page, so reverse proxy seems to be working.  

LOAD BALANCING TESTI KESKEN
for i in $(seq 1 10); do curl -s http://localhost > /dev/null; done
sudo docker logs nginx-web1 | grep GET
sudo docker logs nginx-web2 | grep GET
sudo docker logs nginx-web3 | grep GET

## References

Dockerdocs. Install Docker Engine on Debian. URL: https://docs.docker.com/engine/install/debian/#install-using-the-repository. Accessed: 25.11.2025  

Manandhar, G. 2024. How to use Nginx with Docker Compose effectively with examples. URL: https://geshan.com.np/blog/2024/03/nginx-docker-compose/. Accessed: 25.11.2025  

TechWorld with Nana. Full NGINX Tutorial - Demo Project with Node.js, Docker. URL: https://www.youtube.com/watch?v=q8OleYuqntY&t=3980s. Accessed: 25.11.2025  




