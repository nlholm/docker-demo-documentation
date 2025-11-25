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

![kuva1](./Pictures/kuva1.png)  
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
        Suites: {{ grains['oscodename'] }} # Salt automatically detects the OS name using grains, so we don't need to hard-code "bookworm"
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
    - version:
      - docker-ce: 29.0.4-1~debian.12~bookworm
      - docker-ce-cli: 29.0.4-1~debian.12~bookworm

docker_service:
  service.running:
    - name: docker
    - enable: True
```

Now we can try how it works on our minion-virtualmachine by running command `sudo salt 'minion1' state.apply docker` from master.  






## References

Dockerdocs. Install Docker Engine on Debian. URL: https://docs.docker.com/engine/install/debian/#install-using-the-repository. Accessed: 25.11.2025  


