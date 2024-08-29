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
