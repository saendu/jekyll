---
layout: article
title:  "Projects"
author: sandro
categories: [ Article ]
permalink: /projects.html
image: assets/images/project-desk.jpg
---
When Corona hit the fan and we all stuck in a lockdown, I've searched for new hobbys. The result was a couple of IT project that I've finally started. Below you get a brief overview about some of the noticeable project. If you also want to get your hands dirty, check if there is a article linked below. Furthermore you should also find links to Github and if available to the apps itself. Most of them are still in progress, so don't go to hard on me. ;-)

Currently there are six main projects running on my private cloud:
- [Raspberry Pie Cluster](#raspberry-pie-cluster)
- [Fuebiapp](#fuebiapp)
- [Microservices built with OAM / Dapr](#microservices-with-oam-and-dapr) 
- [Private website (sandrofelder.ch)](#private-website)
- [Mail Gateway](#mail-gateway)

## Raspberry Pie Cluster
![Cluster]({{ site.baseurl }}/assets/images/rpi4-cluster.jpg)
**Tech**: Raspberry Pie 4, Ubuntu, ARM64, Kubernetes, Docker, Nginx

Obviously enough I needed something to let all my projects run. So a Raspberry would have been a good choice. But since buying just 'one' RPI was too easy, I instead bought four and build my own Raspberry Pie 4 Cluster. And what would be a cluster without an orchestrator. That is why I decided to create a Kubernetes Cluster with my Raspberries. 

If you wanna know more or maybe build your own, then read me article [Building your own private cloud with Raspberry Pi 4 and Kubernetes]({{ site.baseurl }}/your-own-private-cloud/)

## Fuebiapp
![Fuebiapp-Banner]({{ site.baseurl }}/assets/images/fuebiapp-banner.jpg)
**Tech**: Jitsi, React, Redux 
**Github**: [fuebiapp-web](https://github.com/saendu/fuebiapp-web), [fuebiapp-docker](https://github.com/saendu/fuebiapp-docker)

In my local dialect 'Fuebi' means 'afterwork beer'. Fuebiapp.com (or afterworkbeer.com) was one of my first Corona projects. Reason was I was missing the social times when we could have a beer together and get on each others nerves with poking others to take another round or just have a shots round.
So Fuebiapp is a conference/meeting platform where you can hang out and have a beer together. 

Some of the features include:
+ See all participants at once
+ Announce a shots round
+ Beer counter
+ Poke somebody to have another beer

**Websites**: [fuebiapp.com](https://fuebiapp.com), [afterworkbeer.com](https://afterworkbeer.com) 

## Home Control API
![Cluster]({{ site.baseurl }}/assets/images/home-automation.jpg)
**Tech**: NodeJS, Express
**Github**: [HomeControl](https://github.com/saendu/homecontrol)

The Home Control project allows me to control my IKEA smart plugs with Siri. It consists of a simple Express backend with an API that controls an IKEA gateway.

If you wanna know more, read my article [Hey Siri, I wanna sleep!]({{ site.baseurl }}/home-automation) that describes the project in detail.

## Private website 
![Website]({{ site.baseurl }}/assets/images/website.jpg)

**Tech**: Jekyll
**Github**: [Jekyll](https://github.com/saendu/Jekyll)

The main idea why I created a private website in the first place, was to publish articles about my projects. Almost everybody I was explaining what I did during COVID was interessted to build something similar. So I figured what not do something for the community and contribute back and share all my knowledge. Afterall I also came only that far because other people did the same before me. So here I go. 

If you find any typos, mistake or any other blunder let me know in the comments section, or create an issue on Github. I will be very greatful! 

## Mail Gateway
**Tech**: Raspberry Pie 4, Kubernetes, Ubuntu

Comming soon. 

