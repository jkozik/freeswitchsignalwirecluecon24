# Freeswitch Setup using Cluecon 2024 Lunch and Learn
I attended the [Cluecon 2024](https://www.cluecon.com/) conference this year.  It was my 5th time.  The conference was excellent.

As part of the Wednesday Lunch and Learn with Luca Pradovera, he ran through a Freeswitch + Signalwire demo. It was a compelling demo that should how easy it is to add Signalwire to Freeswitch, and by extension how easy it was to add Signalwire's AI features.  The repository for the demo, [freeswitch-cluecon-lab](https://github.com/signalwire/freeswitch-cluecon-lab) contained everything he used. So I decided I wanted to re-create the demo on my home LAN.

Luca indeed made it look easy, but when I tried it, I had some learning curve issues.  I document my experience here to help others who are new to setting up Freeswitch+Signalwire in a container environment.  Thanks to Luca, Brian and Jon for their help.

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





