## Setting-Up-Nethack-3.6.6-Server
A guide on how to compile [Nethack 3.6.6](https://www.nethack.org/v366/release.html) and [Dgamelaunch](https://github.com/paxed/dgamelaunch), and to configure them to serve game sessions via telnet.

If you just want to start playing, there's a number of public servers out there. [I've got one too.](https://nethack.zacharymwyatt.com/)

This guide details how to set up a Nethack 3.6.6 server on CentOS 7 or 8. If you are using a different distribution, the required packages may have different names, and the commands used to install them will differ depending on your package manager.

The steps to setup a Nethack 3.6.6 server are as follows:
1) [Installing Requisite Packages](#installing-requisite-packages)
2) [Configuring Nethack for the Chroot Environment](#configuring-nethack-for-the-chroot-environment)
3) [Compiling Nethack](#configuring-nethack-for-the-chroot-environment)
4) [Compiling dgamelaunch](#compiling-dgamelaunch)
5) [Creating the Chroot](#creating-the-chroot)
6) [Configuring Dgamelaunch](#configuring-dgamelaunch)
7) [Testing Nethack and Dgamelaunch](#testing-nethack-and-dgamelaunch)
8) [Setting up the Telnet Server](#setting-up-the-telnet-server)

Here are the directories we will be using in this example:

- Nethack will be configured before compiling in /home/nethack-temp/
- The chroot location will be /home/nethack/
- Dgamelaunch will be compiled into /home/dgamelaunch/


### Installing Requisite Packages
The commands that you will run to install all required packages:
```
yum install gzip make gcc ncurses-libs ncurses-devel byacc flex autoconf automake git sqlite sqlite-devel xinetd telnet-server

dnf config-manager --set-enabled powertools #If CentOS8
yum install flex-devel
```

Required packages for compiling nethack:
```
yum install gzip make gcc ncurses-libs ncurses-devel byacc flex
```

Required packages for compiling dgamelaunch:
```
yum install git autoconf automake sqlite sqlite-devel

dnf config-manager --set-enabled powertools #If CentOS8
yum install flex-devel
```

Required packages for the telnet server:
```
yum install xinetd telnet-server
```


#### Downloading Nethack
Download the tarball and unpack it.
```
mkdir  /home/nethack-temp/ && cd  /home/nethack-temp/
wget http://nethack.org/download/3.6.6/nethack-366-src.tgz
tar -xzf nethack-366-src.tgz
cd NetHack-NetHack-3.6.6_Released
```

### Configuring Nethack for the Chroot Environment
##### 1) Edit src/cmd.c

Change function enter_explore_mode() so users cannot do that; comment it out. Here's what that function will look like when commented out:
```
enter_explore_mode(VOID_ARGS)
{
/*    if (wizard) {
*        You("are in debug mode.");
*    } else if (discover) {
*        You("are already in explore mode.");
*    } else {
*#ifdef SYSCF
*#if defined(UNIX)
*        if (!sysopt.explorers || !sysopt.explorers[0]
*            || !check_user_string(sysopt.explorers)) {
*            You("cannot access explore mode.");
*            return 0;
*        }
*#endif
*#endif
*        pline(
*        "Beware!  From explore mode there will be no return to normal game.");
*        if (paranoid_query(ParanoidQuit,
*                           "Do you want to enter explore mode?")) {
*            clear_nhwindow(WIN_MESSAGE);
*            You("are now in non-scoring explore mode.");
*            discover = TRUE;
*        } else {
*            clear_nhwindow(WIN_MESSAGE);
*            pline("Resuming normal game.");
*        }
*    }
*/    return 0;
}
```
##### 2) Edit sys/unix/hints/linux
```
PREFIX=/
#PREFIX=$(wildcard ~)/nh/install
HACKDIR=$(PREFIX)/nh366
#SHELLDIR = $(PREFIX)/games
```

### Compiling Nethack
##### 1)  Run the setup script and `make all`.
```
cd sys/unix/ 
sh setup.sh hints/linux 
cd ../..
make all
make install
```

This will (unfortunately) install nethack under /
When I tried to install nethack using a different prefix, I couldn't get it to run after it had been moved into the chroot. So, we will be moving the compiled nethack into the chroot manually.

##### 2) Execute nethack and make sure it works.
```
/nh366/nethack
```

### Compiling Dgamelaunch

##### 1) Download the latest version from GitHub
```
mkdir /home/dgamelaunch/ && cd /home/dgamelaunch/
git clone https://github.com/paxed/dgamelaunch.git
```

##### 2) Run autogen.sh, making sure to use an etc/dgamelaunch.conf file location within the chroot directory you'll be using.
```
cd dgamelaunch
./autogen.sh --enable-sqlite --enable-shmem --with-config-file=/home/nethack/etc/dgamelaunch.conf
```

##### 3) Compile it.
```
make 
```

### Creating the Chroot
##### 1) Edit these lines in dgl-create-chroot, as follows:
```
CHROOT="/home/nethack/"
NHSUBDIR="/nh366/"
NH_VAR_PLAYGROUND="/nh366/var/"
NH_PLAYGROUND_FIXED="/home/nethack-compiled/nh366"
```

Also in dgl-create-chroot, change all references to "343" to "366". Here they are:
```
mkdir -p "$CHROOT/dgldir/inprogress-nh343"

chown "$USRGRP" "$CHROOT/dgldir/inprogress-nh343"

cp "$CURDIR/dgl-default-rcfile" "dgl-default-rcfile.nh343"

chmod go+r dgl_menu_main_anon.txt dgl_menu_main_user.txt dgl-banner dgl-default-rcfile.nh343
```

##### 2) Setup the Chroot
Create the chroot
```
./dgl-create-chroot
```

Move /nh366 into /home/nethack/
```
mv /home/nethack/nh366/var/ /home/nethack/
mv -f /nh366 /home/nethack/
chown -R games:games nh366/     
rm /games/nethack
```

### Configuring Dgamelaunch
##### Edit these lines in /home/nethack/etc/dgamelaunch.conf as follows:
```
 chroot_path  (enter full chroot path) "/home/nethack/"
 maxusers     (set the maximum number of registered users, not simultaneous users)
 SERVERID    (your server name)
 shed_uid     (UID of user "games")
 shed_gid     (GID of group "games")
 menu_max_idle_time (uncomment)
```

Also in dgamelaunch.conf, change all references to "343" to "366". Here they are:
```
commands["o"] = ifnxcp "/dgl-default-rcfile.nh343" "%ruserdata/%n/%n.nh343rc",

exec "/bin/virus" "%ruserdata/%n/%n.nh343rc"

commands["p"] = play_game "NH343"

game_path = "/nh343/nethack"

short_name = "NH343"

game_args = "/nh343/nethack", "-u", "%n"

rc_template = "/dgl-default-rcfile.nh343"

rc_fmt = "%ruserdata/%n/%n.nh343rc"

inprogressdir = "%rinprogress-nh343/"

commands = cp "/nh343/var/save/%u%n.gz" "/nh343/var/save/%u%n.gz.bak",

setenv "NETHACKOPTIONS" "@%ruserdata/%n/%n.nh343rc",
```

### Testing Nethack and Dgamelaunch
##### 1) Make sure that you can run nethack inside the chroot environment.
Try the following as root:
```
cd /home/nethack/
chroot ./ nh366/nethack
```

If you get an error like this..
```
# cd /home/nethack/
# chroot ./ nh366/nethack
nh366/nethack: error while loading shared libraries: libncurses.so.6: cannot open shared object file: No such file or directory
```

.. then manually copy the file into the chroot directory, and try running nethack again
```
# cp -v  /lib64/libncurses.so.6 /home/nethack/lib64/
'/lib64/libncurses.so.6' -> '/home/nethack/lib64/libncurses.so.6'
```

##### 2) Make sure that dgamelaunch works.
```
cd /home/nethack/
./dgamelaunch
```

If you do not get any output when running dgamelaunch, something is wrong.

### Setting up the Telnet Server
##### 1) Install the required packages if you have not already.
```
yum install telnet-server
```

##### 2) Set up the telnetd to accept incoming connections; if you're using xinetd like we are in this example, you can put this in /etc/xinetd.d/dgl:
```
service telnet
{
socket_type     = stream
protocol        = tcp
user            = root
wait            = no
server          = /usr/sbin/in.telnetd
server_args     = -h -L /home/nethack/dgamelaunch
rlimit_cpu      = 120
}
```
##### 3) Start the telnet server:
```
service xinetd start
chkconfig xinetd on
```

#### Testing the Telnet Server

Try connecting locally using `telnet 127.0.0.1`

If you get an immediate "connection closed by foreign host", then something is probably wrong with dgamelaunch. A common issue is that the dgamelaunch config file was not specified correctly, and dgamelaunch is looking for it in /etc/dgamelaunch.conf and not in the chroot.

#### Customizing the Dgamelaunch Menus

- Edit dgl-banner
- Edit dgl_menu_*

In particular, change the menu option in dgl_menu_main_user.txt from "Play NetHack 3.4.3" to 3.6.6.

#### Cleaning Up

/home/nethack-temp/ can be removed if you don't need it.

### External Links and References Used to Compile This Guide
https://nethackwiki.com/wiki/User:Paxed/HowTo_setup_dgamelaunch

https://web.archive.org/web/20170107151800/http://wiki.mc128k.info/index.php/Dgamelaunch_configuration
