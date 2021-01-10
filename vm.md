# General setup
VirtualBox Version: `5.0.32 r112930`  
Guest OS: `Ubuntu 14.04`, `Ubuntu 16.04`  
Host OS: `Windows 10`

- Download guest ISO
- Create a new virtual machine
- Create a dynamically allocated storage disk for easy resizing
- Assign appropriate resources:
    - ~60MB of Video Memory
    - ~2GB of RAM
    - Enable 3D acceleration
    - General -> Advanced -> Drag n' Drop + Shared Clipboard -> Bidirectional
    - Network -> NAT
    - System -> Processor -> 2
- Load ISO and start the machine
- Setup
- Install necessary build tools and build dependencies:
    ```bash
    sudo apt-get install -y dkms build-essential linux-headers-generic linux-headers-$(uname -r)
    ```
- Either mount Guest Additions and install or if installing server:
    Devices -> Insert Guest Additions CD Image:
    ```bash
    sudo mount /dev/cdrom /media/cdrom
    sudo /media/cdrom/VBoxLinuxAdditions.run
    ```
    Add `--nox11` when installing on GUI-less machine (no need on Ubuntu 16+)  

- Installing on Kali Linux:
    ```bash
    apt-get update
    apt-get dist-upgrade
    reboot
    apt-get install -y virtualbox-guest-x11
    ```
- Reboot

# Resizing dynamic drive
- Download GParted ISO
- On Windows (from VirtualBox install directory):
    ```powershell
    VBoxManage.exe modifyhd "<absolute path to .vdi image>" --resize <size_in_MB>
    ```
- Load GParted on VM's disk drive and boot
- Press F12 and load from CDROM
- Unallocated space should be shown as a new partition
- If swap partition is preventing extension:
    - Delete swap partition
    - Move swap space to the far right
- Extend the main partition to occupy the unallocated space
- Recreate linux-swap partition
- Apply changes
- Reboot

# Adding internal network between VMs
- Have an adapter for internet (ususally NAT)
- Add a secondary network adapter: Internal Network
- Make up a name or use the default "intnet"
- Do this for all VMs you want on the network; use same network name as previous step
- Setup a DHCP server for the network:
    ```powershell
    VBoxManage dhcpserver add --netname <name> --ip <dhcp> --netmask 255.255.255.0 --lowerip <low> --upperip <high> --enable
    ```
    `<name>`: the network name you chose before  
    `<dhcp>`: random ip for dhcp server  
    `<low>`: lower bound for ip range(netmask dictates which digits can be used)  
    `<high>`: upper bound for ip range  

    The above netmask (`255.255.255.0`) means only the last byte will be taken into account, so you can only use a range that changes the last number.  
    Example:   
    ```powershell
    VBoxManage.exe dhcpserver add --netname intnet --ip 100.100.100.1 --netmask 255.255.255.0 --lowerip 50.50.50.10 --upperip 50.50.50.20 --enable
    ``` 
- You can use `VBoxManage list dhcpservers` to make sure your server has been created. Make sure it's enabled ;)
- Both networks should now be enabled. If they aren't:
    - Find missing interface name:
        - On Ubuntu > 17.04:            
            A new network configuration tool has been introduced, netplan:
            ```bash
            # get the inactive interface name
            $ ifconfig -a
            # Edit /etc/netplan/01-netcfg.yaml
            # Add the ifname under the "ethernets" submenu. E.g.:
            ...
            ethernets:
              enp0s8:
                dhcp4: yes
            ...
            $ sudo netplan apply
            ```
        - On Ubuntu > 15.10 (http://askubuntu.com/a/778679):
            ```bash
            $ lspci
            
            # Cross-reference network card name with the one in VirtualBox network settings for internal adapter
            # Figure out the internal network interface name from the entry, e.g:
            #    00:08.0 Ethernet controller: Intel Corporation 82540EM Gigabit Ethernet Controller (rev 02)
            #    ... means it will be `enp0s8` (<ifname>)
            # Add following at the end of /etc/network/interfaces:
            auto <ifname>
            iface <ifname> inet dhcp

            $ service networking restart
            ```            
        - On Ubuntu < 15.10 (http://unix.stackexchange.com/a/37227):            
            If your first adapter is on eth0, your new interface is probably on eth1; obviously no research in this ;)
            - Add following at the end of `/etc/network/interfaces`:
                ```
                auto eth1
                iface eth1 inet dhcp
                ```
            ```bash    
            ifup eth1
            ```
            - In general your `/etc/network/interfaces` file should look like this:
                ```
                # The loopback network interface
                auto lo
                iface lo inet loopback

                auto eth0
                iface eth0 inet dhcp

                auto eth1
                iface eth1 inet dhcp
                ```
                If it doesn't it probably means it's misconfigured so you should add `eth0` as well as `eth1` and reboot.
- Good to go. You can SSH from one machine to the other on the same network (you need `openssh-server` installed first)

# Adding auto-login to ubuntu 16.04
http://askubuntu.com/questions/771837/how-to-create-ubuntu-server-16-04-autologin  
http://askubuntu.com/questions/819117/autologin-at-startup-to-ubuntu-server-16-04-1-lts/819185  
```bash
$ sudo systemctl edit getty@tty1

# Add:
[Service]
ExecStart=
ExecStart=-/sbin/agetty -a <USERNAME> --noclear %I $TERM

$ sudo systemctl restart getty@tty1
```

# Expose VM port to outside world
- Have NAT network adapter
- Advanced -> Port Forwarding -> Add new rule
- Add the host port -> guest port mapping, e.g. 123 -> 22
- Setup static IP for your host machine
    - On Windows:
        - Search -> Network Connections -> Select your connection
        - Properties -> TCP/IPv4 Properties -> Properties -> Use the following IP address
        - Make sure you select an IP outside the DHCP range; If not possible - lower range
        - Enter the subnet mask, default gateway & DHCP server as they were before; use `ipconfig /all`
        - You got a static internal IP for your device
- Edit router and allow port 123 for host machine's IP above
- You might need to add firewall rule
- Now you can access `<external_ip>:123` and it will redirect to `VM:22` (if you have VM running and something listening)

# Access host localhost from guest (on NAT network)
`10.0.2.2`

# Enable X11 Forwarding on Ubuntu 16+
Host:  
- Install VcXsrv (https://sourceforge.net/projects/vcxsrv/)
- Start it up from XLaunch or use `%VcxsrvDir%\vcxsrv.exe -multiwindow -swrastwgl -silent-dup-error`. For first option use all the defaults.
- In Putty session for target guest machine:
    - Connection -> SSH -> X11: 
    - Enable X11 Forwarding
    - X display location: `127.0.0.1:0.0`

Guest:  
- On a clean install of Ubuntu Server 16+ install VBox guest additions
- Install an application with GUI, e.g gnome-terminal:
    ```bash
    $ sudo apt install gnome-terminal
    # needed for x11 forwarding
    $ sudo apt install dbus-x11
    ```
- Start the GUI application using dbus-launch, otherwise some weird errors happen. Also make sure that you start it from the same user each time:
    ```bash
    $ dbus-launch gnome-terminal
    ```
- If errors check if `.Xauthority` is present in user home dir.

# Using Host-Only adapter for victim machines
- Go to File->Prefs->Network->Host-Only Networks
- Select the network you will be using and click edit
- Make sure DHCP server is enabled and assign low/high IP addresses
- Set up a 2nd host-only adapter on attacker machine
- For Ubuntu 17.10 you have netplan. Edit /etc/netplan/01-netcfg.yaml:
    ```bash
    # Add the name of the iface below ethernets (get it from ifconfig -a, the disabled one)
    # Add the addresses of the dhcp server ip range (from step 3). 
    # Example:
    ...
    enp0s8:
      addresses: [192.168.10.10/24]
    ...

    $ sudo netplan apply
    ```
- On attacker use only a single adapter with Host-Only.
- If this doesn't work try switching the adapter type (Network->Advanced) to something else. PCnet-PCI II seems to work for some.
- Test that the victim is discoverable usign `nmap -sn 192.168.10.1/24` (put the dhcp range here).

# Setting up a remote desktop environment on Ubuntu Server 18+
We will be setting up xfce4 as the desktop environment (for lightweight browser look at `midori`):
```bash
$ apt install firefox dbus-x11 xfce4 xfce4-goodies tightvncserver xfonts-base x11-xserver-utils

# Kill the autostarted server
$ vncserver -kill :1

## Edit ~/.vnc/xstartup:
#!/bin/bash
xrdb $HOME/.Xresources
startxfce4 &

# Restart vncserver and setup password (non view-only)
$ vncserver

# Enable ufw to disable remote access
$ ufw allow 22/tcp
$ ufw enable

# Enable a fast, compressed ssh connection with local port forwarding (N flag = no shell/command execution)
$ ssh -C4Nc aes128-gcm@openssh.com -L 5901:localhost:5901 <user>@<target_ip>

# Connect with vncviewer on your machine, entering the password that you set up previously
$ vncviewer localhost:5901
```
