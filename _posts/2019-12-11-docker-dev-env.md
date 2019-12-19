---
layout: post
title: I set up an available-anywhere development environment, and so can you.
tags: [programming, docker]
published: true
---

Since I left my last job I haven't had a real portable development environment. It's pretty expensive to get decent performance for Scala compilation in a laptop form factor. When I saw that Lenovo had a Black Friday sale on an extraordinarily inexpensive laptop ($129!) with good build quality but anemic specs, I had an idea: why not use this cheap laptop as a thin client for a remote development environment? I have [a relatively snappy little server](https://gist.github.com/briangordon/24edc590a6f030a07a39bb7dc3f1e1d2) at home that can do all of the heavy lifting, which should mean that the laptop gets great performance without sacrificing on cost, weight, or battery life.

My Linux server somewhat conservatively runs headless Debian, which is great for a server but not really ideal for my development workflow. So the plan will be to use Docker to run Arch in a container. I'll expose the container to the outside internet so that I can access it wherever I am in the world. It would be possible to do all of this with vagrant - there's even [an archlinux box](https://app.vagrantup.com/archlinux/boxes/archlinux) - but I'm looking for something a little lighter-weight than a dedicated virtual machine. [systemd-nspawn](https://wiki.archlinux.org/index.php/Systemd-nspawn) is perhaps more fit-for-purpose when running a whole guest OS in a container, but it lacks some of the convenience features provided by Docker, especially around automation and network configuration. Besides, this is a good opportunity to get some hands-on experience with Docker. It's worth pointing out, though, that we're accepting some risk by using containers because if there are ever any forward-incompatible kernel ABI changes then our Arch guest's libc might try to make system calls that the Debian host kernel doesn't understand. 

Before we move on, let's discuss the different options available for how to remotely control our development environment from the laptop. LTSP and Thinstation don't seem to exactly align with what I'm trying to do. I know from experience that basic X forwarding over SSH won't give me anywhere near adequate performance (low latency, compatability with low-bandwidth WAN links) for comfortable use. [Xpra](https://xpra.org/) adds some tricks and H.264 video compression to make X forwarding more bearable, so that might work - it even has an HTML5 client that works over SSL, in case you only have access to a web browser. [TigerVNC/Xvnc over SSH](https://wiki.archlinux.org/index.php/TigerVNC) would probably work okay. From my research, though, it looks like NoMachine has the best technology out there for remote desktop on Linux, and fortunately the freeware version of their product [supports](https://www.nomachine.com/FR10L02842) a virtual framebuffer via an embedded X server. If FOSS is a hard requirement for you, an old version (c. 2013) of their technology was open source, and it lives on in [the X2Go project](https://wiki.x2go.org/doku.php/doc:newtox2go). From reports I see on the web, it seems that X2Go provides better performance than VNC and Xpra, so that would be my second choice. But I'm going with NoMachine.

Step zero is to make sure that Docker is installed on the host server. The version of Docker in the Debian repository is pretty old (even in Debian testing) so we'll want to get it directly from Docker's APT repo. The [official instructions](https://docs.docker.com/install/linux/docker-ce/debian/#install-using-the-repository) for doing this seem a little bit... tryhard? This is what I did instead:

```sh
sudo -i
curl -fs https://download.docker.com/linux/debian/gpg | gpg --dearmor > /etc/apt/trusted.gpg.d/docker.asc.gpg
echo "deb [arch=amd64] https://download.docker.com/linux/debian buster stable" > /etc/apt/sources.list.d/docker.list
apt-get update
apt-get install docker-ce docker-ce-cli
```

Now we need a Dockerfile specifying how to build the image for our development environment. This is what I came up with; feel free to [customize it](https://docs.docker.com/engine/reference/builder/) for your needs.

```dockerfile
FROM archlinux/base
ARG username=brian
WORKDIR /root

# The default mirrors are sometimes not synchronized. Updating the mirrorlist first is necessary for the Dockerfile to reliably run without failures.
RUN pacman -Syu --noconfirm
RUN pacman -S --noconfirm reflector
RUN reflector --verbose --fastest 5 --age 6 --save /etc/pacman.d/mirrorlist
RUN pacman -Syu --noconfirm
RUN pacman -S --noconfirm --needed man man-pages nano openssh iputils procps-ng base-devel git systemd-sysvcompat
RUN pacman -S --noconfirm --needed xfce4 xfce4-goodies pulseaudio ttf-roboto ttf-ubuntu-font-family ttf-dejavu

# Create our user *before* mounting the home directory as a volume so that it won't be owned by root.
RUN useradd --create-home --user-group --groups wheel --shell /bin/bash $username
RUN echo '%wheel ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/wheel

# Enable OpenSSH daemon with password-based authentication disabled. You can remove some of this if you don't have authorized_keys set up.
RUN systemctl enable sshd
RUN sed -i 's/UsePAM yes/UsePAM no/' /etc/ssh/sshd_config
RUN sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
COPY --chown=$username:$username authorized_keys /home/brian/.ssh/authorized_keys

# Set up key-based auth for NX, though as we'll see it doesn't actually work. You can remove this if you don't have authorized_keys set up.
RUN mkdir -p /home/brian/.nx/config
RUN cp /home/brian/.ssh/authorized_keys /home/brian/.nx/config/authorized.crt
RUN chown -R brian:brian /home/brian/.nx
RUN chmod 0600 /home/brian/.nx/config/authorized.crt

# Patch makepkg so that it will let us run it as root. 
# Evil, but not as dumb as switching to a user only for makepkg to prompt for a nonexistent sudo password to install dependencies.
RUN cp --preserve=all /usr/bin/makepkg /usr/bin/makepkg.old
RUN sed -i 's/exit $E_ROOT/#exit $E_ROOT/g' /usr/bin/makepkg

# Build/install NoMachine. The vanilla installer doesn't support Arch but there's an AUR package which patches the Fedora support to work.
# The installer script checks /proc/1 to see if we're running systemd so if we want it to install the `.service` file we have to rename the shell.
# You can remove the DefaultDesktopCommand line if NoMachine autodetects your desktop environment. See https://www.nomachine.com/AR10K00725
RUN git clone https://aur.archlinux.org/nomachine.git
WORKDIR nomachine
RUN makepkg -sric --noconfirm
RUN cp /bin/bash systemd-sh
RUN sed -i 's_#!/bin/bash_#!/root/nomachine/systemd-sh_' /usr/NX/scripts/setup/nxserver
RUN /usr/NX/scripts/setup/nxserver --enableservices
RUN sed -i 's_#!/root/nomachine/systemd-sh_#!/bin/bash_' /usr/NX/scripts/setup/nxserver
RUN sed -i 's_DefaultDesktopCommand "/etc/X11/Xsession default"_DefaultDesktopCommand "/usr/bin/dbus-launch /usr/bin/startxfce4"_' /usr/NX/etc/node.cfg
RUN sed -i 's/EnableUPnP NX/#EnableUPnP NX/' /usr/NX/etc/server.cfg
RUN echo "UDPPort 4000" >> /usr/NX/etc/server.cfg
RUN echo "CreateDisplay 1" >> /usr/NX/etc/server.cfg
RUN echo "DisplayOwner \"$username\"" >> /usr/NX/etc/server.cfg
RUN echo "#EnableNXClientAuthentication 1" >> /usr/NX/etc/server.cfg
RUN echo "#AcceptedAuthenticationMethods NX-private-key" >> /usr/NX/etc/server.cfg
WORKDIR /root
RUN rm -rf nomachine

# Build/install yay, an AUR helper. You can remove this section if you use a different helper or prefer to run makepkg manually.
RUN git clone https://aur.archlinux.org/yay.git
WORKDIR yay
RUN makepkg -sric --noconfirm
WORKDIR /root
RUN rm -rf yay

# Put makepkg back the way it was.
RUN mv -f /usr/bin/makepkg.old /usr/bin/makepkg

# Since we're running systemd as our PID1 we need Docker to send it the SIGRTMIN+3 stop signal that systemd expects rather than the default SIGTERM.
STOPSIGNAL RTMIN+3

VOLUME /home/$username
CMD [ "/sbin/init" ]
```

Note that we're using systemd as our entry point. Typically you would not have an init system running in a container at all. It conventionally suffices to set up the filesystem and start your containerized application as PID1 directly. In our case, however, it would be nice to have systemd running so that we can install multiple services and start/stop them with systemctl. We want a full operating system, not just a single application, after all. Running systemd also helps us by reaping zombie processes which might otherwise be ignored by an application-specific entry point.

Put our Dockerfile in a directory and build an image from it. Change `brian` to your own username:

```sh
sudo -i
mkdir dev
cp ~brian/.ssh/authorized_keys .
nano Dockerfile
docker build -t dev --build-arg username=brian .
```

Now create a new container and start it up in the background. We need [a couple of extra options](https://developers.redhat.com/blog/2016/09/13/running-systemd-in-a-non-privileged-container/) so that systemd can run as PID1 in a non-privileged container. The `SYS_PTRACE` capability seems to be needed for NoMachine to start up its embedded X server. `SYS_RESOURCE` is useful and relatively harmless, and stops sudo from complaining at us when it tries to call `setrlimit`.

```sh
docker run -d -p 2222:22 -p 4000:4000 --name dev --hostname dev --cap-add=SYS_PTRACE --cap-add=SYS_RESOURCE --restart always --tmpfs /tmp --tmpfs /run -v /sys/fs/cgroup:/sys/fs/cgroup:ro dev
```

For now we can start a new root shell in the container so that we can set a password for our user. It's a good idea to do this interactively so that your password isn't recorded into `docker history`.

```sh
docker exec -it dev bash
passwd brian
exit
```

I had one additional step, which was to update my firewall configuration on the host server to permit IPv4 packets to pass through to the container via the Docker bridge network device.

```sh
sudo -i
cat > /etc/ufw/applications.d/nxserver <<-ENDOFMESSAGE
	[nxserver]
	title=NX protocol server
	description=Part of NoMachine, a remote desktop solution.
	ports=4000
ENDOFMESSAGE
ufw app update nxserver
ufw allow from 192.168.0.0/16 to 0.0.0.0/0 app nxserver
ufw limit log proto tcp from 192.168.0.0/16 to 0.0.0.0/0 port 2222
```
<br />
### Operation

In order to remote into your server, download and install the freeware [NoMachine Enterprise Client](https://www.nomachine.com/product&p=NoMachine%20Enterprise%20Client). Create a new connection, selecting the NX protocol (the freeware version of the server software doesn't support SSH) and entering the relevant hostname or IP address. You can select "Use UDP communication for multimedia data" if you change the port to 4000.

The Dockerfile copies your server's `.ssh/authorized_keys` into `~/.nx/config/authorized.crt` but I was unable to get key-based authentication to work (connections fail if `EnableNXClientAuthentication` is `1` and [this guy's fix](https://forums.nomachine.com/topic/nxclientauthentication-fails) doesn't work for me) so password-based authentication is left enabled. Use a strong password if you expose this to the internet!

<br />
### Docker tips

This section is a collection of pointers for Docker beginners to help with common tasks that you might want to perform with your container.

- If you want to pull files out of your container's home directory, navigate to `/var/lib/docker/volumes/` on the host. Do not try to insert files *into* the container this way. You should use SCP/SFTP or mount a network share into your container for inbound file transfers.
- With the Dockerfile above, any changes outside your user's home directory will be persisted to the overlay filesystem, which can't be easily accessed outside of Docker.
- To stop your container, run `docker stop dev`. To start it again, run `docker start dev`. Don't call `docker run` again as it will create a brand new container.
- To delete the (stopped) container, run `docker rm dev`. To delete the image, run `docker image rm dev`. To delete unused volumes, run `docker volume prune`.
- Because we called `docker run` with the `--restart always` argument, our container should automatically come back up if it crashes or if the host reboots.

[![NoMachine screenshot](/images/nomachine-vs-14w-thumb.png)](/images/nomachine-vs-14w.png)
