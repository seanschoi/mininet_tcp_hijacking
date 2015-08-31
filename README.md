# mininet_tcp_hijacking

This is a demo of a TCP hijacking using Mininet.

sudo apt-get install xinetd telnetd 

sudo touch /etc/xinetd.d/telnet

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

netstat -tulpn | grep :23
