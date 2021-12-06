# Docker in WSL2 Walk-through

This is a complete guide to install Docker in WSL2 without using docker-desktop. This was motivated due the fact that Docker is going to start charging for the use of Docker-Desktop for enterprise, so with this guide we'll have everything set up for working with docker.

**Disclaimer**: At the date of this guide (30-11-2021), there are multiple flaws in WSL2 that are going to need a few workarounds (The NAT instead of the bridge connection), that maybe are not needed in newer versions.

## Prerequisites

### Check your Windows 10 version
Before start we have to check our windows 10 version (at the moment of writing this guide the version is 21H2) and be sure that we have the latest version (Windows 11 already has support for WSL2, so it should work).

If you are not in this version, you should run the windows update to get the latest version.

### Install windows features

After that we'll have to activate the windows subsystems for linux and the support for virtualization for Hyper-V. To do this you'll have to go to Start -> Type: Turn Windows feature on or off -> Select it.

Now we have to mark the following:
 - Hyper-V
 - Windows Hypervisor Platform
 - Windows Subsystem for Linux

After that press Ok, and they'll be installed in your computer, and you'll have to reboot.

Now we should be ready to start.

# 1. Install WSL2 Linux Distribution

Open a Powershell terminal and execute the following command:

> wsl --list --online

If everything was set up correctly it should show something similar to this:

```
The following is a list of valid distributions that can be installed.
Install using 'wsl --install -d <Distro>'.

NAME            FRIENDLY NAME
Ubuntu          Ubuntu
Debian          Debian GNU/Linux
kali-linux      Kali Linux Rolling
openSUSE-42     openSUSE Leap 42
SLES-12         SUSE Linux Enterprise Server v12
Ubuntu-16.04    Ubuntu 16.04 LTS
Ubuntu-18.04    Ubuntu 18.04 LTS
Ubuntu-20.04    Ubuntu 20.04 LTS
```

Then we'll execute the following:

> wsl --set-default-version 2

This is just in case to be sure that we'll use the WSL2

After that we'll install the linux distribution:

> wsl --install -d Ubuntu

I personally recommend Ubuntu, because it's easier to start with if you never have worked with Linux before (There is also more documentation to solve problems for this distribution).

After this launch the Ubuntu terminal: Go to Start -> Type: Ubuntu -> Select it.

Now you'll have to set your username and password for Linux.
After this is done, we are ready to start working and configuring the Linux environment.

# 2. Tweaking useful things in WSL2 Ubuntu

You can skip this section, this is entirely a personal configurations that I like to have.

## Add config file to map windows drives out of /mnt
Fist we are going to change the mount point for the C:\ drive from /mnt/c to /c. To do this we'll have to create the configuration file for it. So we'll execute the following commands:

> sudo touch /etc/wsl.conf

This will create an empty configuration file.

> sudo vim /etc/wsl.conf

This will open the text editor for the file and we copy this inside:

> NOTE: If you don't know how to use the Vi text editor just follow this tips: Press _*i*_ to start the insertion mode. Then with your mouse right button click to paste the selection. Press escape (esc) to leave the insert mode, then type: :wq to save the changes and exit the editor.

```
# Enable extra metadata options by default
[automount]
enabled = true
root = /
options = "metadata,umask=22,fmask=11"
mountFsTab = false

# Enable DNS – even though these are turned on by default, we’ll specify here just to be explicit.
[network]
generateHosts = true
generateResolvConf = true
```

After adding this configuration close the linux terminal, go back to powershell and execute this command.

> wsl --shutdown

This will terminate the WSL running session. Now we'll start the Ubuntu terminal like we did before, this way the changes from the config file we'll be applied. (To be sure check that you can access the **c:\\** drive doing with the command: _**ls /c**_)

## Change the default home path

Now we'll change the default home for the user (It should be **/home/\<your-username\>**) to be **/c/Users/\<your-windows-user\>** (In my opinion this is more consistent approach in WSL than user the Linux default path).

To do this we'll edit the _**/etc/passwd**_ file, we'll use _vim_ to modify the file (you can also use nano if you are more comfortable with it):

> sudo vim /etc/passwd

We'll have to find the line that is like:

```
<your-username>:x:1000:1000:,,,:/home/<your-username>:/bin/bash
```
And we'll modify it to be like:

```
<your-username>:x:1000:1000:,,,:/c/Users/<your-windows-username>:/bin/bash
```

In this way, if your windows username is, for example jhon.doe, it should be:

```
<your-username>:x:1000:1000:,,,:/c/Users/jhon.doe:/bin/bash
```

Just in case, we'll copy all the config files in our actual HOME to the new location:
> sudo cp -r /home/\<your-username\>/* /c/Users/\<your-windows-username\>

Close the linux terminal, shut WSL from powershell (wsl --shutdown), and open it again to see the changes. You can validate it with the following command:
> echo $HOME

Validate that it points to _**/c/Users/\<your-windows-username\>**_

## Limiting WSL2 resources usage

A common problem with WSL2 is that it uses the resource management from Linux. For Linux having free RAM idle it's a waste so it likes to cache things, and when the memory it's needed it automatically frees it. This causes a problem in Windows because it run inside a VM, it starts to taking RAM in the system (You we'll see a process call **Vmmem** stating to grow in memory usage). To solve this problem we are going to limit the amount of resources that WSL can take from the host.

To do this we'll have to create a config file in the HOME of the user. To do this execute the following commands:

>cd $HOME

Just to be sure that we are in the correct path.

> sudo touch .wslconfig

We create the file

> sudo vim .wslconfig

We open the file with the editor.
We'll paste the following information inside the file:

```
[wsl2]
memory=4GB   # Limits VM memory in WSL 2 up to 4GB
processors=4 # Makes the WSL 2 VM use four virtual processors
```

Feel free to replace the values with those that fir better with your needs.

Save the values in the editor, check the content is saved in the file (you can do it with this command: _**cat .wslconfig**_)

Restart the WSL process (close the terminal, run in powershell -> wsl --shutdown, open the Linux terminal again).

This are my favorites tweaks for WSL, fell free to add yours on top if you have any.

# 3. Installing Docker in WSL2

Now that everything is set, we are going to install Docker-CLI and Docker-Engine. First let's search for updates for Linux itself, let's run the following commands:

> sudo apt update

> sudo apt upgrade

This will update your system with the last packages and updates.

It's also good to install the following package, because we are going to need some commands from it in the future:

> sudo apt install net-tools

Now you can install Docker:

> sudo apt install `docker.io`

This will install both Docker-CLI and Docker-Engine (It's the official package)

Since WSL2 does not have support for systemctl (The Linux service manager) it won't start automatically when you open a Linux terminal, so we have to tweak it to make it work.

First we'll modify the visudo file to make your current user use the dockerd command password-less. So we invoke the next command:

> sudo visudo

We scroll all the way down to the end of the file (using the arrows), an we append the following lines at the end of the file:

```
# Docker daemon specification
<your-linux-username> ALL=(ALL) NOPASSWD: /usr/bin/dockerd
```

_**Notes**_: Remember to replace \<your-linux-username\> with your current user (The one that linux prompt).

Also we'll need to add your current user to the docker group, so it's able to run docker as non-root user. To do it, run this command:

> sudo usermod -aG docker $USER

Now we'll add the following piece of logic at the end of your .bashrc file. To do this, the easiest way is to follow this steps:

> cd $HOME

Move to your home directory.

> sudo vim .bashrc

Open the file with the text editor, and add this at the end of the file:

```
# Start Docker daemon automatically when logging in if not running.
RUNNING=`ps aux | grep dockerd | grep -v grep`
if [ -z "$RUNNING" ]; then
    sudo dockerd > /dev/null 2>&1 &
    disown
fi
```

Now we can validate the installation restarting WSL (from powershell -> wsl --shutdown and opening the linux terminal again) and running the following command:

> docker run hello-world

If everything worked correctly you should see the following message:

```
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

With this the installation of docker in WSL2 is completed, but it's going to need a little bit of work to complete integrate it with windows.

**Note**: Every time you shutdown or restart the computer, you need to open the linux terminal to lunch the docker service and make it work.

# 4. Configure the network to access the containers through localhost

In the current WSL2 version there is a problem that prevents connecting to containers as is usually done with docker-desktop (using localhost).

Long story, short, the current WSL2 version uses a NAT to manage the connection with the host, so the windows firewall blocks the connections (currently it's possible if the service in the container uses IP6 protocol, but not for IP4).

The best workaround that I could make it work it's the following one.

We are going to create a powershell script to create the custom rules in the windows firewall to be able to access the containers. The script is the following:

```
$remoteport = bash.exe -c "ifconfig eth0 | grep 'inet '"
$found = $remoteport -match '\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}';

if( $found ){
  $remoteport = $matches[0];
} else{
  write-host "The Script Exited, the ip address of WSL 2 cannot be found";
  exit;
}

#[Ports]

#All the ports you want to forward separated by coma
$ports=@(80,443,1443);


#[Static ip]
#You can change the addr to your ip config to listen to a specific address
$addr='0.0.0.0';
$ports_a = $ports -join ",";


#Remove Firewall Exception Rules if exists
$rule_found = Get-NetFirewallRule -DisplayName 'WSL 2 Firewall Unlock' 2> $null;

if ($rule_found)
{
	write-host "Rule found. Removing it."
	iex "Remove-NetFireWallRule -DisplayName 'WSL 2 Firewall Unlock' ";
}

#adding Exception Rules for inbound and outbound Rules
write-host Creating Outbound rule
iex "New-NetFireWallRule -DisplayName 'WSL 2 Firewall Unlock' -Direction Outbound -LocalPort $ports_a -Action Allow -Protocol TCP" 1>$null;

write-host Creating Inbound rule
iex "New-NetFireWallRule -DisplayName 'WSL 2 Firewall Unlock' -Direction Inbound -LocalPort $ports_a -Action Allow -Protocol TCP" 1>$null;

for( $i = 0; $i -lt $ports.length; $i++ ){
  $port = $ports[$i];
  iex "netsh interface portproxy delete v4tov4 listenport=$port listenaddress=$addr" 1>$null;
  iex "netsh interface portproxy add v4tov4 listenport=$port listenaddress=$addr connectport=$port connectaddress=$remoteport" 1>$null;
}

write-host Rules set correctly.
```

Copy it to a file and rename it as you wish, with the ps1 extension, for example: wsl2-networking.ps1

The variable ports is the one that you will have to modify to add the new rules for those ports that you want to open, in this example we set the ports 80, 443 and 1433. If you need another one, add it to the list.

Lunch a powershell with administrator rights and execute the script. If you didn't see an error everything should work correctly.

Now if you create a container that has its ports bound to be accessible from the network, you should be able to access it through **localhost:\<open-port\>**


# 5. Additional configurations

## 5.1 Install Kubernetes using Minikube

Minikube is a great tool to develop using Kubernetes in our local environment to test deployments. So we are going to install it.

To do it we are going to use the official guide located in its website: https://minikube.sigs.k8s.io/docs/start/

> cd $HOME
>
> curl -LO `https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb`
>
> sudo dpkg -i minikube_latest_amd64.deb

This will install the minikube tool, so now we'll start the kubernetes cluster, we'll do it like this:

> minikube start --driver=docker --cpus=4

**Note**: We only execute the command like this the first time to create the cluster with the proper configuration (you can set the number of cpus based in your computer specs), one the cluster is created we'll just need to do the following to start the cluster again:

> minikube start

We'll have to do it every time we start our computer after a shutdown or a restart.

We can also stop the cluster with the command:

> minikube stop

To test that minikube is up and running we could execute the kubernetes dashboard, to do so we'll do the following:

> minikube dashboard

This command should open your web browser and lunch the dashboard, if that's not the case, just copy the url that the command will prompt into your web browser.

With this you should see the status of the cluster.

## 5.2 Installing and configuring kubectl to interact with minikube

Once minikube is installed if we want to run commands in our k8s cluster, minikube provide its own kubectl interface, so any command we want to run has to use minikube as a prefix, for example:

> minikube kubectl get pods

To make it shorter to type, we have 2 options, create an alias to do it shorter, or actually installing the kubectl command itself. We are going to go with the second option, because at the end of this tutorial we are going to set things to be able to use all the commands (docker, minikube, kubectl, etc) accessible from or powershell command line, to do this with kubectl, we need to have it installed, because we can not use alias to invoke this commands.

To install kubectl we are going to follow this steps.

First, we are going to install some packages that we need:

> sudo apt-get install -y apt-transport-https ca-certificates curl

After this, we'll get the gpg keys to add a new repository to download packages from:

> sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg `https://packages.cloud.google.com/apt/doc/apt-key.gpg`

Then we'll add the new repository to be used for the apt command:

> echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] `https://apt.kubernetes.io/` kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

And the we'll run an update to download the info for the new repository:

> sudo apt-get update

Now we can install kubectl using apt:

> sudo apt-get install -y kubectl

Once we have it installed, we need to do one last step to link the minikube cluster with kubectl to be able to run the commands. To do this we have to edit the .bashrc file, to export a variable to make it work.

We open the file with the text editor:

> sudo vim $HOME/.bashrc

Now we add the following line at the end of the file:

> export KUBECONFIG=/c/Users/\<your-windows-user\>/.kube/config

**Note**: In case you followed all the steps in this guide, you home directory should be similar to the example. If that's not the case just replace _/c/Users/\<your-windows-user\>_ with your home, because it's where the config file it's created by minikube.

Save and close the text editor.

Now to make it effective, just run:

> source $HOME/.bashrc

This will reload the file, and now the variable should be accessible to your current session.

To test if this has worked, make sure the minikube k8s cluster is up and running:

> minikube status

Then if you run the following command:

> kubectl cluster-info

It should prompt the same info that if you run this one:

> minikube kubectl cluster-info

If this is the case, congrats! you can run kubectl commands in your cluster!

# Bonus: Make commands accessible from powershell

At this point, everything should work properly from the WSL2 terminal, but if we try to access docker, or the kubernetes cluster from powershell we'll get the message that it can not find those commands. This makes the experience bad compared with docker-desktop, so we are going to make the experience the same that if we have docker-desktop installed.

First we are going to check if we have a powershell profile to work with. Let's run this command:

> Test-Path $PROFILE

If the response of the command is false, that means the profile does not exist, so we'll have to create it. To do it, we'll run the following command:

> New-Item –Path $Profile –Type File –Force

This will create the profile file. In case that the _Test-Path_ command returns True, we'll skip the last command, because that means we already have a profile set up.

Now we'll edit the profile, to do it, we'll open it in the notepad, to do it from the command line, we'll execute the following:

> notepad $PROFILE

This will open the profile in the text editor. Then we'll add the following lines to the end of the file:

```
function triky-docker
{
    wsl docker $args
}

function triky-minikube
{
    wsl minikube $args
}

function triky-kubectl
{
    wsl kubectl $args
}

# Alias
Set-Alias docker triky-docker
Set-Alias minikube triky-minikube
Set-Alias kubectl triky-kubectl
```

**Note**: Feel free to change the functions name, it does not matter, just be sure the new names are correctly set in the alias.

Save the changes in the text editor, and let's apply the changes to the current session with the following command:

> . $PROFILE

Now the changes should be ready in the current session, so you can test them running any of the following commands:

> docker ps
>
> minikube status
>
> kubectl --version

So now, you are ready to work with docker and kubernetes from powershell or WSL2 without using docker-desktop!