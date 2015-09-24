# TCP Hijacking Attack Demo using Mininet

## Introduction
In this demo, we are going to show you how a malicious attacker can hijack a tcp connection between two hosts via ARP Spoofing to become the Man in the middle host. The attacker will forge a valid packet to be sent to one of the host, which will complete the hijacking procedure. We will show this using Telnet TCP connection.

## Getting Started

### Mininet Installation

This example requires Mininet installation. Mininet is a software based network emulator. It allows the users to run a collection of end-hosts, switches, routers, and links on a single Linux kernel. You can either install Mininet on your computer or run this example through a VM image which is provided in the Mininet website. More detailed installation instructions and more information about Mininet can be found in http://mininet.org/download/

### Ettercap Installation

Ettercap is a comprehensive suite for man in the middle attacks. It features sniffing of live connections, content filtering on the fly and many other interesting tricks. It supports active and passive dissection of many protocols and includes many features for network and host analysis. We recommend using the Mininet VM image, as it is based on Ubuntu 14.04, making ettercap installations easier as well.

#### Ubuntu 14.04 Installation (Mininet VM Image)
Simply run `sudo apt-get install ettercap-graphical`

#### Others
Refer to https://github.com/Ettercap/ettercap for detailed instructions on building and installing ettercap

#### Change the conf
Under /etc/ettercap/etter.conf make the following changes

From
```
# if you use iptables:
# redir_command_on = "iptables -t nat -A PREROUTING -i %iface -p tcp --dport %port -j REDIRECT --to-port %rport"
# redir_command_off = "iptables -t nat -D PREROUTING -i %iface -p tcp --dport %port -j REDIRECT --to-port %rport"
```

To
```
# if you use iptables:
redir_command_on = "iptables -t nat -A PREROUTING -i %iface -p tcp --dport %port -j REDIRECT --to-port %rport"
redir_command_off = "iptables -t nat -D PREROUTING -i %iface -p tcp --dport %port -j REDIRECT --to-port %rport"
```

### Telnet Installation

This is an instruction to install Telnet specifically for Ubuntu 14.04 with xinetd. For other operation systems, please use this as a reference, but note that it may not be applicable.

First install telnet by running `sudo apt-get install xinetd telnetd`

Then, create the telnet configuration file for xinetd by running `sudo touch /etc/xinetd.d/telnet`
Finally, edit `/etc/xinetd.d/telnet` to have the following contents.
```
service telnet
{
        flags           = REUSE
        socket_type     = stream
        wait            = no
        user            = root
        server          = /usr/sbin/in.telnetd
        log_on_failure  += USERID
        disable         = no
}
```
Now restart xinetd by running `/etc/init.d/xinetd` to apply the above changes.

Finally, check that telnet is running by running `netstat -tulpn | grep :23` to see which service is listening on port 23. Unless you configured it otherwise, there should be an entry for telnet service.

## Running the example

### Clone the repository

You would first need to clone this repository and go into the folder by running
```
git clone https://github.com/yo2seol/mininet_tcp_hijacking.git
cd mininet_tcp_hijacking
```
The folder contains 4 files, `README.md, tcp_hijacking.py, run_mininet.sh, run_ettercap.sh`.  `tcp_hijacking.py` is the main mininet topology and configuration file. Other scripts are to run aid the user to run the respective services.

### Run the mininet CLI 

Start the mininet topology as defined in `tcp_hijacking.py` by running `./run_mininet.sh` script or running `sudo python tcp_hijacking.py`. The topology contains 3 hosts attached to a single switch with the following definitions.
```
host_name ip  mac_address
h1  192.168.0.3 00:00:00:00:01
h2  192.168.0.4 00:00:00:00:02
h3  192.168.0.5 00:00:00:00:03
```
`h1` will act as a ssh client, and `h2` will act as a ssh server. `h3` will be the man in the middle spoofing on the traffic between `h1` and `h2`.

Running this command will complete the following actions.

* Setup the topology as described above
* Start the ssh server on host 2
* Create logfile on host 3 to capture the ssh output

### Open the terminals for each hosts
Open the terminal for each host by running `xterm h1 h2 h3`

If using ssh terminal to access your VM, you may need to add `-Y` parameter for your `ssh` function. Also, Mac OSX users may need to install xQuartz in http://xquartz.macosforge.org/landing/, as X11 is no longer a default software installed in Mac OSX.

### Set up IP forwarding and start ettercap on the Attacker

#### (Optional) Set up IP Forwarding
In host 3's terminal, run the following command to set up ip forwarding.
```
echo 1 > /proc/sys/net/ipv4/ip_forward, cat /proc/sys/net/ipv4/ip_forward
```
This enables linux kernel IP forwarding, so that it can forward packets received from a host to another host.

#### Start Ettercap
Run ettercap using the following command on `h3`
```
./run_ettercap.sh
```
or which basically runs
```
ettercap -G
```
This starts ettercap in a graphical interface.

### Ettercap Directions

#### Initial Setup

This part will initialize the sniffing as well as arp spoofing. More about ARP spoofing can be found in https://en.wikipedia.org/wiki/ARP_spoofing

1. `Options`-> Check to see that only `Promisc Mode` is checked
2. `Sniffing` -> `Unified Sniffing` -> `h3-eth0` as the network interface -> Click `Ok`
3. `Hosts`-> `Scan for Hosts`. This will scan all hosts connected to your network.
4. `Hosts` -> `Host List` to view the hosts. You will see `h1, h2` as defined by their IP addresses.
5. Add `192.168.0.3` to target 1 and `192.168.0.4` to target 2.
6. Check that the targets are assigned correctly by going to  `Targets` -> `Current Targets`
7. Enable ARP Spoofing as follows. `Mitm` -> `Arp poisoning` -> Check `Sniff Remote Connections` -> Click `Ok`.
  Check the log to see that GROUP assignments are similar to your targets.
8. `Start` -> `Start Sniffing`

This will cause `h3` to send malicious ARP information to `h1` and `h2` and cause the traffic between two hosts to flow through `h3`. 

#### TCP Hijacking

1. Run `telnet 192.168.0.4` to connect to `h2`. You can use tools are wireshark (https://www.wireshark.org/) in `h3` to see that the ssh requests are going through `h3` instead of going straight to `h2`. Login with username `mininet` and password `mininet`, if using the mininet vm.
2. In ettercap go to `View` -> `Connections` to see the telnet connection between `h1` and `h2`. Double click on the connection to view the connection data.
3. Now, you can hijack the TCP Connection here by injecting data. Click on `Inject Data` and select the destination to be `192.168.0.4`. Try adding commands like `ls\n`, which will show the list of files from `h2` on the right side. You have successfully Hijacked a TCP Connection!

