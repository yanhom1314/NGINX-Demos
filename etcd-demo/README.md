# Demo to show Nginx Plus Dynamic Reconfiguration API (upstream_conf) with etcd

This demo shows NGINX Plus being used in conjuction with etcd, a distributed, consistent key-value store for shared configuration and service discovery. This demo is based on docker and spins'
up the following containers:

* [etcd](https://github.com/coreos/etcd) for service discovery
* [Registrator](https://github.com/gliderlabs/registrator) to register services with etcd. Registrator monitors for containers being started and stopped and updates key-value pairs in etcd when a container changes state.
* [nginxdemos/hello](https://hub.docker.com/r/nginxdemos/hello/) as a NGINX webserver that serves a simple page containing its hostname, IP address and port to simulate backend servers
* and of course [NGINX Plus](http://www.nginx.com/products) R8
The demo is based off the work described in this blog post: [Service Discovery for NGINX Plus with etcd](https://www.nginx.com/blog/service-discovery-nginx-plus-etcd/)
 
## Setup Options:

### Fully automated Vagrant/Ansible setup:

Install Vagrant using the necessary package for your OS:

https://www.vagrantup.com/downloads.html

1. Install provider for vagrant to use to start VM's.  

     The default provider is VirtualBox [Note that only VirtualBox versions 4.0 and higher are supported], which can be downloaded from the following link:

     https://www.virtualbox.org/wiki/Downloads

     A full list of providers can be found at the following page, if you do not want to use VirtualBox:

     https://docs.vagrantup.com/v2/providers/

1. Install Ansible:

     http://docs.ansible.com/ansible/intro_installation.html

1. Clone demo repo

     ```$ git clone https://github.com/nginxinc/NGINX-Demos.git```

1. Copy ```nginx-repo.key``` and ```nginx-repo.crt``` files for your account to ```~/NGINX-Demos/ansible/files/```

1. Move into the etcd-demo directory and start the Vagrant vm:

     ```
     $ cd ~/NGINX-Demos/etcd-demo
     $ vagrant up
     ```
     The ```vagrant up``` command will start the virtualbox VM and provision it using the ansible playbook file ~/NGINX-Demos/ansible/setup_etcd_demo.yml. The ansible playbook file also invokes another script provision.sh which sets the HOST_IP environment variable to the IP address of the eth1 interface (10.2.2.70 in this case assigned in the Vagrantfile) and invokes the ```docker-compose up -d``` command

1. SSH into the newly created virtual machine and move into the /vagrant directory which contains the demo files:

     ```
     $ vagrant ssh
     $ sudo su
     ```
The demo files will be in /srv/NGINX-Demos/etcd-demo

1. Now simply follow the steps listed under section 'Running the demo'.


### Ansible only deployment

1. Create Ubuntu 14.04 VM

1. Install Ansible on Ubuntu VM

     ```
     $ sudo apt-get install ansible
     ```

1. Clone demo repo into ```/srv``` on Ubuntu VM:

     ```
     $ cd /srv
     $ sudo git clone https://github.com/nginxinc/NGINX-Demos.git
     ```

1. Copy ```nginx-repo.key``` and ```nginx-repo.crt``` files for your account to ```/srv/NGINX-Demos/ansible/files/```

1. Move into the etcd-demo directory which contains the demo files and set HOST_IP on line 5 in script.sh to the IP of your Ubuntu VM on which NGINX Plus will be listening.
     ```
     $ cd /srv/NGINX-Demos/etcd-demo
     ```

1. Run the ansible playbook against localhost on Ubuntu VM:

     ```
     $ sudo ansible-playbook -i "localhost," -c local /srv/NGINX-Demos/ansible/setup_etcd_demo.yml
     ```

1. Now simply follow the steps listed under section 'Running the demo'.


### Manual Install

#### Prerequisites and Required Software

The following section assumes you are running on Mac OSX. The following software needs to be installed on your machine:

* [Docker Toolbox](https://www.docker.com/docker-toolbox) if running on OSX
* [docker-compose](https://docs.docker.com/compose/install). I used [Homebrew](http://brew.sh) to install it: `brew install docker-compose`
* [jq](https://stedolan.github.io/jq/), I used [brew](http://brew.sh) to install it: `brew install jq`
* [etcdctl](https://github.com/coreos/etcd/tree/master/etcdctl) is a command line client for etcd. Follow the steps under 'Getting etcdctl' section and and copy over etcdctl executable under /usr/local/bin

As the demo uses NGINX Plus a `nginx-repo.crt` and `nginx-repo.key` needs to be copied into the `nginxplus/` directory

#### Setting up the demo

1. Clone demo repo

     ```$ git clone https://github.com/nginxinc/NGINX-Demos.git```

1. Copy ```nginx-repo.key``` and ```nginx-repo.crt``` files for your account to ```~/NGINX-Demos/etcd-demo/nginxplus/```

1. Move into the demo directory:

     ```
     $ cd ~/NGINX-Demos/etcd-demo
     ```
1. If you have run this demo previously or have any docker containers running, start with a clean slate by running
    ```
    $ ./clean-containers.sh
    ```

1. NGINX Plus will be listening on port 80 on docker host, and you can get the IP address by running 
     ```
     $ docker-machine ip default
     192.168.99.100
     ```
     Export this IP into an environment variable HOST_IP `export HOST_IP=192.168.99.100` (used by script.sh & etcd_exec_watch.sh scripts)

1. Spin up the etcd, Registrator and NGINX Plus containers first: 

     ```
     $ docker-compose up -d
     ```

1. Execute the etcd_exec_watch.sh script in background (This invokes an etcdctl exec-watch command watching for changes in etcd keys and trigger script.sh whenever a change is detected).
    ```
     $ ./etcd_exec_watch.sh &
     ```

1. Spin up the two hello-world containers which will act as NGINX Plus upstreams
     ```
     $ docker-compose -f create-services.yml up -d
     ```

## Running the demo

1. You should have a bunch of containers up and running now:
    ```
    $ docker ps
    CONTAINER ID        IMAGE                           COMMAND                  CREATED             STATUS              PORTS                                                                NAMES
26544538c3ec        nginxdemos/hello:latest         "nginx -g 'daemon off"   2 hours ago         Up 2 hours          443/tcp, 0.0.0.0:8081->80/tcp                                        service1
27abd73b4621        nginxdemos/hello:latest         "nginx -g 'daemon off"   2 hours ago         Up About an hour    443/tcp, 0.0.0.0:8082->80/tcp                                        service2
9b976338829b        etcddemo_nginxplus              "nginx -g 'daemon off"   2 hours ago         Up 2 hours          0.0.0.0:80->80/tcp, 443/tcp, 0.0.0.0:8080->8080/tcp                  nginxplus
d9806ab06d05        gliderlabs/registrator:latest   "/bin/registrator etc"   2 hours ago         Up 2 hours                                                                               registrator
8f8f1c80c7e4        quay.io/coreos/etcd:v2.0.8      "/etcd -name etcd0 -a"   2 hours ago         Up 2 hours          0.0.0.0:2379-2380->2379-2380/tcp, 0.0.0.0:4001->4001/tcp, 7001/tcp   etcd
    ```

1. If you followed the Fully automated Vagrant/Ansible setup option above, HOST_IP referred below is the IP assigned to your Vagrant VM (i.e 10.2.2.70 in Vagrantfile) and is set already. And if you followed the Ansible only deployment option, HOST_IP will be the IP of your Ubuntu VM on which NGINX Plus is listening (IP of the interface set on line 6 of provision.sh, set to eth1 by default). For the manual install option, HOST_IP was already set above to `docker-machine ip default`

1. Go to `http://<HOST_IP>` in your favorite browser window and that will take you to one of the nginx-hello containers printing its hostname, IP Address and the port of the container. `http://<HOST_IP>:8080/` will bring up the NGINX Plus dashboard. The configuration file NGINX Plus is using here is /etc/nginx/conf.d/app.conf which is included from /etc/nginx/nginx.conf. If you would like to see all the services registered with etcd you could do a `curl http://$HOST_IP:4001/v2/keys | jq '.'`. **We are also using the persistent on-the-fly reconfiguration introduced in NGINX Plus R8 using the [state](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#state) directive. This means that NGINX Plus will save the upstream conf across reloads by writing it to a file on disk.**

1. Now spin up two more containers named service3 and service4 which use the same [nginxdemos/hello](https://hub.docker.com/r/nginxdemos/hello/) as above. Go to the Upstreams tab on Nginx Plus dashboard and observe the two new servers being added to the backend group.
     ```
     $ docker-compose -f add-services.yml up -d
     ```

1. Now stop any two services and observe that they get removed from the upstream group on Nginx Plus dashboard automatically
     ```
     $ docker stop service2 service4
     ```

1. Play by creating/removing/starting/stopping multiple containers. Creating a new container with SERVICE_TAG "production" or starting a container will add that container to the NGINX upstream group automatically. Removing or stopping a container removes it from the upstream group.

1. The way this works is everytime there is a change in etcd, script.sh gets invoked (through etcd_exec_watch.sh) which checks some of the environment variables set by etcd and adds the server specified by ETCD_WATCH_VALUE to the NGINX upstream block if ETCD_WATCH_ACTION is 'set' and removes it if ETCD_WATCH_ACTION is 'delete'. The removal happens by traversing through all NGINX Plus upstreams and removing the ones not present in etcd.

All the changes should be automatically reflected in the NGINX config and show up on the NGINX Plus Dashboard.
