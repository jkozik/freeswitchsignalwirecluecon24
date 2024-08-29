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
jkozik@u2004:~/projects/freeswitch-cluecon-lab$
```




