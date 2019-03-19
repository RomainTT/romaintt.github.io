---
title: "Controlling my 3D printer from the internet"
description: "How I control my 3D printer from anywhere in a secure way."
date: 2019-xx-xx
layout: post
categories: raspberry 3d-printing
language: english
---

So I have this 3D printer which is awesome and I have some new ideas of things to print 
every day. But the point is that printing takes **a lot** of time, far more than I can spend 
at home, watching my printer with my own eyes. In order to print things even when I am not 
at home, I needed a way to control and check my printer from the internet, so that I can do
it from anywhere. Of course that raised a lot of concerns: safety, security, privacy, 
reliability and so on. I finally built a solution which suits me well and I want to share 
my setup with you.

## Overview

Here is a picture to summarize my setup. Every part of it is detailed in the next sections.

![Schema of principle](/images/octoprint/setup_principle.png)

## Step 1: The hardware

Here is all the hardware I am using:

* **3D printer MicroDelta Rework from [RepRap France](https://www.reprap-france.com/)**  
  I bought this a few years ago and it works great. 
  It cost 400€ as a kit to mount myself.
  It has a USB port to control it with a computer.
* **Raspberry Pi 3 B+**  
  At this date, some full kits can be found for 60€ with a power supply, a casing and 
  an SD card. What is great with version 3B is the integrated WiFi, no need for a dongle!
  This micro-computer is more than enough to run all the software I needed, in fact 
  a Raspberry Pi 1 works fine as well.
* **Webcam**  
  Any webcam supporting [Motion JPG](https://en.wikipedia.org/wiki/Motion_JPEG) are fine
  to be used with Octoprint.
* **Freebox mini**  
  My Internet service provider is "Free" and provides me with a router (also doing modem). 
  This is what I use to connect my local network to the outside world.

## Step 2: The RPI basic setup

I flashed the SD card with the last [Raspbian](https://www.raspberrypi.org/downloads/raspbian/) release.
On Windows, I used [Etcher](https://www.balena.io/etcher/) to write the image on the card. 
On Linux, it can be done with a few command lines like it is explained 
[here](https://www.raspberrypi.org/documentation/installation/installing-images/linux.md).

To connect to my RPI the first time, I needed to enable SSH. To do so I created a file named 
`ssh` in the `/boot` partiton of the SD card. Then I started the RPI with this card. 
I connected it to my router with an ethernet cable, and looked for its address on the *Freebox* 
web interface in the *DHCP* menu.

Using a standard console on Linux or [Putty](https://putty.org/) on Windows, I opened a ssh session 
on the RPI, default login and password being `pi` and `raspberry`.

Then comes the configuration with the command `raspi-config`:
* **To connect the RPI to my WiFi network**,  
  because I can't let it near the router (I don't have much space)
  but if you can, just let the ethernet cable it will be much better ! The RPI 3 B+ has an integrated Wifi antenna
  and does not need an external dongle, which is great !
* **To change user password**  
  Please select a strong password !
* **Expand the file system**  
  If the file system does not use the whole SD card yet.

Then of course the system needed to be upgraded:

    sudo apt-get update
    sudo apt-get upgade

Finally, I checked that my printer was visible in the system when I connected it on the USB port. 
In my case, `/dev/serial1` appears when I connect it.

## Step 3: Octoprint

### Installation and configuration

### Test with the printer

### The webcam

## Step 4: Connect the Pi to the outside world

### Secure the Raspberry Pi

### Port redirection

For instance, 7654.

### Redirect domain name to my home

## Step 5: Access to Octoprint in a secure way

The easiest (and dumbest) way of making octoprint accessible to the outside
world would be to forward port 80 of the Raspberry Pi to a port of the router
(here my Freebox), so that one only has to reach home.taprest.fr:8080 for
instance, to reach the interface of octoprint. **But that is stupid**, because
that means anyone in the world can take control of my printer, upload malicious
files, watch my webcam and so on. [This article](https://dshield.org/forums/diary/3D+Printers+in+The+Wild+What+Can+Go+Wrong/24044/)
explains that in more details.

In my case, I chose the technique of **SSH tunneling**. In simple words, I
first create a tunnel between my computer (anywhere outside my home) and the
Raspberry Pi using SSH. Afterwards, I access the website of octoprint through
that tunnel, which encrypts everything. More information about this technique
can be found on [the website of SSH](https://www.ssh.com/ssh/tunneling/).

To do so, I kept the port redirection of the SSH connection as I explained in
the previous section of this article, and configured nothing else !

Now, to establish the tunnel and to connect to the Octoprint server from an 
external host, several options in function of the system. 

On **Linux**, I run the following command:

    ssh -L local_port:local_host:distant_port username@distant_host -p ssh_port

which, in my case, can be for instance:

    ssh -L 8080:localhost:80 pi@home.taprest.fr -p 7654

After the connection is established, I can access the Octoprint interface on my
browser by reaching the address `localhost:8080`, which is tunneled to the port
80 of the Raspberry Pi.

On **Windows**, I use [Putty](https://putty.org/) to establish the tunnel
(there is a dedicated menu for this functionality). And the rest is all the
same.

On **Android**, I use the app [termius](https://www.termius.com/) for the same
purpose. And while writing this article, I’m just discovering that termius can
be used on Desktop too. It may be prefered over Putty.

To deactivate the tunnel, I just close the connection and Octoprint is not 
reachable anymore.

To conclude, this way of doing is as secure (and insecure) as a common SSH 
connection to the Raspberry Pi.
