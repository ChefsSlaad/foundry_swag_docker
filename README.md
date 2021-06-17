# foundry_swag_docker

version 0.04 - Animated Object

This is a how-to on running foundry-vtt in a docker container and securing the connection using nginx and letsencrypt. If that does not mean anything to you, this is basically a how to on running a reasonably secure version of foundry. It is: 
- **containered** - even if someone is able to hijack your foundry system through a vulnerability or by guessing your password, they cannot go any further. they're basically stuck in your container. It also has all sorts of portability and scalability advantages that do not really matter for your single home server.
- **encrypted** - the connection between your player's PC and your server is encrypted, which means that other users cannot easily steal your password or hijack your connection.
- **not idiot proof** - you may notice I use some carefull language. That is because, while I know this will make you more secure than doing nothing, there is no such thing as an unhackable system. So, please **choose a strong password** and **update your system**. Also, dont go arround daring other people to hack you. That's just stupid.


# Disclaimer
* This guide is written by me, based on my own experience self-hosting foundryVTT. There are bound to be mistakes in this guide. Please contact me if I missed anything or if you feel this guide could be improved. Or, you know, make a pull request. this is github after all
* Im assuming you have a passing familiarity with linux, the terminal and a rudimentary understanding of containers. I may forego or adjust this assumption in the future, but right now it is what it is. 
* I'm assuming you own a licence key for [foundry-vtt](https://foundryvtt.com/)
* I'm assuming you have a static IPv4 adress. find out what your IP is easily using www.whatsmyip.org

# Overview
This guide is set up in 5 steps:
* Preparations
* Setting up the host (the system or server you will be using to run foundry vtt)
* Setting up the foundry-vtt container
* Setting up the SWAG container
* Wrapping up


# preparations
The preparations are about making sure you have everything ready to install and run foundry. Besides the licence key and the static ip (which I assume you allready have), you will also need a domain name, as that is what the letsencrypt ssl certificate is attached to.

## Getting a domain name
You basically have two choices: a fancy pantsy second level domain like AgeOfWorms.com or TheCityOfSharn.org (or whatever you campaign or setting or whatever is called), or a free third level domain like mycampaign.duckdns.org. The .org, .com and .net domains are paid and usually start at $8,- per year, while duckdns.org is completely free. If you want a second level domain name, shop arround a bit. prices vary and sometimes you can get the domain at a discount. 
the steps are :
### 2nd level dns (like godaddy.com or domains.com) 
* register the domain name
* add (or replace) your IP adrress to the dns A records. The details depend on your provider, but the result should be something like this:

      name       type     content           ttl
      www        A        100.100.100.1     24H
      @          A        100.100.100.1     24H


What this does is tell your provider that you would like to forward trafic for www.yourdomain.com and yourdomain.com to the ip address 100.100.100.1. The provider will the tell other DNS servers that whenever someone asks for the adresses you specified, they should forward likewise. Those providers tell other providers, etc. etc. This is called propagation. this is quite fast, but the internet is a big place, so it still takes a couple of hours before all DNS servers are aware of your new domain name. 


### duckdns(the cheap and easy option)
* create an account with duckdns.org
* enter the domainname you want to register
* duckdns will look up your public IP and fill that in. If it's incorrect for whatever reason, fill in the correct ip adress

### Test 
test that the domain resolves to the correct IP
  * go to www.mxtoolbox.com/DNSLookup.aspx 
  * enter your domain name
  * verify that it resolves to your IP address. If it doesnt, wait a bit and try again. 

## hardware selection
Here's a secret. you dont need powerfull hardware to host a foundry-vtt server. a raspberry pi (4B) will do, as will a NAS or NUC. Or, if you dont mind, use an old desktop or laptop as a server. 
what you need is: 
* one or two core CPU's (anythng over 1Ghz will do)
* 1GB ram
* at least 1GB disk space
The biggest bottleneck for a smooth running game is for your server to serve all your assets quickly. Make sure you are using an SSD (so upgrade that if you are using old hardware). If you are going with a raspberry pi, make sure you it's a Pi 4B as you will want usb 3.0 and gigabit ethernet.


# setting up the host
This is about configuring your host machine (which we discussed in the hardware selection section just now) to run your server. If this is not a NAS, I recommend doing a fresh install of your OS of choice. I assume this will be some sort of BSD or linux system; such as raspbian, debian, or perhaps proxmox or similar. I wont go into the details of doing this. The rest of this tutorial assumes you have debian installed (mostly because that's what I'm running).

make sure your host has a static IP adress

make a note of the ip address of you host. on debian type:

    ip addr

you will also need a way to access your host and the terminal. I assume you are familiar with ssh or [PuTTY](https://www.putty.org/), so I won't go into it here.  

anyway, before we start, lets make sure everything is up to date:

    sudo apt update
    sudo apt upgrade


## Install docker
Docker has a really good install guide for multiple systems. 

[guide for debian](https://docs.docker.com/engine/install/debian/)

There are also some recommende [post-install steps.](https://docs.docker.com/engine/install/linux-postinstall/) I personally configure docker to be managed by  as a non-root user, and I configure docker to start on boot.  

## (optional) Install portainer
Portainer is a container management system. It basically adds a web interface to docker and gives you some handy tools. You can absolutely do without. It just makes life that little bit easier. 

As portainer itself runs in docker, deploying it is as simple as running two commands

     docker volume create portainer_data

This creates a persistent place to store some of the container's data. Ususally containers will lose all data when you restart the container. This is a feature that makes containers more predictable and more secure. But sometimes you need certain data, such as config files to remain after you have restarted a container. That is where volumes come in. basically you are telling docker to reserve a place called portainer_data where this data can be stored.   
     
     docker run -d --name=Portainer --hostname=Portainer -p 8000:8000 -p 9000:9000 --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data -e TZ='Europe/Amsterdam' portainer/portainer
     
This tells docker to start portainer. The variables are:

    docker run               --> tell docker to run a container
    -d                       --> run in daemon or detached mode. basically run in the background
    --name=Portainer         --> the name that docker uses to identify this container
    --hostname=Portainer     --> the name other computers use to identify this portainer on the network
    -p 8000:8000             --> map port 8000 on your host to the same port in the container. Port 8000 is used mostly for managing other portainer instances, so I'm not sure if you need this. 
    -p 9000:9000             --> map port 9000 on your host to the same port in the container. This means that users that visit http://<yourip>:9000 will be served the portainer web interface. 
    --restart=allways        --> allways restart (recover) the container after a crash.
    -v var/run/.....         --> this maps (shares) what is going on with docker on your host to the container. The container needs this to monitor and manage other containers on your network
    -v portainer_data:/..    --> this maps (shares) the persistent volume you created to your container, so that your configurations remain persistent between restarts
    -e TZ='Europe/Amsterdam' --> set the timezone to where you live. You can change it to where you live. If you remove this part entirely, the container will default to UTC
    portainer/portainer      --> the name of the base image. Docker will look up this container on your host system, or download it from the docker repository if it is not present. 

Test if portainer is working by visiting http://yourip:9000

You should see a registration screen. register and press +create user
  
next, chose the install type: LOCAL
  
you should see a dashboard
--> click on local
--> click on containers
--> you should see 1 container active; you can inspect it using portainer, restart it, stop it or kill it (dont do those last two!).. oh, and maybe next time I should put the warning before the command that will destroy your pretty web interface...


## Setting up a data folder for your resources
It's a good idea to create a folder where you will store all your art assets in one easily searchable place. As your library will probaby grow (god knows it never shrinks), its a good idea to put some thought into the organsiation now, as its a pain to change it later. You can put this is your home directory (/home/user/resources) or anyplace else that makes sense for you. 
  
I personally use a structure:
  
- resources
  - assets
  - maps
  - tokens
  - campaign specific stuff. 
    
For the rest of the guide, I am assuming you hae a folder called resources that contains all this stuff. 

# setting up foundry-vtt
There is currently no official foundry-vtt container, but there are plenty of options created by fans. Which one is the best is going to vary over time. Have a look on [foundryvtt.com](https://foundryvtt.wiki/en/setup/hosting/Docker) for some o the more popular options. Two of my favorites:
- https://hub.docker.com/r/felddy/foundryvtt is quite popular and seems easy to set-up and configure. If you go this route, mae sure you use secrets.json to store your  password and key 
- https://github.com/BenjaminPrice/fvtt-docker (aka direckthit) strikes a  happy medium (for me) between convenience and security. basically, you download the zipfile yourself and a script in the container does the rest. No credentials to store, no credentials to accidentally leak.
      
for this guide, I'm using direckthit. 

## deploying the container
create a directory to store your game data. where you do this depends greatly on your system configuration. If you are using a raspberry pi, the best place may the your home directory. 
            
      mkdir -p ~/foundryvtt
      cd ~/foundryvtt

now is also a good time to download the foundryvtt-0.x.x.zip file and copy it into this directory
      
edit the configuration file
      
      nano docker-compose.yaml
      
If you are using portainer, instead of creating a docker-compose file, you can click on stacks (in the left menu) --> add stack and then copy the following text into the web editor. Give the stack a clear name, such as foundryvtt.
      , 
If you are not using portainer, copy the following into the empty docker-compose.yaml file.
      
      version: '3'
      services:
        foundryvtt:
          image: direckthit/fvtt-docker:latest
          container_name: foundryvtt
          ports: 
            - 30000:30000
          volumes:
            - /path/to/your/foundry/data/directory:/data/foundryvtt
            - /path/to/your/resources:/data/foundryvtt/Data/resources
            - /path/to/your/foundry/zip/file:/host
          restart: unless-stopped


in volumes, replace the following paths:
* path/to/your/foundry/data/directory --> the directory you created for your persistent game data (probably /home/user/foundryvtt)
* path/to/your/resources --> the directory you created for your resources. (probably /home/user/resources)
* /path/to/your/foundry/zip/file --> the place where you stored foundryvtt-0.x.x.zip (probably /home/user/foundryvtt)

save the docker-compose.yaml file.
   
If you are using portainer, click on 'deploy the stack' to finish the install. You should see the stack with the associated container up and running
      
if you are not using portainer, run:
      
      docker-compose up -d

this will configure the container and run in detached or daemon mode.      
      
## testing foundry
visit http://hostip:30000

you should see a screen that asks for your registration key. If you are, you're done!
      
## Updating 
updating foundry is done by stopping, removing and redeploying the stack. Before you do this, **shut down your game world.** you may want to create a backup as well. 
      
In portainer updating is done by copying the text from the web-editor, deleting the old stack and deploying a new stack. portainer v2.6 should allow you to do do this without a janky copy-paste, but it's not in the current version.

without portainer, run     
      
      docker-compose rm --stop
      docker-compose up -d      

again **close your world** and **back up your data**
      
# setting up SWAG
Linuxserver.io has made an excellent set of containers. I personally have a bunch of them running on my home server. One of the best ones is [SWAG](https://docs.linuxserver.io/general/swag), a container that combines Letsencrypt, nginx, a reverse proxy and fail2ban. Trust me, it's cool.   

what is does, handle your incoming connections and directs them to the correct server, while keeping the bad stuff out. 

## Configure your router
you need to forward http and https trafic to your host. You do this by configuring your router to forward port 80 and port 443 trafic from WAN (the internet) to your host IP. Unfortunately, different routers do this is in different ways.  [This guide](https://www.noip.com/support/knowledgebase/general-port-forwarding-guide/) has some help for different brands of routers. 

## deploy the SWAG container
The swag docker-compose file is a bit more involved, but its fairly straigh forward when you get the hang of it. I recommend looking at the [doccumentation](https://docs.linuxserver.io/general/swag)for all the parameters

first create a folder where you can store the persistent data

mkdir ~/swag

the config file is different depending on the type of domain you have. I have added the files for both duckdns and an regular 2nd level domain. I believe e.g. cloudflare has some options that will make your config more easy, but i have no experience there. 

### duckdns docker-compose.yaml:


     version: "2.1"
    services:
      swag:
        image: ghcr.io/linuxserver/swag
        container_name: swag
        cap_add:
          - NET_ADMIN
        environment:
          - PUID=1000
          - PGID=1000
          - TZ=Europe/Amsterdam 
          - URL=sudbdomain.duckdns.org #use your own url
          - SUBDOMAINS=wildcard
          - VALIDATION=duckdns
          - DUCKDNSTOKEN=yourtoken #use the token on your duckdns page
          - EMAIL=you@yourname #optional  
        volumes:
          - /path/to/config:/config
        ports:
          - 443:443
          - 80:80
        restart: unless-stopped

### 2nd level domain docker-compose.yaml:

    version: "2.1"
    services:
      swag:
        image: linuxserver/swag
        container_name: swag
        cap_add:
          - NET_ADMIN
        environment: 
          - PUID=1000
          - PGID=1000
          - TZ=Europe/Amsterdam
          - URL=yoururl.com  #insert your domain name 
          - SUBDOMAINS=www,
          - VALIDATION=http
        volumes:
           - /path/to/config:/config
        ports:
          - 443:443
          - 80:80
        restart: unless-stopped

in both cases you need to configure 2 things:
* url=yoururl.com --> add your url here. this can be either a duckdns url or a 2nd level domain. do not add www to the url. that will be covered in the subdomains section
* /path/to/config --> replace this with your config file. probably /home/user/swag

if you are using portainer, go to stacks --> add stack and copy the config into the web editor.

if you are not using portainer, create a file called docker-compose.yaml and paste the content into that file

    cd ~/swag
    nano docker-compose.yaml
    
now lets run the container again    
    
    docker-compose up -d

## configure reverse proxy

look into the swag config files.

cd  ~/swag/config/nginx/site-confs/

you should add an entry for foundry:

nano foundryvtt

add the following content to the file:

    upstream foundryvtt {
        server 10.0.0.10:30000;    # the server and port inside the network
        }

    # only serve https
    map $http_upgrade $connection_upgrade {
            default upgrade;
            '' close;
        }

    server {
            listen 443 ssl http2;
            server_name your_domain.com www.yourdomain.com;  #add your domain name here. if you want to use both with and without ww, add both here.

            include /config/nginx/ssl.conf;
 
            client_max_body_size 0;
            add_header Strict-Transport-Security "max-age=31536000; includeSubdomains";
            ssl_session_cache shared:SSL:10m;
            proxy_buffering off;
 
            location / {
                proxy_pass http://foundryvtt;    # Matches to the "upstream" name above
                proxy_set_header Host $host;
                proxy_redirect http:// https://;
                proxy_http_version 1.1;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  # for forcing password validation from outside
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection $connection_upgrade;

        }
    }

save and close.

restart swag

    docker-compose restart
    
or in portainer go to the stack, select the container and the restart


verify everything works by going to www.yourdomain.com. you should see the foundry login screen.



# Wrapping up
* backups
* ...
