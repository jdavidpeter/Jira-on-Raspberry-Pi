Jira on Raspberry Pi 3

Ever wondered if a Raspberry Pi 3+ can host a fully functional Jira Core or Jira Host instance using a proper database? Yes, it can.
I wouldn't recommend this for any production project, nor for anything with 10+ frequent contributors. But for a home ticket system, a hobby project, a Pi running in your home is more than sufficient.
Here's htop to show how much toll it takes on my system:

  1  [||||||||                                         12.3%]   Tasks: 42, 170 thr; 2 running
  2  [                                                  0.0%]   Load average: 1.29 1.30 0.77
  3  [|||                                               3.9%]   Uptime: 5 days, 15:32:16
  4  [|||||||||||||||||||||||||||||||||                59.7%]
  Mem[|||||||||||||||||||||||||||||||||||||||||||||675M/927M]
  Swp[||||||||||||||||                            573M/2.00G]
  
I did the project about 3 months ago, so I might not remember all the important steps, but here's a very detailed installation guide. If something is missing, let me know or just raise a pull request.
The entire thing including research took about 6 hours from unboxing the Pi to the working instance. With this guide, you can do it in probably 1 hour. Let me know if I was mistaken.

Important notice:
 - Atlassian does not provide support for Raspberry Pi, not matter what license you have or what OS you run on your Pi. If something is messed up, you can go to their community forums, but assume that you're on your own.
 - The primary bottleneck for running Jira on a Pi is memory. Make sure you use a lightweight OS and get rid of your other memory intensive apps.


#0. Prerequisite

This guide was made for a Rasberry Pi 3, running a 4.9.59-v7+ RASPBIAN STRETCH WITH DESKTOP. However, Desktop was disabled using the sudo raspi-config command, based on this guide: http://ask.xmodulo.com/disable-desktop-gui-raspberry-pi.html.
Since we run headless, with no GUI, we need SSH to access the Pi. Follow the official guide based on your configuration: https://www.raspberrypi.org/documentation/remote-access/ssh/


#1. Increase swap

The primary bottleneck is memory. If the Jira JVM can't allocate enough heap, it will crash. I know, swap is slow (but not as much since it's on an SD card), but your system is clever enough to keep hot memory in the actual RAM (or even in cache), so performance will be acceptable even with swap usage.
Another question is how much this will kill your SD card as flash has finite write cycles, but I have no answer to that. I use 2 GB swap because why not, but I would recommend at least 1 GB.

 1. Go to /etc/dphys-swapfile with your favourite editor
 2. Replace the relevant line by #CONF_MAXSWAP=2048
 3. Reboot
 4. Verify e.g. with htop that your swap is now 2G. Look for the line Swp on the top.


#2. Set up a proper database.

The built-in, in-memory database of Jira is not a safe bet, it handles power outages and other nasty things badly. We need something more robust and easier to back up. Thankfully, RASPBIAN has a mysql implementation. Let's use that.
I go with my own naming scheme, but you can choose your own names.

 1. Get mysql
      sudo apt-get install mysql-server
      sudo mysql -u root
 2. Create user and database for Jira
      CREATE USER 'jirapi'@'localhost' IDENTIFIED BY 'YourDbPasswordGoesHere';
      CREATE DATABASE jirapidb CHARACTER SET utf8 COLLATE utf8_bin;
      GRANT SELECT,INSERT,UPDATE,DELETE,CREATE,DROP,ALTER,INDEX on jirapidb .* TO 'jirapi'@'localhost' IDENTIFIED BY 'YourDbPasswordGoesHere';
      flush privileges;
      quit
 3. Verification:
      mysql \--user=jirapi \--password='Mar!Pi001Mar!Pi001' \--database=jirapidb


#3. Deploy Jira to your Pi

In order to install Jira, we will need a working version of Java, a dedicated user to run Jira and the linux version of Jira Core or Jira Software dowloaded from Atlassian home.

 1. Make sure that you have a java version installed that is supported by JIRA
    In my case java 1.8 was installed, you can find this out by typing java -version

 2. Create a jira user
      sudo adduser jira
      Pass: YourPasswordGoesHere
      usermod -aG sudo jira
    Switch over to new user

 3. Download the tar.gz archive from https://www.atlassian.com/software/jira/download
    I was using version JIRA v7.6.2
    Untar them to a folder in the home of the jira user. That folder will host your instance. I will refer that folder as $JIRA_BINARIES. Create an env variable if you copy-paste the commands from this guide to cmd.


#4. Customise Jira for the Pi

 1. Set the jira home directory by manipulating some Jira files.
      nano $JIRA_BINARIES/atlassian-jira/WEB-INF/classes/jira-application.properties
      edit: jira.home = "~jira/jira/jira_home"
      nano $JIRA_BINARIES/bin/setenv.sh
	  edit: JIRA_HOME="~jira/jira/jira_home"
 
 2. Set up ipv4 as preferred protocol. This will save a lot of hassle. If you know what you do, you can go with ipv6, although I ran to this issue:
    https://community.atlassian.com/t5/Jira-questions/Tomcat-server-unavailable-on-tcp6-IPV6/qaq-p/229018
      nano $JIRA_BINARIES/bin/setenv.sh
	  edit: -Djava.net.preferIPv4Stack=true should be added in the JVM_REQUIRED_ARGS string
 
 3. Install the db connector for mysql
    Follow step #2 in this guide: https://confluence.atlassian.com/adminjiraserver076/connecting-jira-applications-to-mysql-945110796.html


#5. Start Jira and attach it to your database.

 1. Start jira by running $JIRA_BINARIES/bin/start-jira.sh
 2. Connect to the web interface running on your host on the default 8080 port. You need to find out the IP of your pi first.
 3. Run the setup wizard. It may crash 2-3 times, rendering your whole Pi unresponsive. Don't give up, do a reboot and run the wizard again. It will continue. Use the wizard to connect to your mysql database.
    https://confluence.atlassian.com/adminjiraserver071/running-the-setup-wizard-802592199.html


#6. Nice extras

 1. You might want to make sure your Jira instance starts when your Pi reboosts for whatever reason. There are couple of ways to achieve this, I'd recommend using /etc/init.d. There's an example file named jira in this repo which you should copy to /etc/init.d/jira. Then run the following:
    update-rc.d jira defaults
  
 2. In order to access the Jira from the world wide web, you either need static IP, a thing not commonly provided for residential users, or a dynamic DNS. I recommend duckdns.org, a free dynamic DNS provider. It has very straightforward setup guides both for Pi and various routers.
 
 3. Set up auto backup. If your instance has low usage, you can go with the default XML backup from Atlassian. That's not a proper tool as it might create a corrupt backup if someone is actively writing to the Jira database during the time of backup, but for a home instance, schedule it to run every night when folks sleep and be fine with it. You still need to backup file attachments manually or writing a script for that. See more in the guide:
    https://confluence.atlassian.com/adminjiraserver071/automating-jira-application-backups-802592966.html
    
 
#7. Important warning

 1. You must be aware that by default, Jira uses plain http and not https. There's a convoluted guide on Atlassian about setting up Tomcat with https (https://confluence.atlassian.com/adminjiraserver071/running-jira-applications-over-ssl-or-https-802593051.html), but I haven't tried it so far. Without https, all the data you transmit, including your user name and password is easy to capture. Should you keep http, make sure you never access the instance by an admin user from the open web, so you can lessen the damages caused by a potential intruder who stole your credentials. E.g. create a separate admin user for admin purposes only and never use that user outside your safe local network. All other users shouldn't have admin rights.
    Still, using http from the open web is very-very dangerous and you expose yourself to data theft or even worse.
 
