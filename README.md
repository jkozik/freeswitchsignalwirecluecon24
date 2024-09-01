# Freeswitch Setup using Cluecon 2024 Lunch and Learn
I attended the [Cluecon 2024](https://www.cluecon.com/) conference this year.  It was my 5th time.  The conference was excellent.

As part of the Wednesday Lunch and Learn with Luca Pradovera, he ran through a Freeswitch + Signalwire demo. It was a compelling demo that showed how easy it is to add Signalwire to Freeswitch.  The repository for the demo, [freeswitch-cluecon-lab](https://github.com/signalwire/freeswitch-cluecon-lab) contained everything he used. So I decided I wanted to re-create the demo on my home LAN.

Luca indeed made it look easy, but when I tried it, I had some learning curve issues.  I document my experience here to help others who are new to setting up Freeswitch+Signalwire in a container environment.  Thanks to Luca, Brian and Jon for their help.

In summary, I cloned Luca's repository, built it, exec'd into the container and connected it to signalwire, configured the SIP clients, verified that they registered, make the SIP clients call each other, setup my home LAN firewall, added Signalwire incoming and outgoing extensions in the freeswitch dial plan and finally had a SIP client call my mobile phone and had my mobile phone call my SIP client. 

I learned alot about the tools and best practices for setting up and troubleshooting Freeswitch.  In the writeup below, I included descriptions of how I verified the SIP registration, traced the SIP calls, traced the dialing plan, and tweaked the parameters of Freeswitch.

For reference, here's the architecture chart for what I setup on my Home LAN.  

![image](https://github.com/user-attachments/assets/89ac8b83-ed67-45aa-a42b-e16d9ad01190)


# Install Freeswitch from container

I am running a Ubuntu22.04 server on my home LAN.  I have docker and kubernetes installed.  I do almost all of my work in containers. 

I followed the [freeswitch-cluecon-lab](https://github.com/signalwire/freeswitch-cluecon-lab) repository README file.  I got a Signalwire account, I cloned the repositiory. I edited the env.example file and saved it as .env. I did a docker compose build and finally a docker compose up. 

## Build Freeswitch
### Clone
```
jkozik@u2004:~/projects$ git clone https://github.com/signalwire/freeswitch-cluecon-lab.git
Cloning into 'freeswitch-cluecon-lab'...
remote: Enumerating objects: 388, done.
remote: Counting objects: 100% (388/388), done.
remote: Compressing objects: 100% (273/273), done.
remote: Total 388 (delta 134), reused 338 (delta 96), pack-reused 0 (from 0)
Receiving objects: 100% (388/388), 228.58 KiB | 2.33 MiB/s, done.
Resolving deltas: 100% (134/134), done.
```
### Set .env
```
jkozik@u2004:~/projects$ cd freeswitch-cluecon-lab
jkozik@u2004:~/projects/freeswitch-cluecon-lab$ cp env.example .env
jkozik@u2004:~/projects/freeswitch-cluecon-lab$ vi .env
jkozik@u2004:~/projects/freeswitch-cluecon-lab$ cat .env
FREESWITCH_PASSWORD=XXXXXXXXX
SIGNALWIRE_TOKEN=pat_QYsXXXXXXXXXXXXXXXXXXXX
jkozik@u2004:~/projects/freeswitch-cluecon-lab$
```
### Build Freeswitch
```
jkozik@u2004:~/projects/freeswitch-cluecon-lab$ docker compose build
[+] Building 1.4s (17/17) FINISHED                                                                                                             docker:default
 => [freeswitch internal] load build definition from Dockerfile 
 => => transferring dockerfile: 2.58kB
 => [freeswitch internal] load metadata for docker.io/library/debian:bullseye
 => [freeswitch internal] load .dockerignore
 => => transferring context: 2B  
 => [freeswitch  1/12] FROM docker.io/library/debian:bullseye@sha256:0bb606aad3307370c8b4502eff11fde298e5b7721e59a0da3ce9b30cb92045ed  
 => [freeswitch internal] load build context   
 => => transferring context: 1.03kB              
 => CACHED [freeswitch  2/12] RUN groupadd -r freeswitch --gid=999 && useradd -r -g freeswitch --uid=999 freeswitch  
 => CACHED [freeswitch  3/12] RUN apt-get update && apt-get install -y locales wget && rm -rf /var/lib/apt/lists/*     && localedef -i en_US -c -f UTF- 
 => CACHED [freeswitch  4/12] RUN apt-get update && apt-get install -y gosu curl gnupg2 wget lsb-release apt-transport-https ca-certificates     && wge 
 => CACHED [freeswitch  5/12] RUN cat /etc/apt/sources.list.d/freeswitch.list   
 => CACHED [freeswitch  6/12] RUN mv /bin/hostname /bin/hostname.bkp;   echo "echo myhost.local" > /bin/hostname;   chmod +x /bin/hostname
 => CACHED [freeswitch  7/12] RUN apt-get update && apt-get install -y freeswitch-meta-all      
 => CACHED [freeswitch  8/12] RUN apt-get update && apt-get install -y dnsutils     && apt-get clean && rm -rf /var/lib/apt/lists/* 
 => CACHED [freeswitch  9/12] RUN mv /bin/hostname.bkp /bin/hostname   
 => CACHED [freeswitch 10/12] RUN apt-get autoremove                     
 => CACHED [freeswitch 11/12] COPY build/freeswitch.limits.conf /etc/security/limits.d/  
 => CACHED [freeswitch 12/12] COPY build/docker-entrypoint.sh /   
 => [freeswitch] exporting to image                                 
 => => exporting layers   
 => => writing image sha256:9f5a194baf5c1f69ce1a9a65594727c7580688f6dedceed44028c753d8bdef3c   
 => => naming to docker.io/library/freeswitch-cluecon-lab-freeswitch
```
The underlying Dockerfile installs freeswitch from packages onto a Debian bullseye, version 11, which orignally came out in 2021.  Version 11.10 came out in June of 2024. The initial build took about 15 minutes.  The example above, everything was cached and it ran really quick.
Note also, I use "docker compose" -- the new way.  The README said to use "docker-compose".  

### Bring up container
The original docker-compose file didn't work for me.  Maybe I should have used another one, but to get this one to work, I had to uncomment-out the host networking, selecting "network_mode: host" and enabling IPv6.
```
jkozik@u2004:~/projects/freeswitch-cluecon-lab$ docker compose up -d
[+] Running 0/1
 ⠇ Container freeswitch-community  Starting                                                                                                              1.9s
Error response from daemon: failed to create task for container: failed to create shim task: OCI runtime create failed: runc create failed: sysctl "net.ipv6.conf.all.disable_ipv6" not allowed in host network namespace: unknown

jkozik@u2004:~/projects/freeswitch-cluecon-lab$ vi docker-compose.yml
jkozik@u2004:~/projects/freeswitch-cluecon-lab$ cat docker-compose.yml

services:
  freeswitch:
    container_name: "freeswitch-community"
    build:
      context: .
      args:
        SIGNALWIRE_TOKEN: ${SIGNALWIRE_TOKEN}
    volumes:
      - ./conf:/etc/freeswitch
        #sysctls:
        #net.ipv6.conf.all.disable_ipv6: 1
        #net.ipv6.conf.default.disable_ipv6: 1
        #net.ipv6.conf.lo.disable_ipv6: 1
    stdin_open: true
    tty: true
    env_file: .env
    network_mode: "host"
    command: ["freeswitch", "-nonatmap", "-nonat"]
jkozik@u2004:~/projects/freeswitch-cluecon-lab$

jkozik@u2004:~/projects/freeswitch-cluecon-lab$ docker compose up -d
[+] Running 1/1
 ✔ Container freeswitch-community  Started                                                                                                               1.5s
jkozik@u2004:~/projects/freeswitch-cluecon-lab$ docker compose ps
NAME                   IMAGE                               COMMAND                  SERVICE      CREATED          STATUS                    PORTS
freeswitch-community   freeswitch-cluecon-lab-freeswitch   "/docker-entrypoint.…"   freeswitch   13 minutes ago   Up 13 minutes (healthy)

jkozik@u2004:~/projects/freeswitch-cluecon-lab$ netstat -tulp | grep -E "sip|5080"
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 u2004.kozik.net:sip     0.0.0.0:*               LISTEN      -
tcp        0      0 u2004.kozik.net:5080    0.0.0.0:*               LISTEN      -
udp        0      0 u2004.kozik.net:sip     0.0.0.0:*                           -
udp        0      0 u2004.kozik.net:5080    0.0.0.0:*                           -
jkozik@u2004:~/projects/freeswitch-cluecon-lab$
```
In the official freeswitch repository in the [freeswitch/docker](https://github.com/signalwire/freeswitch/tree/master/docker) folder there is a section on [Ports](https://github.com/signalwire/freeswitch/tree/master/docker#ports).  It said to run docker containers with options "--network host".  So, I changed the docker compose file.  But that broke the sysctl lines that tried to disable IPv6. 

Further, it is a good sanity test to see ports 5060 and 5080 visible on the netstat report.
### Exec into the container
The best next step is to login to the container and verify basic that the base freeswitch command line works.
```
jkozik@u2004:~/projects/freeswitch-cluecon-lab$ docker exec -it freeswitch-community  bash

root@u2004:/# ls
bin  boot  dev  docker-entrypoint.sh  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var

root@u2004:/# echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

root@u2004:/# cd /etc

root@u2004:/etc# ls
adduser.conf            debian_version  group        issue          login.defs      netconfig      protocols    rmt            ssl                vulkan
alternatives            default         group-       issue.net      logrotate.d     networks       python3      rpc            ssmtp              wgetrc
apt                     deluser.conf    gshadow      kernel         machine-id      nsswitch.conf  python3.9    security       subgid             X11
bash.bashrc             dhcp            gshadow-     ldap           magic           opt            rc0.d        selinux        subuid             xattr.conf
bindresvport.blacklist  dpkg            gss          ld.so.cache    magic.mime      os-release     rc1.d        sensors3.conf  sysctl.d           xdg
binfmt.d                e2scrub.conf    host.conf    ld.so.conf     mailname        pam.conf       rc2.d        sensors.d      systemd
ca-certificates         environment     hostname     ld.so.conf.d   mime.types      pam.d          rc3.d        services       terminfo
ca-certificates.conf    ethertypes      hosts        libaudit.conf  mke2fs.conf     passwd         rc4.d        shadow         timezone
cron.d                  fonts           hosts.allow  locale.alias   modules-load.d  passwd-        rc5.d        shadow-        tmpfiles.d
cron.daily              freeswitch      hosts.deny   locale.gen     motd            perl           rc6.d        shells         ucf.conf
dbus-1                  fstab           init.d       localtime      mtab            profile        rcS.d        skel           update-motd.d
debconf.conf            gai.conf        inputrc      logcheck       mysql           profile.d      resolv.conf  snmp           vdpau_wrapper.cfg

root@u2004:/etc# cd freeswitch

root@u2004:/etc/freeswitch# ls
autoload_configs  dialplan         freeswitch.serial  ivr_menus        mime.types            README_IMPORTANT.txt  tetris.ttml  voicemail.tpl
chatplan          directory        freeswitch.xml     jingle_profiles  mrcp_profiles         sip_profiles          tls          web-vm.tpl
config.FS0        extensions.conf  fur_elise.ttml     lang             notify-voicemail.tpl  skinny_profiles       vars.xml     yaml
root@u2004:/etc/freeswitch# cd

root@u2004:~# fs_cli
.=======================================================.
|            _____ ____     ____ _     ___              |
|           |  ___/ ___|   / ___| |   |_ _|             |
|           | |_  \___ \  | |   | |    | |              |
|           |  _|  ___) | | |___| |___ | |              |
|           |_|   |____/   \____|_____|___|             |
|                                                       |
.=======================================================.
| Anthony Minessale II, Ken Rice,                       |
| Michael Jerris, Travis Cross                          |
| FreeSWITCH (http://www.freeswitch.org)                |
| Paypal Donations Appreciated: paypal@freeswitch.org   |
| Brought to you by ClueCon http://www.cluecon.com/     |
.=======================================================.


.=======================================================================================================.
|       _                            _    ____ _             ____                                       |
|      / \   _ __  _ __  _   _  __ _| |  / ___| |_   _  ___ / ___|___  _ __                             |
|     / _ \ | '_ \| '_ \| | | |/ _` | | | |   | | | | |/ _ \ |   / _ \| '_ \                            |
|    / ___ \| | | | | | | |_| | (_| | | | |___| | |_| |  __/ |__| (_) | | | |                           |
|   /_/   \_\_| |_|_| |_|\__,_|\__,_|_|  \____|_|\__,_|\___|\____\___/|_| |_|                           |
|                                                                                                       |
|    ____ _____ ____    ____             __                                                             |
|   |  _ \_   _/ ___|  / ___|___  _ __  / _| ___ _ __ ___ _ __   ___ ___                                |
|   | |_) || || |     | |   / _ \| '_ \| |_ / _ \ '__/ _ \ '_ \ / __/ _ \                               |
|   |  _ < | || |___  | |__| (_) | | | |  _|  __/ | |  __/ | | | (_|  __/                               |
|   |_| \_\|_| \____|  \____\___/|_| |_|_|  \___|_|  \___|_| |_|\___\___|                               |
|                                                                                                       |
|     ____ _             ____                                                                           |
|    / ___| |_   _  ___ / ___|___  _ __         ___ ___  _ __ ___                                       |
|   | |   | | | | |/ _ \ |   / _ \| '_ \       / __/ _ \| '_ ` _ \                                      |
|   | |___| | |_| |  __/ |__| (_) | | | |  _  | (_| (_) | | | | | |                                     |
|    \____|_|\__,_|\___|\____\___/|_| |_| (_)  \___\___/|_| |_| |_|                                     |
|                                                                                                       |
.=======================================================================================================.

Type /help <enter> to see a list of commands




[This app Best viewed at 160x60 or more..]
+OK log level  [7]
freeswitch@u2004.kozik.net>
2024-08-30 13:06:08.382598 97.50% [NOTICE] mod_signalwire.c:401 Go to https://signalwire.com to set up your Connector now! Enter connection token c6bc2107-f513-42b7-b03d-2XXXXXXXXXXXXX
2024-08-30 13:06:08.402607 97.50% [INFO] mod_signalwire.c:1125 Next SignalWire adoption check in 15 minutes
```
### Setup Freeswitch <-> Signalwire connector
When running fs_cli, every few minutes a message pops up to the [signalwire.com](https://signalwire.com) website and setup a connecor. Using the token provided by the prompt, I found the New Freeswitch Connection page under the Integrations menu and created a connection.
![image](https://github.com/user-attachments/assets/e29256fa-10e0-4824-8562-e64de8f7f9cf)

Once connected, Signalwire prompts for a phone number to associate with this connection.  This number is used when calling to the PSTN, this is what will show as the calling number, its it the number that will display on the called-party's phone.  I bought a phone number from signalwire and I am using that number. You could also enter your mobile phone for this.  
![image](https://github.com/user-attachments/assets/153f33c2-d0e6-4e8a-85fd-76d1f5c8267e)
### Setup Signalwire Phone number to Freeswtich Connector linkage
In the signalwire Phone Number menu, edit your phone number to point to your Freeswitch connector.

Phone Number Menu:
![image](https://github.com/user-attachments/assets/a702a63e-6e92-47f4-8e43-62c7f163dc75)

Edit phone number to connector to your Freeswitch integration:
![image](https://github.com/user-attachments/assets/8039b7e9-e20b-405d-93cc-d43b0e656aaa)

### Verify SIP stack
The default SIP configuration should basically be working.  From the fs_cli command line we can see the following:
```
freeswitch@u2004.kozik.net> /log 4
+OK log level 4 [4]
freeswitch@u2004.kozik.net> sofia status
                     Name          Type                                       Data      State
=================================================================================================
               signalwire       profile          sip:mod_sofia@69.243.158.102:6051      RUNNING (0) (TLS)
   signalwire::signalwire       gateway   sip:4dc289a674caa985be7f3f8cc029040b@jkozik-63eecb1e59a343e4b71567f323a71d52.sip.signalwire.com       REGED
          192.168.100.128         alias                                   internal      ALIASED
                 external       profile          sip:mod_sofia@69.243.158.102:5080      RUNNING (0)
    external::example.com       gateway                    sip:joeuser@example.com      NOREG
                 internal       profile         sip:mod_sofia@192.168.100.128:5060      RUNNING (0)
=================================================================================================
3 profiles 1 alias
```
Here's what this says:
- The gateway called signalwire::signalwire shows registered to the signalwire service.  This is the freeswitch SIP trunk.
- The interal profile is listening on port 5060 on my home LAN.  It only sees SIP clients on my home network.
- The external profile is listening on port 5080 and only listens to my WAN interface;  which by the way is NAT'd and I am using the Freeswitch default settings and it works fine.
- I don't have a domain name setup for my freeswitch, thus its name is the same as the Freeswitch IP address, 192.168.100.128 

## Firewall Settings
So far, I have not needed to set any firewall rules.  But Freeswitch does need some ports open.  This freeswitch is running Docker on my non-firewalled Ubuntu 22.04 server (as recommended in the README).  I need to open ports on my Home LAN firewall. The [Freeswitch Network->Firewall](https://developer.signalwire.com/freeswitch/FreeSWITCH-Explained/Networking/Firewall_1048908) is very detailed.

My firewall is a pfsense router, and in the Firewall->NAT menu, I setup some rules for SIP port 5080 and RTP port on 20000-20100.

![image](https://github.com/user-attachments/assets/f85a273f-99f8-4ab0-9674-8a818cf9ee64)

The rules say:
- map TCP/UDP traffic from ports 5080/5081 to the Freeswitch hosted at 192.168.100.128.
- open a block of UDP ports for RTP traffic and map them to the same Freeswitch host

Note: I already have a couple of other Voip solutions in my house and I am reluctant to open too many UDP ports for RTP. Freeswitch's default range overlaps with my existing setup.  I hope the range is ok.

Because I am using a restricted RTP port range, the rtp-start-port and rtp-end-port parameters in the Swithc.conf.xml file need to be reset. (Source: [Switch.conf.xml](https://developer.signalwire.com/freeswitch/FreeSWITCH-Explained/Configuration/Configuring-FreeSWITCH/Switch.conf.xml_9634306). 

To edit this file, I exited fs_cli, I exited the container and I edited a switch.conf.xml mounted under the conf directory.

```
jkozik@u2004:~/projects/freeswitch-cluecon-lab/conf/autoload_configs$ cd ~/projects/freeswitch-cluecon-lab
jkozik@u2004:~/projects/freeswitch-cluecon-lab$ cd conf
jkozik@u2004:~/projects/freeswitch-cluecon-lab/conf$ # This directory is mounted inside of the freeswitch container
jkozik@u2004:~/projects/freeswitch-cluecon-lab/conf$ cd autoload_configs/
jkozik@u2004:~/projects/freeswitch-cluecon-lab/conf/autoload_configs$ # Change the default RTP ports
jkozik@u2004:~/projects/freeswitch-cluecon-lab/conf/autoload_configs$ grep -E "(rtp-start-port|rtp-end-port)" switch.conf.xml
    <!-- <param name="rtp-start-port" value="16384"/> -->
    <!-- <param name="rtp-end-port" value="32768"/> -->
jkozik@u2004:~/projects/freeswitch-cluecon-lab/conf/autoload_configs$ # change them to 20000->20100
jkozik@u2004:~/projects/freeswitch-cluecon-lab/conf/autoload_configs$ vi switch.conf.xml
jkozik@u2004:~/projects/freeswitch-cluecon-lab/conf/autoload_configs$ grep -E "(rtp-start-port|rtp-end-port)" switch.conf.xml
         <param name="rtp-start-port" value="20000"/>
         <param name="rtp-end-port" value="20100"/>
jkozik@u2004:~/projects/freeswitch-cluecon-lab/conf/autoload_configs$
jkozik@u2004:~/projects/freeswitch-cluecon-lab/conf/autoload_configs$ docker exec -it freeswitch-community  bash
root@u2004:/# fs_cli
freeswitch@u2004.kozik.net> 
freeswitch@u2004.kozik.net> reloadxml
+OK [Success]
freeswitch@u2004.kozik.net> status
UP 0 years, 0 days, 17 hours, 57 minutes, 36 seconds, 785 milliseconds, 995 microseconds
FreeSWITCH (Version 1.10.12 -release-10222002881-a88d069d6fgit a88d069 2024-08-02 21:02:27Z 64bit) is ready
0 session(s) since startup
0 session(s) - peak 0, last 5min 0
0 session(s) per Sec out of max 30, peak 0, last 5min 0
1000 session(s) max
min idle cpu 0.00/96.87
Current Stack Size/Max 240K/8192K
```
## Connect SIP Clients
Freeswitch is cycling, its sip stack looks normal, and now the next step is to connect SIP phones.  I have two of them running on my home LAN:
- [ZioPer](https://www.zoiper.com/).  It is a freeware client that runs on PCs and mobile phones.  I have it running my Windows 11 laptop in on my home LAN
- [Yealink W60B](https://www.yealink.com/en/product-detail/zoom-phone-w60p). This is an IP phone plus a basestation that can support upto 8 phones.  It uses DECT wireless between the physical phones and the basestation.  The base station connect to the local network over ethernet (my case) or WIFI.  All the IP phones share the same IP address. The IP phone defaults to one of 8 SIP accounts.  The phone can select an outboud SIP account on a per call basis.  I have the phone connected to my account with Voip.ms.  I added an accout with my Freeswitch.

### Setup ZoiPer on Freeswitch
![image](https://github.com/user-attachments/assets/756f1a8f-16df-4535-a795-6c26a9cc4f60)

I didn't find documentation the freeswitch for setting up Zoiper, but I found a good writeup on [Zoiper — FusionPBX Docs documentation](https://docs.fusionpbx.com/en/latest/applications/provision/provision_manual_zoiper.html).  Here's what I did:
- I downloaded and installed the free client from [Zoiper.com](https://www.zoiper.com/)
- Open Zoiper and create an account.  Pick user 1001 and use 192.168.100.128 as the domain.  Use the password provided in the .env file when freeswitch was installed.
![image](https://github.com/user-attachments/assets/ec54e9b1-87ec-407a-9c62-945ecdc62d72)

- It will prompt for a provider, enter the freeswitch IP address 192.168.100.128:5060 with port number
![image](https://github.com/user-attachments/assets/59b7ec66-5ca6-4119-b3a6-037437c
a692b)

![image](https://github.com/user-attachments/assets/6326a8b5-d057-43f7-b22a-c620c819dfe6)

- Skip outbound proxy
  
- Verify that Zoiper is able to register
![image](https://github.com/user-attachments/assets/abfe6448-ce64-4abb-8db4-1b6b93874f8d)

- When finished you should see this:
![image](https://github.com/user-attachments/assets/dcb87f70-25be-4d31-8b23-47294e3de01e)

- Verify that freeswitch shows the registration
```
jkozik@u2004:~$ docker exec -it freeswitch-community  bash
root@u2004:/# fs_cli

Type /help <enter> to see a list of commands




[This app Best viewed at 160x60 or more..]
+OK log level  [7]
freeswitch@u2004.kozik.net> show registrations
reg_user,realm,token,url,expires,network_ip,network_port,network_proto,hostname,metadata
1001,192.168.100.128,M_x25dCeGAnXuDs_J-zlPQ..,sofia/internal/sip:1001@192.168.100.106:61970;rinstance=4a2b9ccbed8e5dc4;transport=tcp,1725046112,192.168.100.106,60065,tcp,u2004.kozik.net,

freeswitch@u2004.kozik.net> /log 4
+OK log level 4 [4]
freeswitch@u2004.kozik.net>
```
#### Zoiper / Freeswitch troubleshooting
This installation comes preconfigured with 20 preconfigured 4 digit extensions in its directory with user ids 1000-1019, each sharing the same default password.  If the registration doesnt work, it is possible that the user id was entered wrong or the password was not setup correctly.

One tool to try is sngrep.  Install it on the host and run it, not in the freeswitch docker container, and see if you can see the fault. Here's what a Zoiper registration looks like. 
![image](https://github.com/user-attachments/assets/0884ac34-ff94-4a3c-ad2b-e588cf0e65c5)

The tool lets you drill down to see the details of each of the messages.

Further, the freeswitch fs_cli tool has an option to trace SIP messages.  To trace messages with a Voip client, select the internal profile.  See editted example trace of a registration
```
jkozik@u2004:~/projects/freeswitch-cluecon-lab$ docker exec -it freeswitch-community  bash
root@u2004:/# fs_cli
Type /help <enter> to see a list of commands

[This app Best viewed at 160x60 or more..]
+OK log level  [7]

freeswitch@u2004.kozik.net> /log 4
+OK log level 4 [4]

freeswitch@u2004.kozik.net> sofia profile internal siptrace on
Enabled sip debugging on internal
recv 875 bytes from tcp/[192.168.100.106]:60065 at 19:39:37.410129:
------------------------------------------------------------------------
REGISTER sip:192.168.100.128:5060;transport=TCP SIP/2.0
Via: SIP/2.0/TCP 192.168.100.106:61970;branch=z9hG4bK-524287-1---9f009a793c824a9c;rport
Max-Forwards: 70
Contact: <sip:1001@192.168.100.106:61970;rinstance=e52c63865c034b38;transport=tcp>;expires=0
To: <sip:1001@192.168.100.128:5060;transport=TCP>
From: <sip:1001@192.168.100.128:5060;transport=TCP>;tag=1b25b75b
Call-ID: 4tmRZhmm70Bu3Bcvd-S3zQ..
CSeq: 3 REGISTER
Allow: INVITE, ACK, CANCEL, BYE, NOTIFY, REFER, MESSAGE, OPTIONS, INFO, SUBSCRIBE
User-Agent: Z 5.5.12 v2.10.18.2
Authorization: Digest username="1001",realm="192.168.100.128",nonce="fcbfacd5-a32f-4dce-9186-fb87bb194c4f",uri="sip:192.168.100.128:5060;transport=TCP",response="e7a76b4f70f322ab6331e83f44e874fb",cnonce="7345dcad0b8d09da114aaa601bfaa771",nc=00000002,qop=auth,algorithm=MD5
Allow-Events: presence, kpml, talk
Content-Length: 0


send 592 bytes to tcp/[192.168.100.106]:60065 at 19:39:37.423456:
------------------------------------------------------------------------
SIP/2.0 200 OK
Via: SIP/2.0/TCP 192.168.100.106:61970;branch=z9hG4bK-524287-1---9f009a793c824a9c;rport=60065
From: <sip:1001@192.168.100.128:5060;transport=TCP>;tag=1b25b75b
To: <sip:1001@192.168.100.128:5060;transport=TCP>;tag=Z3K02XQ60HgSc
Call-ID: 4tmRZhmm70Bu3Bcvd-S3zQ..
CSeq: 3 REGISTER
Date: Fri, 30 Aug 2024 19:39:37 GMT
User-Agent: FreeSWITCH-mod_sofia/1.10.12-release-10222002881-a88d069d6f+git~20240802T210227Z~a88d069d6f~64bit
Allow: INVITE, ACK, BYE, CANCEL, OPTIONS, MESSAGE, INFO, UPDATE, REGISTER, REFER, NOTIFY
Supported: timer, path, replaces
Content-Length: 0
freeswitch@u2004.kozik.net>
```
### Setup Yealink IP Phone
![image](https://github.com/user-attachments/assets/54df9a30-16a3-4338-9a00-9dbef5bd9a4b)

The ATA box that comes with my IP phone has a web portal on it.  Under the Account tab, I selected an unused account, Account 5 and configured extension 1002 to the server host 192.168.100.128 using the same password as the .env.  See screen capture below.
![image](https://github.com/user-attachments/assets/310e5144-19f5-4d0b-964e-a0e08f5af62d)
I pressed confirm and the top of the page indicated Registered.  

Verify that freeswitch sees the second registration:
```
freeswitch@u2004.kozik.net> show registrations
reg_user,realm,token,url,expires,network_ip,network_port,network_proto,hostname,metadata
1002,192.168.100.128,4_1905926168@192.168.100.94,sofia/internal/sip:1002@192.168.100.94:5060;transport=TCP,1725052903,192.168.100.94,12698,tcp,u2004.kozik.net,
1001,192.168.100.128,3KBvk-7T-uLuwrJLb0FIjg..,sofia/internal/sip:1001@192.168.100.106:61970;rinstance=2572aeda33b3c223;transport=tcp,1725050252,192.168.100.106,60065,tcp,u2004.kozik.net,

2 total.

freeswitch@u2004.kozik.net>
```
## Verify SIP Clients on Home LAN
### Verify that Zoiper <-> Yealink can call each other
At the Zoiper client, enter the phone number 1002. You should hear audible ring from the Zoiper client an the Yealink phone should ring.  Answer the call on the Yealink. Verify the talk path, both ways.  Hangup.

Repeat for Yealink IP phone calling the Zoiper softphone. This should work. 

#### Yealink to Zoiper 1002->1001 Dialplan trace
For background, it is useful to study what is going on here.  When one extension calls another, this is an internal call and it uses the default dial plan. No interactions with Signalwire happen.  

It is useful to turn on full debug level logging to see all the steps freeswitch goes through to setup a call.  I would like to however focus on a subset on what the Dialplan log shows.  It turns out to be very useful for troubleshooting.  

First set the log and logfilter rules as follows then place a call from 1002 (Yealink) to 1001 (Zoiper) and see the following Dialplan log traces
```
freeswitch@u2004.kozik.net> /log 4
+OK log level 4 [4]

freeswitch@u2004.kozik.net> /logfilter Dialplan
Logfilter enabled

Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [public->unloop] continue=false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (PASS) [unloop] ${unroll_loops}(true) =~ /^true$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [unloop] ${sip_looped_call}() =~ /^true$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [public->outside_call] continue=true
Dialplan: sofia/internal/1002@192.168.100.128:5060 Absolute Condition [outside_call]
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action set(outside_call=true)
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action export(RFC2822_DATE=${strftime(%a, %d %b %Y %T %z)})
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [public->call_debug] continue=true
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [call_debug] ${call_debug}(false) =~ /^true$/ break=never
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [public->public_extensions] continue=false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (PASS) [public_extensions] destination_number(1001) =~ /^(10[01][0-9])$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action transfer(1001 XML default)
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [default->unloop] continue=false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (PASS) [unloop] ${unroll_loops}(true) =~ /^true$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [unloop] ${sip_looped_call}() =~ /^true$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [default->tod_example] continue=true
Dialplan: sofia/internal/1002@192.168.100.128:5060 Date/TimeMatch (FAIL) [tod_example] break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [default->holiday_example] continue=true
Dialplan: sofia/internal/1002@192.168.100.128:5060 Date/TimeMatch (FAIL) [holiday_example] break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [default->global-intercept] continue=false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [global-intercept] destination_number(1001) =~ /^886$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [default->group-intercept] continue=false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [group-intercept] destination_number(1001) =~ /^\*8$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [default->intercept-ext] continue=false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [intercept-ext] destination_number(1001) =~ /^\*\*(\d+)$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [default->redial] continue=false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [redial] destination_number(1001) =~ /^(redial|870)$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [default->global] continue=true
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [global] ${call_debug}(false) =~ /^true$/ break=never
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [global] ${default_password}(Klick?123) =~ /^1234$/ break=never
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [global] ${rtp_has_crypto}() =~ /^(AEAD_AES_256_GCM_8|AEAD_AES_128_GCM_8|AES_CM_256_HMAC_SHA1_80|AES_CM_192_HMAC_SHA1_80|AES_CM_128_HMAC_SHA1_80|AES_CM_256_HMAC_SHA1_32|AES_CM_192_HMAC_SHA1_32|AES_CM_128_HMAC_SHA1_32|AES_CM_128_NULL_AUTH)$/ break=never
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [global] ${endpoint_disposition}(RECEIVED) =~ /^(DELAYED NEGOTIATION)/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [default->snom-demo-2] continue=false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [snom-demo-2] destination_number(1001) =~ /^9001$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [default->snom-demo-1] continue=false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [snom-demo-1] destination_number(1001) =~ /^9000$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [default->eavesdrop] continue=false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [eavesdrop] destination_number(1001) =~ /^88(\d{4})$|^\*0(.*)$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [default->eavesdrop] continue=false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [eavesdrop] destination_number(1001) =~ /^779$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [default->call_return] continue=false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [call_return] destination_number(1001) =~ /^\*69$|^869$|^lcr$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [default->del-group] continue=false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [del-group] destination_number(1001) =~ /^80(\d{2})$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [default->add-group] continue=false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [add-group] destination_number(1001) =~ /^81(\d{2})$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [default->call-group-simo] continue=false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [call-group-simo] destination_number(1001) =~ /^82(\d{2})$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [default->call-group-order] continue=false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [call-group-order] destination_number(1001) =~ /^83(\d{2})$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [default->extension-intercom] continue=false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [extension-intercom] destination_number(1001) =~ /^8(10[01][0-9])$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [default->Local_Extension] continue=false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (PASS) [Local_Extension] destination_number(1001) =~ /^(10[01][0-9])$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action export(dialed_extension=1001)
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action bind_meta_app(1 b s execute_extension::dx XML features)
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action bind_meta_app(2 b s record_session::/var/lib/freeswitch/recordings/${caller_id_number}.${strftime(%Y-%m-%d-%H-%M-%S)}.wav)
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action bind_meta_app(3 b s execute_extension::cf XML features)
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action bind_meta_app(4 b s execute_extension::att_xfer XML features)
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action set(ringback=${us-ring})
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action set(transfer_ringback=local_stream://moh)
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action set(call_timeout=30)
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action set(hangup_after_bridge=true)
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action set(continue_on_fail=true)
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action hash(insert/${domain_name}-call_return/${dialed_extension}/${caller_id_number})
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action hash(insert/${domain_name}-last_dial_ext/${dialed_extension}/${uuid})
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action set(called_party_callgroup=${user_data(${dialed_extension}@${domain_name} var callgroup)})
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action hash(insert/${domain_name}-last_dial_ext/${called_party_callgroup}/${uuid})
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action hash(insert/${domain_name}-last_dial_ext/global/${uuid})
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action hash(insert/${domain_name}-last_dial/${called_party_callgroup}/${uuid})
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action bridge(user/${dialed_extension}@${domain_name})
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action answer()
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action sleep(1000)
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action bridge(loopback/app=voicemail:default ${domain_name} ${dialed_extension})
```
A few comments on above:
- Each extension is defined in a ../conf/directory/default/100[0-9].xml configuration file.  In that file the local extensions are assigned to the  name="user_context" value="default" user context.  This means that dialed digits will be processed by the default dial plan, located at ../conf/dialplan/default.xml and ../conf/dialplan/dialplan directory.
- In this log trace, the calling party 1002 dialed a destination_number 1001
- In the default.xml file are a series of extensions that execute an action if a condition is true.  Each unique extension is named, shown in [...]
- The above trace show dozens of extensions, most of which use a regular expression string to test against the destination_number
- The first match for the 1001 is the extension `Local_Extension` regular expression `^(10[01][0-9])$`
- Once the condition is matched a series of actions are performed
- The key action at the end of the extention object are the following
```
        <action application="bridge" data="user/${dialed_extension}@${domain_name}"/>
        <action application="answer"/>
```
The freeswitch bridges the incoming call to the dialed_extension and answers the call. Ta-Dah !

It is useful to review the default.xml file.  Is is full of example extensions that hightlight the power of the freeswitch feature set. 

#### Yealink to Zoiper 1002->1001 SIP trace
For completeness, here's one half of the SIP call trace for the call from 1002 to 1001.  Note, this was run from the host root login outside of the freeswitch docker container.  
![image](https://github.com/user-attachments/assets/0a655103-0cd7-45ca-8e3d-206b70176f3a)

## Configure Signalwire SIP trunk
The signalware connector creates a gateway sip profile automatically, but there needs to be a linkage between the SIP trunk and the dialing plan.  The [Signalwire Dialplan example](https://developer.signalwire.com/freeswitch/FreeSWITCH-Explained/Modules/mod_signalwire_19595544/#3-dialplan-sample) gives boilerplate that can be added to the dialing plan.
### Dialplan for Signalwire incoming call
The incoming call is routed to freeswitch from signalwire with a destination_number equal to the phone number bought and registered from the signalwire portal.  In my case it's +1630387XXXX.  My default the call routes to freeswitch and gives a busy signal.  See the first few lines of default fs_cli trace:
```
jkozik@u2004:~$ docker exec -it freeswitch-community  bash
root@u2004:/# fs_cli
Type /help <enter> to see a list of commands
[This app Best viewed at 160x60 or more..]
+OK log level  [7]
2024-08-31 20:57:32.802859 95.47% [INFO] mod_dialplan_xml.c:639 Processing +1630215XXXX <+1630215XXXX>->+1630387XXXX in context default
2024-08-31 20:57:32.862560 95.47% [NOTICE] switch_core_state_machine.c:382 sofia/signalwire/+1630215XXXX@sip.signalwire.com has executed the last dialplan instruction, hanging up.
```
Note that it shows the incoming call from my mobile phone to my signalwire number.  But it fails with a busy signal.  Note:  the incoming call was processed in the `default` dialplan.  

I copy/pasted and edited the following default extension:
```
/home/jkozik/projects/freeswitch-cluecon-lab/conf/dialplan/default
jkozik@u2004:~/projects/freeswitch-cluecon-lab/conf/dialplan/default$ cat 01_abSignalWireIncomingFromPSTN.xml
<include>
    <extension name="SignalWire INTEGRATIONS incoming call">
      <condition field="destination_number" expression="^(\+16303874223)$"> <!-- the number you assigned in your dashboard -->
        <action application="bridge" data="user/1001"/>
      </condition>
    </extension>
</include>
```
This new file gets automatically included into the default.xml dialplan.
Then going back to the fs_cli session, reload the parameters (reloadxml) and observe the dialplan trace (excerpt).
```
2024-08-31 21:52:46.422594 95.30% [INFO] mod_dialplan_xml.c:639 Processing +16302153142 <+16302153142>->+16303874223 in context default
. . .
Dialplan: sofia/signalwire/+16302153142@sip.signalwire.com parsing [default->hold_music] continue=false
Dialplan: sofia/signalwire/+16302153142@sip.signalwire.com Regex (FAIL) [hold_music] destination_number(+16303874223) =~ /^9664$/ break=on-false
Dialplan: sofia/signalwire/+16302153142@sip.signalwire.com parsing [default->laugh break] continue=false
Dialplan: sofia/signalwire/+16302153142@sip.signalwire.com Regex (FAIL) [laugh break] destination_number(+16303874223) =~ /^9386$/ break=on-false
Dialplan: sofia/signalwire/+16302153142@sip.signalwire.com parsing [default->101] continue=false
Dialplan: sofia/signalwire/+16302153142@sip.signalwire.com Regex (FAIL) [101] destination_number(+16303874223) =~ /^101$/ break=on-false
Dialplan: sofia/signalwire/+16302153142@sip.signalwire.com parsing [default->pizza_demo] continue=false
Dialplan: sofia/signalwire/+16302153142@sip.signalwire.com Regex (FAIL) [pizza_demo] destination_number(+16303874223) =~ /^(pizza|74992)$/ break=on-false
Dialplan: sofia/signalwire/+16302153142@sip.signalwire.com parsing [default->Talking Clock Time] continue=false
Dialplan: sofia/signalwire/+16302153142@sip.signalwire.com Regex (FAIL) [Talking Clock Time] destination_number(+16303874223) =~ /^9170$/ break=on-false
Dialplan: sofia/signalwire/+16302153142@sip.signalwire.com parsing [default->Talking Clock Date] continue=false
Dialplan: sofia/signalwire/+16302153142@sip.signalwire.com Regex (FAIL) [Talking Clock Date] destination_number(+16303874223) =~ /^9171$/ break=on-false
Dialplan: sofia/signalwire/+16302153142@sip.signalwire.com parsing [default->Talking Clock Date and Time] continue=false
Dialplan: sofia/signalwire/+16302153142@sip.signalwire.com Regex (FAIL) [Talking Clock Date and Time] destination_number(+16303874223) =~ /^9172$/ break=on-false
Dialplan: sofia/signalwire/+16302153142@sip.signalwire.com parsing [default->SignalWire INTEGRATIONS incoming call] continue=false
Dialplan: sofia/signalwire/+16302153142@sip.signalwire.com Regex (PASS) [SignalWire INTEGRATIONS incoming call] destination_number(+16303874223) =~ /^(\+16303874223)$/ break=on-false
Dialplan: sofia/signalwire/+16302153142@sip.signalwire.com Action bridge(user/1001)
```
The call from my mobile phone to my signalwire matches the regular expression `^(\+16303874223)$` and gets bridged to extension 1001 which is my Zoiper client.  
### Dialplan for Signalwire outgoing call
In this case my home Yealink phone is dialing my mobile phone (630215-XXXX.  My mobile phone sees the caller id as my signalwire phone number (630387-XXXX). Likewise, the [Signalwire Dialplan example](https://developer.signalwire.com/freeswitch/FreeSWITCH-Explained/Modules/mod_signalwire_19595544/#3-dialplan-sample) gives boilerplate that can be added to the outgoing dialing plan.

First, try to make a call and look at the trace and confirm that the call uses the outgoing / public dialing plan.
```
2024-08-31 22:04:20.809944 95.20% [INFO] mod_dialplan_xml.c:639 Processing yealink <1002>->16302153142 in context public
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [public->unloop] continue=false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (PASS) [unloop] ${unroll_loops}(true) =~ /^true$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [unloop] ${sip_looped_call}() =~ /^true$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [public->outside_call] continue=true
```
The above is excerpt of the trace from an outgoing call attempt.  It gave a busy signal because no signalwire dialing plan extension was configured.  Different from the incoming, I need to add an extension to the public dialing plan.
```
jkozik@u2004:~/projects/freeswitch-cluecon-lab/conf/dialplan/public$ pwd
/home/jkozik/projects/freeswitch-cluecon-lab/conf/dialplan/public

jkozik@u2004:~/projects/freeswitch-cluecon-lab/conf/dialplan/public$ cat 01_aaSignalWirePSTN.xml
<include>
   <extension name="signalwire INTEGRATIONS outgoing call">
    <condition field="destination_number" expression="^(\d{11})$">
      <action application="set" data="effective_caller_id_number=${outbound_caller_id_number}"/>
      <action application="set" data="effective_caller_id_name=${outbound_caller_id_name}"/>
      <action application="answer"/>
      <action application="bridge" data="sofia/gateway/signalwire/+$1"/>
    </condition>
   </extension>
</include>
jkozik@u2004:~/projects/freeswitch-cluecon-lab/conf/dialplan/public$
```
Note:
- The extension xml file is in the public folder.  It will automatically get included on a reloadxml command from the fs_cli prompt
- The signalwire SIP trunk is configured as a SIP profile of type gateway, named signalwire.  See sofia status dump.
- The outgoing call has 10 digits:  1630215XXXX. The regular expression `^(\d{11})$' is matched and its value is put into $1
-  The SIP trunk requires a + sign.  See the bridge data parameter

Then, going back to the fs_cli prompt, run reloadxml and try an outgoing test call from the Yealink IP phone (1002) to my mobile phone (1630215-XXXX) and verify that the incoming call shows a caller id of my signalwire phone number (1630387-XXXX).
```
freeswitch@u2004.kozik.net> reloadxml
+OK [Success]

2024-08-31 22:21:20.902617 95.33% [INFO] mod_dialplan_xml.c:639 Processing yealink <1002>->16302153142 in context public
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [public->unloop] continue=false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (PASS) [unloop] ${unroll_loops}(true) =~ /^true$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [unloop] ${sip_looped_call}() =~ /^true$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [public->outside_call] continue=true
Dialplan: sofia/internal/1002@192.168.100.128:5060 Absolute Condition [outside_call]
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action set(outside_call=true)
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action export(RFC2822_DATE=${strftime(%a, %d %b %Y %T %z)})
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [public->call_debug] continue=true
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [call_debug] ${call_debug}(false) =~ /^true$/ break=never
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [public->public_extensions] continue=false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [public_extensions] destination_number(16302153142) =~ /^(10[01][0-9])$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [public->public_conference_extensions] continue=false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [public_conference_extensions] destination_number(16302153142) =~ /^(3[5-8][01][0-9])$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [public->public_did] continue=false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (FAIL) [public_did] destination_number(16302153142) =~ /^(5551212)$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 parsing [public->signalwire INTEGRATIONS outgoing call] continue=false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Regex (PASS) [signalwire INTEGRATIONS outgoing call] destination_number(16302153142) =~ /^(\d{11})$/ break=on-false
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action set(effective_caller_id_number=${outbound_caller_id_number})
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action set(effective_caller_id_name=${outbound_caller_id_name})
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action answer()
Dialplan: sofia/internal/1002@192.168.100.128:5060 Action bridge(sofia/gateway/signalwire/+16302153142)
```
The outgoing call worked. Skim the dialplan log trace and see how outgoing calls are parsed. 
And for reference the outgoing call over the signalwire SIP trunk generated the call flow below:
![image](https://github.com/user-attachments/assets/8df48152-b8be-4673-a8e7-997ef430f2fd)
## Summary
To get the freeswitch-cluecon-lab repository to work on my home LAN with Signalwire I had to modify the following:
- **docker-compose.yml**. I needed to tweek it to do `--network host` and enable IPv6
- **switch.conf.xml**. I needed to restrict the rtp-start/end-port range to fit my home network setup
- **dialplan/default**.  I needed to add an extension for signalwire incoming from PSTN calls. I mapped the calls to my SIP extension 1001 (Zoiper client)
- **dialplan/public**.  I needed to add an extension for signalwire outgoing to PSTN calls. This was configured to show my signalwire phone number as my calling party id
- **pfsense firewall**.  On my home LAN I needed to map incoming 5080 to my home LAN's freeswitch and open a range of UDP ports and map them to my freeswitch. 
## References
- [Connecting FreeSWITCH to SignalWire CLOUD](https://signalwire.com/blogs/freeswitch/mod-signalwire?utm_source=marketo&utm_medium=email&utm_campaign=AI-OH-082024&utm_content=button&mkt_tok=MjYyLUhHUi0zMTEAAAGVFVRWyyjq-0Ib-rw6vIw4Jn0m_Lnc3t1z7bl2PrdBRG2ce-BL0KpVmDpTsl-YCg6-8bTUfIt1BtyGfPkrV-BwNYe3DU-fM6F7KRUGXRs)
- [signalwire/freeswitch](https://github.com/signalwire/freeswitch/tree/master)
    - [freeswitch-cluecon-lab](https://github.com/signalwire/freeswitch-cluecon-lab)
    - [Making and Receiving Phone Calls](https://developer.signalwire.com/guides/voice/making-and-receiving-phone-calls/)
    - [Setting Up a SIP Endpoint](https://developer.signalwire.com/guides/set-up-a-signalwire-phone-number-with-a-sip-endpoint)
    - [Installing FreeSWITCH or FreeSWITCH Advantage](https://developer.signalwire.com/guides/installing-freeswitch-or-freeswitch-advantage)
- [Running a PBX(FreeSwitch) in Docker as a house intercom](https://medium.com/@sertys/running-a-pbx-freeswitch-in-docker-as-a-house-intercom-3f3e4c6ef6c9)
    - [sertys3/freeswitch-docker](https://github.com/sertys3/freeswitch-docker)
- [dheaps/freeswitch](https://hub.docker.com/r/dheaps/freeswitch)
- [drachtio/drachtio-freeswitch-mrf](https://hub.docker.com/r/drachtio/drachtio-freeswitch-mrf)
- [bettervoice/freeswitch-container](https://hub.docker.com/r/bettervoice/freeswitch-container/)
- [signalwire/freeswitch/docker](https://github.com/signalwire/freeswitch/tree/master/docker)
    - [signalwire/freeswitch/docker/base_image](https://github.com/signalwire/freeswitch/blob/ma)ster/docker/base_image/README.md)
- [Omid-Mohajerani/freeswitch](https://github.com/Omid-Mohajerani/freeswitch)
    - [FreeSWITCH 1.10 installation guide from source on debian 11](https://github.com/Omid-Mohajerani/freeswitch/wiki/FreeSWITCH-1.10-installation-guide-from-source-on-debian-11#post-installation)
    - [Learn FreeSWITCH (Part6) - Sip Profiles, Directory and Dialplan](https://www.youtube.com/watch?v=nSj1htnz4vE)
    - [Learn FreeSWITCH part 7 How to add SIP Trunk ( Gateway)](https://github.com/Omid-Mohajerani/freeswitch/wiki/Learn-FreeSWITCH-part-7----How-to-add-SIP-Trunk-(-Gateway))
 - [SignalWire with Fred - SignalWire CLOUD | FreeSWITCH Connector](https://www.youtube.com/watch?v=MMktpp_PdqI)
 - [Freeswitch Explained](https://developer.signalwire.com/freeswitch/FreeSWITCH-Explained/)
    - [XML Dialplan](https://developer.signalwire.com/freeswitch/FreeSWITCH-Explained/Dialplan/XML-Dialplan/]
    - [Dialplan sample](https://developer.signalwire.com/freeswitch/FreeSWITCH-Explained/Modules/mod_signalwire_19595544/)
    - [Switch.conf.xml](https://developer.signalwire.com/freeswitch/FreeSWITCH-Explained/Configuration/Configuring-FreeSWITCH/Switch.conf.xml_9634306)
    - [Command Line Interface (fs_cli)](https://developer.signalwire.com/freeswitch/FreeSWITCH-Explained/Client-and-Developer-Interfaces/1048948/)
    - -[mod_commands](https://developer.signalwire.com/freeswitch/FreeSWITCH-Explained/Modules/mod_commands_1966741/)
- [Freeswitch Port Range](https://wiki.kolmisoft.com/index.php/Freeswitch_Port_Range)
- 
