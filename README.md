demo-joc
=========

This repo is for the $JENKINS_HOME of a complex Jenkins Operations Center Docker demo w/ High Availability, multiple client masters, and shared slaves.



Usage:
-----


To mount a shared $JENKINS_HOME for HA:

    docker run --name storage apemberton/jenkins-storage git clone https://github.com/harniman/demo-joc.git .


This container is based on jenkins-base and has:
- git
- a volume called /data which is exposed
- checks out the contents from GitHub

NOTE: This needs to be changed as we are not using git to backup JENKINS_HOME as the files exceed git limits, and, checking in binary files is not a good idea. See demo-mgmt for an rsync script that will backup the home area


To enable container discovery for the load balancer:
    
    docker run -d -p 172.17.42.1:53:53/udp --name skydns crosbymichael/skydns --nameserver 8.8.8.8:53 --domain beedemo.io
    docker run -d -v /var/run/docker.sock:/docker.sock --name skydock crosbymichael/skydock -ttl 30 -environment dev -s /docker.sock -domain beedemo.io -name skydns
    
Once you run the SkyDock and SkyDNS containers, they will automatically start assigning intuitive urls that services can use to communicate to each other. The URLs take the Syntax of <CONTAINER_NAME>.<IMAGE_NAME>.<ENVIRONMENT>.<DOMAIN>


To run JOC in an HA setup with mounted storage for its $JENKINS_HOME:

    docker run -d --dns=172.17.42.1 --name joc-1 --volumes-from storage -e JENKINS_HOME=/data/var/lib/jenkins/joc harniman/jenkins-operations-center --prefix="/operations-center"
    docker run -d --dns=172.17.42.1 --name joc-2 --volumes-from storage -e JENKINS_HOME=/data/var/lib/jenkins/joc harniman/jenkins-operations-center --prefix="/operations-center"

To run JE in an HA setup with mounted storage for its $JENKINS_HOME:

    docker run -d --dns=172.17.42.1 --name api-team-1 --volumes-from storage -e JENKINS_HOME=/data/var/lib/jenkins/api-team harniman/jenkins-enterprise --prefix="/api-team"
    docker run -d --dns=172.17.42.1 --name api-team-2 --volumes-from storage -e JENKINS_HOME=/data/var/lib/jenkins/api-team harniman/jenkins-enterprise --prefix="/api-team"

    docker run -d --dns=172.17.42.1 --name web-team-1 --volumes-from storage -e JENKINS_HOME=/data/var/lib/jenkins/web-team harniman/jenkins-enterprise --prefix="/web-team"
    docker run -d --dns=172.17.42.1 --name web-team-2 --volumes-from storage -e JENKINS_HOME=/data/var/lib/jenkins/web-team harniman/jenkins-enterprise --prefix="/web-team"

To create shared slaves:

    docker run -d --dns=172.17.42.1 --name slave-1 harniman/jenkins-slave
    docker run -d --dns=172.17.42.1 --name slave-2 harniman/jenkins-slave
    docker run -d --dns=172.17.42.1 --name slave-3 harniman/jenkins-slave

To provide a management host 

	docker run -i -t  --dns=172.17.42.1 --name mgmt --volumes-from storage  harniman/demo-mgmt
	

To allow a load balancer to run in front of these containers:

    docker run -i -t --dns=172.17.42.1 -p 80:80 -p 40012:40012 -p 40013:40013 --name proxy harniman/demo-joc-haproxy

To access the applications, you'll also need to edit your hosts file. On a Mac, this can be found under /private/etc/hosts. Add the following lines to this file:

    192.168.59.103  proxy.demo-joc-haproxy.dev.beedemo.io


Where 192.168.59.103 is the ip address for boot2docker.

The jenkins service are exposed on:

	http://proxy.demo-joc-haproxy.dev.beedemo.io/operations-center/
	http://proxy.demo-joc-haproxy.dev.beedemo.io/api-team/
	http://proxy.demo-joc-haproxy.dev.beedemo.io/web-team/
	
	
Configuring Jenkins Nodes:
--------------------------

All nodes need their JENKINS URL set to the correct values - see above

Enable Security and set fixed JNLP ports - these are statically mapped in HA proxy until we get Andy's dynamic script working:

	40001 : JOC
	40002 : API-TEAM
	40003 : WEB-TEAM



Optional:
---------

To re-start the management host 

	docker start mgmt
	docker attach mgmt

To create the repo for the persistance data - connect to the mgmt host

	cd /data
	git config --global user.email "nharniman@cloudbees.com"
	git config --global user.name "Nigel Harniman" 

	git remote set-url origin https://github.com/harniman/demo-joc.git
	git add -A
	git commit -m "committing my demo"
	git push

To update the persistenance data

	cd /data
	git add -A
	git commit -m "committing my demo"
	git push




If running a custom setup, editing the HAproxy.cfg:

    docker attach proxy
    sudo vi /etc/haproxy/haproxy.cfg
    service haproxy restart

Here is my working example


	####stock haproxy settings####
	global
	            chroot      /var/lib/haproxy
	            pidfile     /var/run/haproxy.pid
	            maxconn     4000
	            user        haproxy
	            group       haproxy
	            daemon
	            stats socket /var/lib/haproxy/stats
	            log 127.0.0.1 local0
	#            log /dev/log local0 info
	#            log /dev/log local0 notice
	#            debug
	
	defaults
	            log                     global
	            option                  dontlognull
	            option                  dontlog-normal
	            option                  http-server-close
	            option                  forwardfor except 127.0.0.0/8
	            option                  redispatch
	            option                  abortonclose
	            retries                 3
	            maxconn                 3000
	            timeout http-request    10s
	            timeout queue           1m
	            timeout connect         10s
	            timeout client          1m
	            timeout server          1m
	            timeout http-keep-alive 10s
	            timeout check           500
	            default-server          inter 5s downinter 500 rise 1 fall 1
	
	
	frontend http_proxy
	  bind *:80
	  mode http
	  acl host hdr_dom(host) -i beedemo.io
	  acl api-team url_beg /api-team
	  acl web-team url_beg /web-team
	  acl joc url_beg /operations-center
	
	
	  use_backend api-team if api-team
	  use_backend web-team if web-team
	  use_backend joc if joc
	  default_backend joc
	
	
	backend joc
	  balance roundrobin
	  option  httpchk /operations-center/ha/health-check
	  mode http
	  server joc1-server  joc-1.jenkins-operations-center.dev.beedemo.io:8080 check
	  server joc2-server  joc-2.jenkins-operations-center.dev.beedemo.io:8080 check
	
	backend api-team
	 balance roundrobin
	 option  httpchk /api-team/ha/health-check
	 mode http
	 server api-team-1-server api-team-1.jenkins-enterprise.dev.beedemo.io:8080 check
	 server api-team-2-server api-team-2.jenkins-enterprise.dev.beedemo.io:8080 check
	
	backend web-team
	 balance roundrobin
	 mode http
	 option  httpchk /web-team/ha/health-check
	 server web-team-1-server web-team-1.jenkins-enterprise.dev.beedemo.io:8080 check
	 server web-team-2-server web-team-2.jenkins-enterprise.dev.beedemo.io:8080 check
	
	
	#Define the static Slave ports for JOC
	frontend fe_joc-jnlp
	        bind *:40001
	        mode tcp
	        log global
	        option tcplog
	        timeout client 3600s
	        backlog 4096
	        maxconn 50000
	        default_backend be_joc-jnlp
	
	backend be_joc-jnlp
	        mode  tcp
	        option log-health-checks
	        option redispatch
	        option tcplog
	        balance roundrobin
	        server joc1server joc-1.jenkins-operations-center.dev.beedemo.io:40001 check
	        server joc2server joc-2.jenkins-operations-center.dev.beedemo.io:40001 check
	        timeout connect 1s
	        timeout queue 5s
	        timeout server 3600s
	
	#Define the static Slave ports for api-team
	frontend fe_api-team-jnlp
	        bind *:40002
	        mode tcp
	        log global
	        option tcplog
	        timeout client 3600s
	        backlog 4096
	        maxconn 50000
	        default_backend be_api-team-jnlp
	
	backend be_api-team-jnlp
	        mode  tcp
	        option log-health-checks
	        option redispatch
	        option tcplog
	        balance roundrobin
	        server api-team-1 api-team-1.jenkins-enterprise.dev.beedemo.io:40002 check
	        server api-team-2 api-team-2.jenkins-enterprise.dev.beedemo.io:40002 check
	        timeout connect 1s
	        timeout queue 5s
	        timeout server 3600s
	
	#Define the static Slave ports for web-team
	frontend fe_web-team-jnlp
	        bind *:40003
	        mode tcp
	        log global
	        option tcplog
	        timeout client 3600s
	        backlog 4096
	        maxconn 50000
	        default_backend be_web-team-jnlp
	
	backend be_web-team-jnlp
	        mode  tcp
	        option log-health-checks
	        option redispatch
	        option tcplog
	        balance roundrobin
	        server web-team-1 web-team-1.jenkins-enterprise.dev.beedemo.io:40003 check
	        server web-team-2 web-team-2.jenkins-enterprise.dev.beedemo.io:40003 check
	        timeout connect 1s
	        timeout queue 5s
	        timeout server 3600s



By default, the Wildfly and Nexus backends are commented out.

If running a forked Docker image, be sure to edit any associated SkyDock urls to refer to your Docker hub id and the new image name. The syntax for a SkyDock URL is:

    <CONTAINER_NAME>.<IMAGE_NAME>.<ENVIRONMENT>.<DOMAIN>

This demo's environment name is "dev" and the domain is "beedemo.io".


Tricks for working with Docker/boot2docker
-----------------------------------------

1. Using NSenter to ssh to a running container:

docker-enter + NSEnter: https://github.com/jpetazzo/nsenter

Do this by first entering your boot2docker VM:

    boot2docker ssh

Then install NSenter:

    docker run --rm -v /usr/local/bin:/target jpetazzo/nsenter
    
Finally, use NSEnter to enter your running container:

    docker-enter "$CONTAINER_NAME"


2. If you're using a Mac, you've restarted boot2docker and the ip address has changed, go to /etc/hosts and edit ip address to the new b2d address.

Then run 

    sudo dscacheutil -flushcache

3. To mass stop all Docker containers:

    docker stop $(docker ps -a -q)

To mass remove all stopped Docker containers:

    docker rm $(docker ps -a -q)

4. If you restart a conatiner, you'll also have to restart the HAproxy service so that HAproxy can get the new IP address for the container from SkyDock.

5. If boot2docker is left running for more than a few days, it will need to be restarted because the VirtualBox VM's clock will be out of sync with the actual local time. This can cause the license for Jenkins Enterprise/JOC to appear to have been issued in the future, which will make the license invalid. No new licenses issued during this mis-sync will work either, as they'll be dated for the correct time but out of sync with the server/VM's internal clock.

6. To exit a running container WITHOUT killing it, use Ctrl+P followed by Ctrl+Q, rather than Ctrl+C. 

7. If error messages relating to disk space in the containers start to appear, it may be necessary to increase the space available to boot2docker vm. See https://docs.docker.com/articles/b2d_volume_resize/


Docker Hub
----------

To view the Dockerfile and repository for the Jenkins Operations Center image:
    
    https://registry.hub.docker.com/u/lavaliere/jenkins-operations-center/

To view the Dockerfile and repository for the Jenkins Enterprise image:

    https://registry.hub.docker.com/u/lavaliere/jenkins-enterprise/


Roadbumps and ToDo
-----------------

1) haproxy resolves dns entries and caches the IP addresses on startup. There is no guarantee that when a docker container restarts it will receive the same IP address. Ideally we need to force a scheduled reoad of haproxy periodically



RBAC Configuration
------------------

The following users are set up - passwords match the userid

	User - Role

	addnie.admin - Overall admin - has access to everything
	adam.api - Team leader for the API team. Can configure new jobs for API Master
	wally.web - Team leader for the Web team. Can configure new jobs for API Master
	dev.api - API team developer
	dev.web - Web team developer
	
The API / Web users are mapped to be able to create/run jobs only on their respective masters

All authenticated users can see all jobs.



	
