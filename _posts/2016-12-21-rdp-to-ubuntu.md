---
layout: post
title: How to connect through VNC to Ubuntu on Azure
---

Connecting to an Ubuntu Server using VNC. Note*:

## Requirements 

- An Ubuntu VM up and running in Azure.
- SSH acces to your VM. You can use [Windows 10 Bash](https://msdn.microsoft.com/en-us/commandline/wsl/about) or [PuTTy](http://www.chiark.greenend.org.uk/~sgtatham/putty/download.html)

## Steps

1. If you haven't done it from the portal, you can use the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-az-cli2) to deploy an Ubuntu Server 16.04. You will need to run this command from a terminal that has SSH installed and your SSH keys already generated because it will try to obtain `~/.ssh/id_rsa.pub`. Also, the username you have in your terminal will give you access to the VM.

    ```Bash
    az group create --name tests --location southcentralus
    az vm create -g tests -n bubulubuntu --image canonical:ubuntuserver:16.04-LTS:16.04.201612210
    ```

    Be patient, the creation of the VM could take some minutes. After it's done you will obtain a public IP that you can use to SSH into your VM `ssh <<VM PUBLIC IP>>`

- Inside of the VM you should be seeing this output.

    ```Bash
    Welcome to Ubuntu 16.04.1 LTS (GNU/Linux 4.4.0-57-generic x86_64)

    * Documentation:  https://help.ubuntu.com
    * Management:     https://landscape.canonical.com
    * Support:        https://ubuntu.com/advantage

    Get cloud support with Ubuntu Advantage Cloud Guest:
        http://www.ubuntu.com/business/services/cloud

    0 packages can be updated.
    0 updates are security updates.


    Last login: Wed Dec 21 21:49:06 2016 from 131.107.160.121
    To run a command as administrator (user "root"), use "sudo <command>".
    See "man sudo_root" for details.


    bruno@bubulubuntu:~$
    ```

    Lets start by installing [TaskSel](https://help.ubuntu.com/community/Tasksel), which will manage the installation of the Desktop Environment.

    ```Bash
    sudo apt-get update
    sudo apt-get upgrade
    sudo apt install tasksel
    ```

1. Run `tasksel` and install `Ubuntu Desktop`

    ```Bash
    sudo tasksel
    ```

    ![alt text][tasksel]

    Here you should put the cursor on the package "Ubuntu Desktop" and press `space` in your keyboard to mark it for installation, then press `tab` to go to `<Ok>` and finally `enter` to start the installation.

    ![alt text][tasksel-install]

1. While this is installing in your server install a VNC client like [VNC Viewer from RealVNC](https://www.realvnc.com/download/viewer/)

<!--1. While this is installing in your server, go ahead and install [Xming X Server for Windows](https://sourceforge.net/projects/xming/). After you are done installing it, you can test it with Windows 10 Bash by installing and running `xterm` in your Windows 10 Bash. Don't forget to set the `$DISPLAY` env variable.

    ```Bash
    export DISPLAY=:10.0
    sudo apt-get install xterm
    xterm
    ```

    If everything was set up correctly, you should see the following window.

    ![alt text][xterm]
-->

1. Back to your Ubuntu 16.04 server in the cloud, when the installation is done, install the Vnc Server you have to reboot the VM.

    ```Bash
    sudo apt-get install vnc4server
    sudo shutdown -r now
    ```

    In your terminal you would see something like this:

    ```Bash
    Connection to 13.66.48.250 closed by remote host.
    Connection to 13.66.48.250 closed.
    ```
    Give it a couple of minutes before trying to going back in again.

1. SSH back into your Linux box and configure `vncserver`, don't forget the password you choose (only 8 chars)

    ```Bash
    vncserver
    ```

1. Edit `xstartup`. First lets stop the server

    ```Bash
    vncserver -kill :1
    ```

    And edit `~/.vnc/xstartup` so it looks like this:

    ```Bash
    #!/bin/sh

    # Uncomment the following two lines for normal desktop:
    unset SESSION_MANAGER
    exec sh /etc/X11/xinit/xinitrc

    [ -x /etc/vnc/xstartup ] && exec /etc/vnc/xstartup
    [ -r $HOME/.Xresources ] && xrdb $HOME/.Xresources
    xsetroot -solid grey
    vncconfig -iconic &
    x-terminal-emulator -geometry 80x24+10+10 -ls -title "$VNCDESKTOP Desktop" &
    x-window-manager &
    ```

    And now restart the vncserver

    ```Bash
    vncserver
    ```

1. Make a tunnel

    ```Bash
    ssh -L 5901:127.0.0.1:5901 -N -f -l
    ```

1. Open VNC viewer and connect to `localhost:5901`

## Resources

- https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-vnc-on-ubuntu-16-04
- http://askubuntu.com/questions/592537/can-i-access-ubuntu-from-windows-remotely
- http://c-nergy.be/blog/?p=9962
- http://c-nergy.be/blog/?p=5956
- http://stackoverflow.com/questions/25657596/how-to-set-up-gui-on-amazon-ec2-ubuntu-server


[tasksel]: ../img/tasksel.jpg "Tasksel does the GUI installation for us"
[tasksel-install]: ../img/tasksel-Ubuntu-desktop-installation.jpg "It takes a while, be patient!"
[xterm]: ../img/xterm-running-windows-10.jpg "Xterm running on Windows 10 Bash"