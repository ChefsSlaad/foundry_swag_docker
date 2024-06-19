# Foundry Swag Docker


![foundry_swag_docker](foundry-docker-swag.png)

version 0.08 - Archon

This is a how-to on running [foundry-vtt](https://foundryvtt.com/) on your home server 24/7 in a docker container and securing the connection using nginx and letsencrypt. If that does not mean anything to you, this is basically a how-to on running a reasonably secure version of foundry. It is: 
- **containered** - even if someone is able to hijack your foundry system through a vulnerability or by guessing your password, they cannot go any further and damage your system or network. They're basically stuck in your container. We will use [Docker](https://www.docker.com/resources/what-container) to achieve this.
- **encrypted** - the connection between your player's PC and your server is encrypted, which means that other users cannot easily steal your password or hijack your connection. We will use [Secure Web Application Gateway aka SWAG](https://hub.docker.com/r/linuxserver/swag) to achieve this.

# Disclaimer
* This guide will make you more secure hosting foundry than doing nothing. However, there is no such thing as an unhackable system. So please **choose a strong password** and **update your system**.  Also, don't go around daring other people to hack you. That's just stupid.
* This guide is geared towards users who want to host foundry 24/7. That assumes you have a home server or some other dedicated hardware (even a raspberry pi) that player and the GM can always access from the internet. If you are only running foundry when you have a game, or if you only need your players to access the game from your LAN, this may be overkill. Then again, you're not paranoid if they're *really* out to get you.  
* This guide is written by me, based on my own experience self-hosting foundryVTT. There are bound to be mistakes in this guide. Please contact me if I missed anything or if you feel this guide could be improved. Or, you know, make a pull request. This is Github after all.
* I'm assuming you have a passing familiarity with Linux, the terminal, and how a web server works in general. I may forego or adjust these assumptions in the future, but right now it is what it is. 

# Overview
This guide is set up in 4 steps:
* [Preparations](#preparations)
* [Setting up the host](#setting-up-the-host) (the system or server you will be using to run foundry vtt)
* [Setting up the containers](#containers)
* [Wrapping up](#wrapping-up)


# Preparations
The preparations are about making sure you have everything ready to install and run foundry.

## Hardware selection
Here's a secret: [You dont need powerfull hardware to host a foundry-vtt server](https://foundryvtt.com/article/requirements/). A raspberry pi (4B) will do, as will a NAS or NUC. Or, if you dont mind, use an old desktop or laptop as a server. 

What you need is: 
* one or two core CPU's (anythng over 1Ghz will do)
* 1GB ram
* at least 1GB disk space

In my experience, the biggest bottleneck for a smooth running game is for your server to serve all your assets quickly. For best results:
* Make sure you are using an SSD (SD Cards or other Flash memory will be quickly destroyed by foundry's frequent read/writes)
* Have a good, wired connection
* If you are going with a raspberry pi, make sure it's a Pi 4B as you will want usb 3.0 and gigabit ethernet

## Getting Foundry-VTT
Before you get started you'll need to have [bought foundry-vtt](https://foundryvtt.com/purchase/) and have received your license.

## Getting a domain name
You basically have two choices: a fancy pancy full domain like `AgeOfWorms.com` or `TheCityOfSharn.org` (or whatever you campaign or setting is called), or a free subdomain like `mycampaign.duckdns.org`. The `.org`, `.com` and `.net` domains are paid and usually start at $8 per year, while duckdns.org is completely free. If you want a full domain name, shop arround a bit. Prices vary and sometimes you can get the domain at a discount. 

The rest of this guide breaks down into two paths based on whether you have bought a full domain name or are using duckdns.

### Option 1: full domain name (like godaddy.com or domains.com) 
* register the domain name
* add (or replace) your IP adrress to the dns A record. The details depend on your provider, but the result should be something like this:

      name       type     content             ttl
      www        A        your.ext.ip.adr     24H
      @          A        your.ext.ip.adr     24H


What this does is tell your provider that you would like to forward traffic for www.yourdomain.com and yourdomain.com to the ip address your.ext.ip.adr (e.g 66.102.13.99). The provider will tell other DNS servers that whenever someone asks for the addresses you specified, they should forward likewise. Those providers tell other providers, etc. etc. This is called propagation. This is quite fast, but the internet is a big place, so it still takes a couple of hours before all DNS servers are aware of your new domain name. 

### Option 2: duckdns (the free and easy option)
* log in to [duckdns.org](https://www.duckdns.org/) however you want
* enter the domain name you want to register
* duckdns will look up your public IP and fill that in.

You do **not** need to [set up a cron job to keep your IP address updated](https://www.duckdns.org/install.jsp), we use a docker container which will do this for us. This prevents us from having to make our router's public IP address public.

### Test 
Test that the domain resolves to the correct IP
  * go to www.mxtoolbox.com/DNSLookup.aspx 
  * enter your domain name
  * verify that it resolves to your IP address.  

If it doesn't, wait a bit and try again.
If it still doesn't, [verify your public IP was set correctly](https://whatismyipaddress.com/)

# Setting up the host
This is about configuring your host machine (which we discussed in the hardware selection section previously) as a server. If this is not a [NAS](https://en.wikipedia.org/wiki/Network-attached_storage), I recommend doing a fresh install of your OS of choice. I assume this will be some sort of BSD or Linux system; such as Raspbian, Debian, or perhaps Proxmox or similar. I won't go into the details of doing this. The rest of this tutorial assumes you have Debian installed (mostly because that's what I'm running).

Note: only install operating systems compatible with your storage device. I.e., don't install Ubuntu onto a SD Card or you will burn the SD out quickly.


## Update your server
For the best security you should always keep your system as up to date as possible. Updates are often security updates which patch known vulnerabilities. 

    sudo apt update
    sudo apt upgrade


## Install Docker
Docker has a really good install guide for multiple systems. 

[Guide for Debian](https://docs.docker.com/engine/install/debian/)

There are also some recommended [post-install steps.](https://docs.docker.com/engine/install/linux-postinstall/) 
For added security, I configure docker to be managed as a non-root user. For ease of use, I configure docker to start on boot.  


## Setting up a data folder for your resources
It's a good idea to create a folder where you will store all your art assets in one easily searchable place. Your library will probably grow (god knows it never shrinks) so it's a good idea to put some thought into the organization now, as it's a pain to change it later. You can put this is your home directory (/home/user/resources) or any place else that makes sense for you. 
  
I personally use a structure:
  
- resources
  - assets
  - maps
  - tokens
  - campaign specific stuff. 
    
For the rest of the guide, I am assuming you have a folder called resources that contains all this stuff. 

## Set up a static IP address for the host machine
You need to make your router always assign the same internal network IP address to the host machine. The steps to do this will depend on your router, but are generalised [in this guide](https://au.pcmag.com/networking/65062/how-to-set-up-a-static-ip-address). Having a static IP for the host machine ensures that you don't have to update port forwarding rules whenever it changes.

Please note this is different from making your router's public IP static.

Make a note of the ip address of your host. If you are using Debian run:

    ip addr
      
## Set up Port Forwarding
You need to configure your router to port forward port 80 (http) port 443 (https) to your host. Unfortunately, different routers do this is in different ways. [This guide](https://www.noip.com/support/knowledgebase/general-port-forwarding-guide/) has some help for different brands of routers.


# Containers
This section is going to be about selecting and configuring your containers. We are going to combine foundry and Swag into a single stack, with docker taking care of most of the plumbing (such as the networking between the containers). 

## Preparing Foundry
There is currently no official foundry-vtt container, but there are plenty of options created by fans. Which one is the best is going to vary over time. Have a look on [foundryvtt.com](https://foundryvtt.wiki/en/setup/hosting/Docker) for some of the more popular options. Two of my favorites:
- https://hub.docker.com/r/felddy/foundryvtt is quite popular and seems easy to set-up and configure. If you go this route, make sure you use `secrets.json` to store your password and key 
- https://github.com/BenjaminPrice/fvtt-docker (aka direckthit) strikes a happy medium (for me) between convenience and security. Basically, you download the zipfile yourself and a script in the container does the rest. No credentials to store, no credentials to accidentally leak.
      
For this guide, I'm using felddy. 


## SWAG
Linuxserver.io has made an excellent set of containers. I personally have a bunch of them running on my home server. One of the best ones is [SWAG](https://docs.linuxserver.io/general/swag), a container that combines Letsencrypt, nginx, a reverse proxy and fail2ban. Trust me, it's cool.   

What is does, handle your incoming connections and directs them to the correct server, while keeping the bad stuff out.

### Setting up the folders for your containers
Create a folder to store the container data. Where it should go depends on your system configuration. If you are using a raspberry pi, the best place may be your home directory. 
            
      mkdir -p ~/swag-foundry/foundry
      mkdir -p ~/swag-foundry/swag
      cd ~/swag-foundry
      

## Get a timed url
go to https://foundryvtt.com/community/me/licenses, select Linux/NodeJS under Operating system and click **timed url** 
a temporary url file will be copied to your clipboard.

*note* 
the next steps may take more time than you have for your timed url. Dont worry about it. You can request a new timed url as often as you like. Just keep the tab open and be ready to copy it again when needed. 


## Deploy the stack
We are going to create a configuration file for docker that tells it how to run our swag/foundry stack:

    nano docker-compose.yaml

copy the following into the file: 

### Option 1: if you are using a full domain name
```yml
version: "3.8"

services:
  foundry:
    container_name: foundry
    hostname: foundry
    image: felddy/foundryvtt:release
    environment:
      - FOUNDRY_RELEASE_URL=<timed_url>                          # replace with the timed url you copied earlier
    expose:
      - 30000
    volumes:
      - path/to/your/foundry/data/directory:/data/foundryvtt     # replace with the correct path to your foundry config dir
      - /path/to/your/resources:/data/foundryvtt/Data/resources  # replace with the correct path to your resources
    restart: unless-stopped
  

  swag:
    image: ghcr.io/linuxserver/swag
    container_name: swag
    cap_add:
      - NET_ADMIN
    environment:
      - PUID=1000
      - PGID=1000
    - TZ=Europe/Amsterdam                                       # replace with your timezone https://en.wikipedia.org/wiki/List_of_tz_database_time_zones
      - URL=yourdomain.com                                      # replace with your domain name
      - SUBDOMAINS=www 
      - VALIDATION=http
      - EMAIL=address@example.com                               # replace with your email address here (optional)
    volumes:
      - ./swag-config:/config                       # replace with the correct path to your swag config dir
    ports:
      - 80:80
      - 443:443
    restart: unless-stopped
```

### Option 2: if you are using using duckdns
```yml
version: "3.8"

services:
  duckdns:
    image: ghcr.io/linuxserver/duckdns
    container_name: duckdns
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Amsterdam                                      # replace with your local timezone https://en.wikipedia.org/wiki/List_of_tz_database_time_zones
      - SUBDOMAINS=mysubdomain                                   # replace with to your DuckDNS subdomain name.
      - TOKEN=blah-blah-blah                                     # replace with your DuckDNS token here
    restart: unless-stopped

  foundry:
    container_name: foundry
    hostname: foundry
    image: felddy/foundryvtt:release
    environment:
      - FOUNDRY_RELEASE_URL=<timed_url>                          # replace with the timed url you copied earlier
    expose:
      - 30000
    volumes:
      - path/to/your/foundry/data/directory:/data/foundryvtt     # replace with the correct path to your foundry config dir
      - /path/to/your/resources:/data/foundryvtt/Data/resources  # replace with the correct path to your resources
      - /path/to/your/foundry/zip/file:/host                     # replace with the correct path to your foundry-x.x.x.zip file
    restart: unless-stopped

  swag:
    image: ghcr.io/linuxserver/swag
    container_name: swag
    cap_add:
      - NET_ADMIN
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Amsterdam                                     # replace with your timezone
      - URL=subdomain.duckdns.org                               # replace with this to your DuckDNS hostname
      - VALIDATION=http
      - DUCKDNSTOKEN=blah-blah-blah                             # replace with your DuckDNS token here
      - EMAIL=address@example.com                               # replace with your email address (optional).
    volumes:
      - ./swag:/config
    ports:
      - 80:80
      - 443:443
    restart: unless-stopped
```

Modify the config on each line where there is a `# replace with` comment.
this is also where you need the **timed_url** you copied to your clipboard. 
Save the docker-compose.yaml file.
   
run:
      
      docker-compose up -d

This will configure the container and run in detached or daemon mode.      
      
## Configure reverse proxy

Look into the swag config files.

      cd  ~/swag-foundry/swag/config/nginx/site-confs/

You should add an entry for foundry:

    nano foundryvtt

Add the following content to the file:

    # only serve https
    map $http_upgrade $connection_upgrade {
            default upgrade;
            '' close;
        }

    server {
            listen 443 ssl http2;
            server_name yourdomain.com www.yourdomain.com;  # add your domain name here. if you want to use both with and without www add both here.

            # make sure ssl is enabled
            include /config/nginx/ssl.conf;

            client_max_body_size 0;
            ssl_session_cache shared:SSL:10m;
            proxy_buffering off;

            location / {
                include /config/nginx/proxy.conf;

                resolver 127.0.0.11 valid=30s;

                set $upstream_proto http;
                set $upstream_app foundry;
                set $upstream_port 30000;
                proxy_pass $upstream_proto://$upstream_app:$upstream_port;
                } 
    }


Add your domain name on the indicated line.

Save and close.

Restart swag, so that the new config is loaded.

    docker-compose restart
    
Verify everything works by going to www.yourdomain.com. You should see the foundry login screen.


## Updating 
Updating is done by stopping, removing and redeploying the stack. Before you do this, **shut down your game world.** You may want to **create a backup** as well. 
to update your server, you will need to get a new timed url and replace the existing one in docker-compose.yaml

see: 
* [Get a timed url](#get-a-timed-url)
* [Deploy the stack](#deploy-the-stack)

      
Run:

      cd ~/swag-foundry
      docker-compose rm --stop
      nano docker-compose.yaml    

edit the file

now restart your containers

      docker-compose up -d      

Again **close your world** and **back up your data**
      

# Wrapping up
Some things not covered here, but which may be useful:

* **backups** make sure you backup your world regularly. I personally have a script that creates a backup every morning using rsync. I may add a how-to later if people are interested
* **extra options in felddy's docker container** this guide is long enough as it is. however, Felddy has added a great number of options to run scripts, configure your server, have a certain world be ready, even a custom login screen. Check out his options [here](https://github.com/felddy/foundryvtt-docker) 
* **searchable resources** Foundry's search function, quite frankly, sucks. My workaround is to have a separate container running [pigallery](https://github.com/bpatrik/pigallery2) which is a free photo album (like google photos) that allows me to search different photos based on keywords. So if I'm looking for a chest, or a candle or a bridge to plop into my game I can easily do that.  
* **HTTP Basic auth** an extra authentication step that limits people's access to your foundry server, even before they hit the logon screen. I may upgrade this to a best practice, but I want to test it out for myself first. A How-to guide can be found [here](https://docs.nginx.com/nginx/admin-guide/security-controls/configuring-http-basic-authentication/).
* **portainer** portainer adds some features to docker that look good and may be helpful. This includes a web interface and some cool tools to manage your containers. There's a short guide below:

## (optional) Install portainer
Portainer is a container management system. It basically adds a web interface to docker and gives you some handy tools. You can absolutely do without. It just makes life that little bit easier. 

As portainer itself runs in docker, deploying it is as simple as running two commands

     docker volume create portainer_data

This creates a persistent place to store some of the container's data. Usually containers will lose all data when you restart the container. This is a feature that makes containers more predictable and more secure. But sometimes you need certain data, such as config files to remain after you have restarted a container. That is where volumes come in. Basically you are telling docker to reserve a place called portainer_data where this data can be stored.   
     
     docker run -d --name=Portainer --hostname=Portainer -p 8000:8000 -p 9000:9000 --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data -e TZ='Europe/Amsterdam' portainer/portainer-ce
     
This tells docker to start portainer. The variables are:

    docker run               --> tell docker to run a container
    -d                       --> run in daemon or detached mode. basically run in the background
    --name=Portainer         --> the name that docker uses to identify this container
    --hostname=Portainer     --> the name other computers use to identify this portainer on the network
    -p 8000:8000             --> map port 8000 on your host to the same port in the container. Port 8000 is used mostly for managing other portainer instances, so I'm not sure if you need this. 
    -p 9000:9000             --> map port 9000 on your host to the same port in the container. This means that users that visit http://<hostip>:9000 will be served the portainer web interface. 
    --restart=allways        --> allways restart (recover) the container after a crash.
    -v var/run/.....         --> this maps (shares) what is going on with docker on your host to the container. The container needs this to monitor and manage other containers on your network
    -v portainer_data:/..    --> this maps (shares) the persistent volume you created to your container, so that your configurations remain persistent between restarts
    -e TZ='Europe/Amsterdam' --> set the timezone to where you live. You can change it to where you live. If you remove this part entirely, the container will default to UTC
    portainer/portainer-ce   --> the name of the base image. Docker will look up this container on your host system, or download it from the docker repository if it is not present. 

Test if portainer is working by visiting http://hostip:9000

You should see a registration screen. register and press +create user
  
next, choose the install type: LOCAL
  
you should see a dashboard
--> click on local
--> click on containers
--> you should see 1 container active; you can inspect it using portainer, restart it, stop it or kill it (dont do those last two!).. oh, and maybe next time I should put the warning before the command that will destroy your pretty web interface...




# Thanks
Thank you to all the wonderful people on reddit for helping me to improve this guide. Shout out specifically to u/PriorProject for his help on HSTS and HTTP basic and to u/WindyMiller2006 for his help on integrating SWAG and foundry into a single stack. 


That's it. Happy gaming, and please give some feedback, either by raising an issue, or making a pull request. 
