Oracle 6.6 + Oracle 12c DB Docker builder  
=============================================

This is repository with scripts to build Vagrant virtual images and install Oracle Database 12c

    REPOSITORY        TAG      IMAGE ID       VIRTUAL SIZE
    oracle/database   latest   11222af458e4   8.605 GB  - Oracle Database 12c + orcl12c DB instance
    oracle/weblogic   latest   6f4cfcd531da   1.876 GB  - weblogic (only installation /opt/weblogic)
    oracle/java       latest   835362a5b3ef   1.032 GB  - JDK + oracle-rdbms-server-12cR1-preinstall
    oraclelinux       6.6      d56e767abb61   319.4 MB  - base OL image


Original Docker file and script based on work done by Yasushi YAMAZAKI https://github.com/yasushiyy/vagrant-docker-oracle12c
However VM boxes built by http://packer.io/ Oracle Linux 6.6  as well as official oracle 6.6 Docker images  
http://public-yum.oracle.com/docker-images/

Versions
-------------------------------------------------

- VirtualBox (4.3.12)
- Packer  (0.7.5)
- Weblogic  12.1.3  :: 
- JDK       1.7.51  :: 
- Oracle linux 6.6  :: http://public-yum.oracle.com/docker-images/OracleLinux/OL6/oraclelinux-6.6.tar.xz 
- Oracle Ent   12c  :: http://www.oracle.com/technetwork/database/enterprise-edition/downloads/index.html 
- Docker       1.4  :: - installed on separate btrfs partition /dev/sdb mounted /dev/sdb /var/lib/docker 
                     : - started with flags: -H tcp://0.0.0.0:4243 -H unix:///var/run/docker.sock -s btrfs

_Optional (if you use OSX)_

- Parallels Version 10.1.1 (28614)  
- Parallels Virtualization SDK 10 for Mac (10.1.2-28859)


Changelog
-------------------------------------------------
- 21.12.2014 - Inintial version fixes 
- 22.12.2014 - Fully automated install script for oracle 
- 22.12.2014 - Added separate btrfs partition for Docker storage
- 11.01.2015 - VM image tmp folders optimizations Docker installation moved to packer stage.
- 12.01.2015 - Added JDK 7u71 and Weblogic 12.1.3 Docker files to the provision 

Installation
-------------------------------------------------

0. __Install Required Tools__
    
- _Vagrant_
 Download and install version for your platform.
 https://www.vagrantup.com/downloads.html

- _Virtualbox_
 It's default virtualizaton provider, 
 https://www.virtualbox.org

- _Parallels_
 In case you use Parallels you need to download also SDK:
 http://www.parallels.com/eu/products/desktop/download/

_Packer_
 http://www.packer.io/docs/installation.html

or with homebrew ->

    $ brew tap homebrew/binary
    $ brew install packer


Building base VM images  
-------------------------------------------------
Build for all platforms: 

    cd bento/packer/
    packer build oracle-6.6-x86_64.json

Or only specify builder type:

    packer build -only=virtualbox-iso oracle-6.6-x86_64.json
    packer build -only=parallels-iso oracle-6.6-x86_64.json


Install Database in the container
-------------------------------------------------

1. __Download the database binary (12.1.0.2.0) from below.__  
2. __Most of the steps already automated by vm_start.sh script__

http://www.oracle.com/technetwork/database/enterprise-edition/downloads/index.html

* linuxamd64_12c_database_1of2.zip
* linuxamd64_12c_database_2of2.zip

_Below steps are already part of Vagrant provision script_

Build Process
-------------------------------------------------
vm_start.sh will detect if it is first run and create all docker container images at first run.
After that it will destroy the VM and load create images from tar.xz files 
This allows to save VM disk space and keep Docker VM small

Building Containers
-------------------------------------------------

__1. Start Docker VM and Download OL6.6 Docker image__

```    
    $vagrant ssh
    cd /vagrant
    wget http://public-yum.oracle.com/docker-images/OracleLinux/OL6/oraclelinux-6.6.tar.xz
```

__2. Build Containers__

```    
    sudo docker build --no-cache -t="oracle/java-7u71" /vagrant/docker/java-7u71/
    sudo docker build --no-cache -t="oracle/weblogic12" /vagrant/docker/weblogic-12/
    sudo docker build --no-cache -t="oracle/oracle12c" /vagrant/docker/oracle-c12/
```

__3. This is will run automatic installation of software and create database :__

```
sudo docker run --privileged -h db12c --name database -p 1521:1521 -t -i -v /vagrant:/vagrant oracle/oracle12c /bin/bash /vagrant/scripts/install_oracle.sh
```

__4. Save our installation as a new image__
```
docker export database | docker import - oracle/database

#remove build container
sudo docker kill database && docker rm database
```

__Examples:__ 

Using Database container in detached mode
-------------------------------------------------

Keep in mind docker need one main process to run detached.
I use /bin/bash for that purpose. This allows to connect to existing container later on.

```
# The following command is to get latest container ID `docker ps --no-trunc -aq`
# You can replace it with any actual container id

#list of all containers running
docker ps -a

#to start images in detached mode - with container name  [database] or [weblogic]
sudo docker run --privileged -h db12c --name database -p 1521:1521 -t -d oracle/database /bin/bash
sudo docker run --privileged -h crm92 --name weblogic  -p 8001:8001 -p 8002:8002 -t -d oracle/weblogic /bin/bash

#execute simple command
sudo docker exec -i weblogic java -version

#connect to existing running container shell
sudo docker exec -i -t database /bin/bash

#check running processes
sudo docker exec -i database ps -ef

#start db
sudo docker exec -i database /bin/bash  /etc/init.d/dbstart

#stop db
sudo docker exec -i database /etc/init.d/dbstart stop

#kill container
docker kill database

#remove container
docker rm database

```

__Test connection__
```
su - oracle

sqlplus system/oracle@localhost:1521/db12c

SQL> select count(1) from user_tables;

  COUNT(1)
----------
       178

SQL> show parameter inmemory

SQL> exit
```
