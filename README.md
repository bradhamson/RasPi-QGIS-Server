# RasPi-QGIS-Server
## Instructions for setting up a QGIS Server on a Raspberry Pi

<p>This is from the first blog post on PopGeo and I thought I might start it off with an interesting project discussion.</p>
<p>Some of you may have already been experimenting with the Raspberry Pi series of microcomputers with projects like a home web server, media center, or any number of other creative ideas. I bought in to this for the first run of Raspberry Pi B boards (the 256 MB of ram version), for $35 I figured I may as well have a go at it. I only ended up using it as an OpenELEC/XBMC media center for playing local media and acting as a DLNA server to stream to other devices in my apartment. This worked fine for a few years, but this board has seen better days and I recently upgraded to a Raspberry Pi 2 for my media center device. I decided to use this as an excuse to try my hand at building a GIS server to host WMS&#8217;s to use as basemaps in Leaflet projects.</p>
<p>Since I began experimenting with Leaflet and other web mapping platforms I have never really been satisfied with the basemaps available. Some of them are beautiful but not really practical, while other services are extremely practical but leave a bit to be desired in the style department. I wanted to be able to have total control over the product without having to learn CartoCSS or subscribe to a web mapping service. So I decide to give my original Raspberry Pi a new life as a QGIS Server.</p>
<p>I have a ton of experience working with ArcGIS Server, but I don’t have the resources to run one myself at home. Since I’ve been learning how to integrate QGIS into my daily workflows and the Raspberry Pi is an open source Linux centric device, I decided to give QGIS server a try.</p>
<p>When I first embarked on this project there was no guide specific to setting up a QGIS Server on the Raspberry Pi, in fact the official QGIS Server configuration documentation was out of date. I had to rely upon a few sources and a lot of trial and error to get the results I wanted. Hopefully this will serve as a complete resource for anyone attempting to go through with this.</p>
<h3>What You Need</h3>
<ul>
<li>A Raspberry Pi with an Ethernet port (original B, B+, 2 etc)</li>
<li>An SD or MicroSD freshly formatted to FAT32 (depending on the Pi)</li>
<li>Power supply, internet connection, usb keyboard, other PC on the network</li>
<li>The latest <a href="http://www.raspberrypi.org/downloads/" target="_blank">Raspbian</a> image</li>
</ul>
<h3>Basic Setup</h3>
<p>I’m not going to spend too much time on this since this process is <a href="http://elinux.org/RPi_Easy_SD_Card_Setup" target="_blank">well documented</a>, but I will outline some of the parameters in the initial configuration that are crucial for continuing this project.</p>
<p>Start by flashing the official Raspbian image, attaching your keyboard and monitor to the Pi, and running raspi-config. In this menu you will need to:</p>
<ol>
<li>Expand the filesystem</li>
<li>Enable SSH</li>
<li>Set region information</li>
<li>Set to boot to command line interface</li>
<li>Choose a hostname</li>
</ol>
<p>From this point on we can continue through an SSH session. If you are using windows I suggesting using PuTTY as your client.</p>
<p>Use <code>ifconfig</code> from the Pi to figure out your Raspberry Pi’s IP address. It should be something like 192.168.X.X depending on how your network is set up. Open an SSH session on you home PC with your Pi’s IP as the hostname. You will be prompted to enter a username and password.</p>
<p>Defaults:<br />
<code>Username: pi</code><br />
<code>Password: raspberry</code></p>
<p>(I STRONGLY suggest changing the default password to something much more complex or removing the Pi user once you add some others. Once this server is exposed to an unsecured network it will be <a href="http://www.reddit.com/r/raspberry_pi/comments/2xumdy/raspberry_pi_webserver_just_got_hacked/" target="_blank">extremely vulnerable</a> if the default credentials remain intact.)</p>
<p>We will then add 2 users, an admin with sudo permissions and a publishing user with no sudo permissions.<br />
<code>sudo adduser USER1</code><br />
Enter a strong password and repeat for the next user</p>
<p>Then we will grant the first user administrative privileges<br />
<code>sudo visudo</code><br />
Enter this line under the line for the root user (mimics the pi user permissions)<br />
<code>USER1 ALL=NOPASSWD: ALL</code><br />
The file should now look like this:<br />
<code># User privilege specification</code><br />
<code>root ALL=(ALL:ALL) ALL</code><br />
<code>pi ALL=NOPASSWD: ALL</code><br />
<code>USER1 ALL=NOPASSWD: ALL</code></p>
<p>Next we will remove the pi user, so go ahead and start a new SSH session as your new admin user then run:<br />
<code>sudo deluser --remove-home pi</code></p>
<h3>Upgrade the Operating System</h3>
<p>This part of the process is made much more difficult by the Pi’s architecture, the version of Debian that Raspbian is based off of, and the lack of documentation from QGIS about the availability of packages for different system architectures. Currently QGIS Server support for the Raspberry Pi’s architecture is not available for this Raspbian image, so we will have to manually upgrade from Debian 7 (Wheezy) to Debian 8 (Jessie). This is not difficult, but may take a while to download, unpack, and install all of the necessary components.</p>
<p>We will start by adding the QGIS repositories to the sources.list file<br />
<code>sudo nano /etc/apt/sources.list</code><br />
Add these lines:<br />
<code>deb http://qgis.org/debian wheezy main</code><br />
<code>deb-src http://qgis.org/debian wheezy main</code></p>
<p>We will need to change all instances of “wheezy” in this file to “jessie”<br />
<code>deb http://archive.raspbian.org/raspbian <del datetime="2015-03-20T03:05:15+00:00">wheezy</del> jessie main contrib non-free</code><br />
<code>deb-src http://archive.raspbian.org/raspbian <del datetime="2015-03-20T03:05:15+00:00">wheezy</del> jessie main contrib non-free</code><br />
<code>deb http://qgis.org/debian <del datetime="2015-03-20T02:00:25+00:00">wheezy</del> jessie main</code><br />
<code>deb-src http://qgis.org/debian <del datetime="2015-03-20T02:00:25+00:00">wheezy</del> jessie main</code></p>
<p>We then add the repo’s public keys<br />
<code>gpg --recv-key DD45F6C3</code><br />
<code>gpg --export --armor DD45F6C3 | sudo apt-key add -</code></p>
<p>Then we can update Apt&#8217;s sources<br />
<code>sudo apt-get update</code></p>
<p>Upgrade all installed packages<br />
<code>sudo apt-get upgrade</code></p>
<p>Upgrade to new distribution version<br />
<code>sudo apt-get dist-upgrade</code></p>
<h3>QGIS Server Installation and Configuration</h3>
<p>First we should install the QGIS desktop suite. Even though this will be running as a headless server, it is still necessary sometimes to work from QGIS desktop remotely.<br />
<code>sudo apt-get install qgis</code></p>
<p>Next we install Apache as our webserver<br />
<code>sudo apt-get install apache2</code><br />
Go ahead and test that Apache is working by entering the Pi&#8217;s IP in your browser&#8217;s address bar. It should return a page like this:</p>
<p><a href="http://www.popgeo.net/wp-content/uploads/2015/03/apache.png"><img src="http://www.popgeo.net/wp-content/uploads/2015/03/apache.png" alt="apache" width="810" height="277" class="alignnone size-full wp-image-77" /></a></p>
<p>Now install QGIS Sever and the required Apache packages<br />
<code>sudo apt-get install qgis-mapserver libapache2-mod-fcgid</code></p>
<p>Next enable CGI<br />
<code>a2enmod cgid</code></p>
<p>Then we restart Apache (you will end up doing this a lot)<br />
<code>sudo service apache2 restart</code></p>
<p>Now go ahead and test to see if your QGIS Server is working by entering your Pi&#8217;s IP in you browser&#8217;s address bar followed by this:</p>
<p><code>http://yourIP/cgi-bin/qgis_mapserv.fcgi?SERVICE=WMS&amp;VERSION=1.3.0&amp;REQUEST=GetCapabilities</code></p>
<p>You should see some XML returned.</p>
<h3>Publishing QGIS Services</h1>
<p>Alright, this is the super easy part. All you have to do to create mapping services for QGIS Server is copy a finished QGIS document (with OWS service properties filled out) into the /usr/lib/cgi-bin/ directory.<br />
<div id="attachment_85" style="width: 885px" class="wp-caption alignnone"><a href="http://www.popgeo.net/wp-content/uploads/2015/03/props.png"><img src="http://www.popgeo.net/wp-content/uploads/2015/03/props.png" alt="Project properties menu" width="875" height="625" class="size-full wp-image-85" /></a><p class="wp-caption-text">Project properties menu</p></div></p>
<p>There are a few ways you can go about doing this like using SCP to copy files and data from your desktop to the server, use SSH to do the same thing, or use a remote desktop client to copy the files up to the server. You could even use the QGIS desktop we installed earlier to create projects directly on the server (this will be a slow and painful task though). Its really a matter of personal preference which way you decide to go.</p>
<p>For SCP I like to use <a href="http://winscp.net/eng/index.php" target="_blank">WinSCP</a>.</p>
<p>If you would like to be able to connect to the QGIS server over a Remote Desktop Connection from Windows, you are going to need to install xrdp<br />
<code>sudo apt-get install xrdp</code></p>
<p>Now all you have to do is open you remote desktop client and enter your Pi&#8217;s IP address, you username, and password to connect to the desktop gui from your home computer.</p>
<p>I like to organize my /cgi-bin/ directory by projects, so I create a new directory for each project. If you want to do this you will need to copy or link to the qgis_mapserv.fcgi executable and the wms_metadata.xml file. You can do it by copying and pasting directly into the directory, or linking like this:<br />
<code>sudo ln -s ../qgis_mapserv.fcgi</code><br />
<code>sudo ln -s ../wms_metadata.xml</code></p>
<h3>Network Setup</h3>
<p>Networking is not my specialty so a lot of this section will refer to outside resources to accomplish its goals. This is also dependent upon the individual user&#8217;s hardware so some research may be necessary to get this working on your existing setup.</p>
<p>The first thing we need to do is establish a Static IP for the Raspberry Pi. An excellent guide to do this can be found <a title="here" href="http://elinux.org/RPi_Setting_up_a_static_IP_in_Debian" target="_blank">here.</a></p>
<p>Next we will need to forward ports. Currently the Pi is only visible on the local network. This has to do with how your router is currently configured. As of now it most likely works like this: You make a request for content that goes through port 80 on your router to the internet and it returns it to you through the router. We want to set it up so that others on the internet can make requests to your server and you can serve content out to them. This is called port forwarding.</p>
<p>Apache listens for traffic on ports 80 (HTTP) and 443 (SSL), so we need to forward those for this to work. How this process is run is entirely dependent on your hardware, so you will need to consult some <a href="http://portforward.com/" target="_blank">outside documentation</a> to get this set up.</p>
<p>If you use Verizon as your ISP then it is very likely that you will not be able to use port 80 since they block access. You will need to <a href="http://httpd.apache.org/docs/2.2/bind.html" target="_blank">configure Apache to listen to a different port for HTTP requests.</a></p>
<p>Once this is set up you need to assign a human readable domain name to your server. Right now you can access your index file and map services using your external IP. If you want to more easily use your services in projects or let other people access your services then it would probably be best to assign it a good domain name. Most people are not used to navigating to a website using its IP address, so picking a good memorable name will greatly help with usability.</p>
<p>You have a few options for where you can source your domain name. You can purchase and register one from any number of companies or you can use a free service. <a href="https://www.dnsdynamic.org/" target="_blank">DNSDynamic</a> is a good one to use and it is simple to set up. All you have to do it pick a domain name and point it to your Pi&#8217;s external IP Address. Now you or any other user can easily access map services from your QGIS Server.</p>
<p>Thats it! Your Raspberry Pi QGIS server is now up and running and serving out WMS&#8217;s, WFS&#8217;s, and WFTS&#8217;s to you or anyone you want. Hopefully this guide has been helpful to you and any future web mapping projects you undertake.</p>
<h5>Sources</h5>
<p><a href="http://anitagraser.com/2012/03/30/qgis-server-on-ubuntu-step-by-step/" target="_blank">QGIS Server on Ubuntu</a>, <a href="https://hub.qgis.org/wiki/17/QGIS_Server_Tutorial" target="_blank">Official QGIS Server Tutorial</a>, <a href="http://readwrite.com/2014/06/27/raspberry-pi-web-server-website-hosting" target="_blank">Raspberry Pi as a webserver</a>, <a href="http://gis.stackexchange.com/questions/116320/how-to-install-gdal-and-qgis-on-a-raspberry-pi" target="_blank">QGIS on a Raspberry Pi</a></p>
