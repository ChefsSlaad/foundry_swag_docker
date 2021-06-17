# foundry_swag_docker
This is a how-to on running foundry-vtt in a docker container and securing the connection using nginx and letsencrypt. If that does not mean anything to you, this is basically a how to on running a reasonably secure version of foundry. It is: 
**containered** - even if someone is able to hijack your foundry system through a vulnerability or by guessing your password, they cannot go any further. they're basically stuck in your container. It also has all sorts of portability and scalability advantages that do not really matter for your single home server.
**encrypted** - the connection between your player's PC and your server is encrypted, which means that other users cannot easily steal your password or hijack your connection.
**not idiot proof** - you may notice I use some carefull language. That is because, while I know this will make you more secure than doing nothing, there is no such thing as an unhackable system. So, please **choose a strong password** and **update your system**. Also, dont go arround daring other people to hack you. That's just stupid.


version 0.02 - Aboleth


# Disclaimer
* this guide is written by me, based on my own experience self-hosting foundryVTT. There are bound to be mistakes in this guide. Please contact me if I missed anything or if you feel this guide could be improved. Or, you know, make a pull request. this is github after all
* Im assuming you have a passing familiarity with linux, the terminal and a rudimentary understanding of containers. I may forego or adjust this assumption in the future, but right now it is what it is. 
* I'm assuming you own a licence key for foundry-vtt
* I'm assuming you have a static IPv4 adress. find out what your IP is easily using www.whatsmyip.org

# Overview
This guide is set up in 5 steps:
* preparations
* setting up the host (the system or server you will be using to run foundry vtt)
* setting up the foundry-vtt container
* setting up the SWAG container
* wrapping up


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
* test that the domain forwards to the correct IP 
  * go to www.mxtoolbox.com/DNSLookup.aspx 
  * enter your domain name
  * verify that it resolves to your IP address. If it doesnt, wait a bit and try again. 

### duckdns(the cheap and easy option)
* create an account with duckdns.org
* enter the domainname you want to register
* duckdns will look up your public IP and fill that in. If it's incorrect for whatever reason, fill in the correct ip adress
* test that the domain resolves to the correct IP
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

anyway, before we start, lets make sure everything is up to date:

    sudo apt update
    sudo apt upgrade


## install docker
Docker has a really good install guide for multiple systems. 

[guide for debian](https://docs.docker.com/engine/install/debian/)

There are also some recommende [post-install steps.](https://docs.docker.com/engine/install/linux-postinstall/) I personally configure docker to be managed by  as a non-root user, and I configure docker to start on boot.  

## (optional) install portainer
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
    

Test if portainer is working by visiting http://<yourip>:9000

you should see a registration screen:
![portainer login](https://www.danielmartingonzalez.com/assets/posts/en/docker-and-portainer-in-debian/portainer-register.jpg)

  
next, chose the install type: LOCAL
![configure install](https://www.danielmartingonzalez.com/assets/posts/en/docker-and-portainer-in-debian/portainer-initial-settings.jpg)
  
you should see a dashboard
--> click on local
--> click on containers
--> you should see 1 container active; you can inspect it using portainer, restart it, stop it or kill it (dont do those last two!).. oh, and maybe next time I should put the warning before the command that will destroy your pretty web interface...
   

## setting up a data folder for your containers
It's a good idea to create a folder where you will store all your art assets in one easily searchable place. As your library will probaby grow (god knows it never shrinks), its a good idea to put some thought into the organsiation now, as its a pain to change it later.
  
I personally use a structure:
  
- resources
  - assets
  - maps
  - tokens
  - campaign specific stuff. 
    
For the rest of the guide, I am assuming you hae a folder called resources that contains all this stuff. 

# setting up foundry-vtt
* chosing a foundry-vtt docker container
* deploying the docker container
* testing foundry
* Updating 

# setting up SWAG
* configuring your router
* configuring reverse proxy
* deploying the SWAG container
* testing SWAG
* updating

# Wrapping up
* backups
* ...
