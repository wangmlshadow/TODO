# Knock-knock: Secure your SSH server using port knocking

> [Knock-knock: Secure your SSH server using port knocking | Code Capsule](https://codecapsule.com/2010/07/06/knock-knock-secure-your-ssh-server-using-port-knocking/)

In this article, I present a small raw socket daemon coded in C called “Knock-knock”, that allows port knocking to secure services under Linux. I have been using Knock-knock daily on a Ubuntu server for two years without any problem, and the code is small and simple enough for anybody to understand it.

I have been using an old computer running under Linux as a personal server for three years now. In order to log into the box, I just set up an ssh server. I thought that except for me, the box would not be of any interest to anybody, and that anyway, it would take them quite long to hack into it using brute force. Actually it took them a year, yep. After a year, some guy managed to break into the server by brute-forcing the ssh server. The fault is totally mine, for a poor choice of password. Luckily I noticed the intrusion, but I had to reinstall the server completely to make sure it was clean, which was an unnecessary hassle. I had to find a solution for this to never happen again.

> ‎在本文中，我提出了一个用C语言编码的小型原始套接字守护程序，称为“敲门声”，它允许端口敲门来保护Linux下的服务。两年来，我每天都在Ubuntu服务器上使用Nock-knock，没有任何问题，而且代码很小，很简单，任何人都可以理解它。‎
>
> ‎三年来，我一直在使用在Linux下运行的旧计算机作为个人服务器。为了登录盒子，我只是设置了一个ssh服务器。我认为，除了我之外，这个盒子对任何人都没有任何兴趣，无论如何，他们使用蛮力入侵它需要很长时间。实际上，他们花了一年时间，是的。一年后，有人通过暴力破解ssh服务器设法闯入服务器。错误完全是我的，因为密码选择不当。幸运的是，我注意到了入侵，但我不得不完全重新安装服务器以确保它是干净的，这是一个不必要的麻烦。我必须找到一个解决方案，让这种情况永远不会再发生。‎

## In search of a solution…

A non-conventional port number is a good start. By default, ssh is reachable through the port 22, therefore choosing a different port is a good way to prevent the server to be discovered too easily. However, a complete scan of all the ports on the machine will quickly reveal the server. Another important point is to have a strong password. This implies at least 12 characters, including upper and lower case letters, numbers and punctuation. But this will not stop the brute-forcing to happen. A solution for that is to use IPTable to block brute force attacks, but I don’t want my server to use its processor to deal with unnecessary traffic. Using an ssh key instead of a password could also be very effective, but it implies that I have the key available anywhere I need it need, which could be an issue. Finally, I decided to use port knocking to solve my problem, avoiding unnecessary traffic and preventing brute force hacking.

## Port knocking

Port knocking is used in network security to prevent malicious users from seeing services they should not have access to. This means that they are not even allowed to interact with the service they want to hack: they just can’t access it. This can be extremely useful if you have a small server at home. Indeed, whether you want it or not, you will undergo the continuous scans of script kiddies in desperate search for a computer to hack. With port knocking, one has to send a packet to a specific closed port in order for another port to be opened by the system or the firewall. I am not saying that the server will be made completely safe, but it will be made hard enough to hack for the average 13-year old script kiddie. This reduces dramatically the number of possible threats on your server. In our case, we want to send a packet to open the access to a ssh server. Variants include possibilities for encrypted packets or complex port knocking timed schemes.

Open source solutions for port knocking already exist, but most of them require the use of a specific port knocking client program. I wanted something simple that I could understand and install on my server easily, and that would not require any additional program for me to open the port to my ssh server. So I decided to code my own port knocking solution using raw sockets, it is called “Knock-knock”.

## Knock, knock!

The code for knock-knock is given at the end of the article, here I am just going to explain the idea behind it.

On most versions of Linux, port access can be allowed and forbidden using the network permission tables, which are located at */etc/hosts.allow* and */etc/hosts.deny* on a Ubuntu server (the location might change on a different version of Linux). Knock-knock is modifying the content of the permission tables on the fly to allow trusted connections. Now, how do we know if a connection can be trusted?

As explained above, several methods for port scheme and timing are possible, but I wanted something really simple that would not require any additional program or file. So I decided to use a double knock. This means that to open a connection, a packet on a specific closed port (let’s call it port #1) has to be sent, followed by another packet on another closed port (let’s call it port #2) within a time interval. The content of these packets is not important here, which means that this step can be performed using any program, like telnet, ssh, or even a web browser. Note also that port #1 and #2 can be the same, but having different port is probably safer. When Knock-knock detects the double knock from the same source on the specific IP ports, it opens the connection by modifying the permissions as stated in the previous paragraph. After the connection is established, port knocking is no longer necessary as the access is considered secured, and network services continue as usual. Knock-knock is running as a daemon, so that it can be starting when the server boots up, and handle all the access checking in the background. It has been designed to protect only one service (modifying the code for handling more services would be easy). Also, Knock-knock can treat only one connection at a time.

## Installing Knock-knock

### 1. Compiling the program

Download the source file from [here](http://github.com/goossaert/algorithms/raw/master/port-knocking/knock-knock.c), and put it on the server. Then compile it and set the permissions properly:

```
$ gcc knock-knock.c -o knock-knock
$ sudo chown root:root knock-knock
$ sudo chmod 700 knock-knock
$ sudo mv knock-knock /usr/bin/
```

### 2. Running the program as a service at startup

This part is specific to Ubuntu, and can be adapted to other Linux versions.

On Ubuntu, scripts in the */etc/rc2.d* directories are run at startup. The number 2 is the run level corresponding to a system running normally, and should be adapted to the run level your server is running on during normal use. To know which running you are in, simply type:

```
$ sudo runlevel
```

In order to run Knock-knock at startup, simply create a shell script in  */etc/init.d/* , and create a symbolic link in  */etc/rc2.d* . The script can be more or less complicated. You can lookup on the Internet for examples of deamon scripts with more options.

Just create a file named *knock-knock.sh* with the following content:

```
#! /bin/sh
/usr/bin/knock-knock
```

Then do the following:

```
$ sudo mv knock-knock.sh /etc/init.d
$ sudo chown root:root /etc/init.d/knock-knock.sh
$ sudo chmod 755 /etc/init.d/knock-knock.sh
$ sudo ln -s /etc/init.d/knock-knock.sh /etc/rc2.d/S90knock-knock
```

### 3. Preparing the permission files

The permission files must be modified prior to installing Knock-knock. Note that the following steps are working on Linux Ubuntu 9.04, and might differ when applied to a different distribution or version of Linux.

#### 3.1 Denying access to everybody

Add the following to the */etc/host.deny* file:

```
sshd: ALL
```

the “sshd: ” indicates the name of the service to modify, and “ALL” means that connections from all IP addresses must be rejected.

#### 3.2 Including a safety in case a problem occurs with the port knocking

This step is not necessary to install Knock-knock, however I recommend that you follow it. This simply adds an exception to the rejection of all connections. The idea is that you want to be able to log into your server even when something goes wrong. One solution is to add an exception for a specific address or address range on the local network of your server, or even in the Internet. For instance, if your server is on the sub-network  *192.168.1.** , you could say that all IP addresses on this network are allowed to access the server without using port knocking. In order to do that, add the following line to the /etc/host.allow file:

```
sshd: 192.168.1. : allow
```

### 4. Adjusting the parameters

As you can see, constants are used for the port numbers in the source file. A cleaner way to handle option variables is generally to use command-line parameters. But these parameters are visible in the list of the running processes. This means that even a user with restricted access on the machine could see these parameters by simply typing “ps aux”. Now, if these parameters include the knocking ports, then this user would learn which ports are used for knocking, and therefore bypass our port knocking mechanism. For that reason, none of the parameters as command-line arguments but as hard coded constants.

The parameters are:

 **SERVICE** : service to protect (default is “sshd”)
 **PORT_KNOCK1** : port number of the first knock
 **PORT_KNOCK2** : port number of the second knock (can be any value except PORT_KNOCK1 – 1 and PORT_KNOCK1 + 1, due to scan detection)
 **TIMEOUT_KNOCK** : timeout delay between the two knocks, in seconds
 **TIMEOUT_CONNECT** : timeout delay for the connection after the two knocks, in seconds
 **FILE_RULE** : path to the network permission file
 **FILE_RULETEMP** : path to a temporary file

## Logging

All accesses through port knocking are logged by the Knock-knock daemon using syslog. The logs can be seen with a grep on the file /var/log/syslog:

```
$ grep -e knock-knock /var/log/syslog
```

## Using Knock-knock

Once knock-knock is installed and running on the server, just send a packet to the first knocking port, following by a packet to the second knocking port, and then try to establish your ssh connection.

In order to send the knocking packets you can use ssh, by typing in a shell:

```
$ ssh 1.2.3.4 -p 8000
```

immediately followed by:

```
$ ssh 1.2.3.4 -p 9000
```

or you can also use a web browser, by typing in the address bar:

```
http://1.2.3.4:8000
```

immediately followed by:

```
http://1.2.3.4:9000
```

Then, immediately try to connect to your ssh server:

```
$ ssh 1.2.3.4 -p 22
```

## Conclusion

In this article, I have shown how to use port knocking to protect a service from being accessed by non-authorized users. I have also provided an implementation of a simple port knocking daemon,  *Knock-knock* , using raw socket programming in C under Linux. The installation process and the usage have been documented. Finally, remember that you can use port knocking for any service or port. Here I have used it for sshd, but it can be adapted to anything, all you have to do is to change the service/port and it will work!

## Source code

The source code is provided below for your convenience, you can also download it [here](http://github.com/goossaert/algorithms/raw/master/port-knocking/knock-knock.c).

```c
/******************************************************
 *  knock-knock 0.1 - GPL License v3.0
 *  Copyright (c) 2010 Emmanuel Goossaert
 *  emmanuel at goossaert dot com
 *
 *  See also: 
 *  - https://github.com/goossaert/algorithms/tree/master/port-knocking
 *  - http://codecapsule.com/2010/07/06/knock-knock-secure-your-ssh-server-using-port-knocking/
 * 
 *  Knock-knock is a daemon that implements port knocking
 *  in order to protect the access to a specific service
 *  on a server. It adds an ip address in the list of the
 *  authorized addresses for a service each time this ip
 *  address knocks on a specified port. After a certain
 *  time, this address is removed from the list: this is
 *  a temporary exception.
 *
 *  The main purpose is to allow just one ip to access
 *  a service during a short period of time, reducing
 *  the risks of brute force attacks. A scan detection
 *  has been included, and prevents the access to be
 *  open if someone knocks sequentially on all ports
 *  during a scan.
 *
 ******************************************************/


#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <time.h>

#include <syslog.h>
#include <signal.h>

#include <sys/socket.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <arpa/inet.h>


#define SIZE_LINE        512
#define SIZE_BUFFER      8192


// Parameters have been hardcoded for security reasons. Indeed, a simple 'ps'
// can reveal the parameters passed to a program, and would therefore reveal
// the knocking ports.

// name of the service to be protected
const char *       SERVICE         = "sshd";                

// port number of the first knock
const unsigned int PORT_KNOCK1     = 8000;                  

// port number of the second knock (can be any value except
// PORT_KNOCK1 - 1 and PORT_KNOCK1 + 1, due to scan detection)
const unsigned int PORT_KNOCK2     = 9000;                  

// timeout delay between the two knocks, in seconds
const unsigned int TIMEOUT_KNOCK   = 5;                     

// timeout delay for the connection after the two knocks, in seconds
const unsigned int TIMEOUT_CONNECT = 5;                     

// path to the network permission file
const char *       FILE_RULE       = "/etc/hosts.allow";    

// path to a temporary file
const char *       FILE_RULETEMP   = "/tmp/knock-knock";    

// name used by the daemon to log its events
const char *       DAEMON_NAME     = "knock-knock";         


struct tcpheader
{
    unsigned short int th_sport;
    unsigned short int th_dport;
    unsigned int th_seq;
    unsigned int th_ack;
    unsigned char th_x2:4, th_off:4;
    unsigned char th_flags;
    unsigned short int th_win;
    unsigned short int th_sum;
    unsigned short int th_urp;
}; // total tcp header length: 20 bytes (=160 bits)


struct ipheader
{
    unsigned char ip_hl:4, ip_v:4;
    unsigned char ip_tos;
    unsigned short int ip_len;
    unsigned short int ip_id;
    unsigned short int ip_off;
    unsigned char ip_ttl;
    unsigned char ip_p;
    unsigned short int ip_sum;
    unsigned int ip_src;
    unsigned int ip_dst;
}; // total ip header length: 20 bytes (=160 bits) 


char* make_rule_line( char * ip )
{
    static char line[ SIZE_LINE ];
    sprintf( line, "%s: %s : allow\n", SERVICE, ip );
    return line;
}


void add_ip( char * ip )
{
    char * line = make_rule_line( ip );
    FILE * file = fopen( FILE_RULE, "at");
    fwrite( line, 1, strlen( line ), file );
    fclose( file ); 
}


void remove_ip( char * ip )
{
    char line[ SIZE_LINE ];
    FILE *file_old = fopen( FILE_RULE, "rt");
    FILE *file_new = fopen( FILE_RULETEMP, "wt");
  
    while( fgets ( line, SIZE_LINE, file_old ) != NULL )
    {
        if( strcmp( make_rule_line( ip ), line ) != 0 )
            fwrite( line, 1, strlen( line ), file_new );
    }
  
    fclose( file_new ); 
    fclose( file_old );
  
    rename( FILE_RULETEMP, FILE_RULE ); 
}



void signal_handler( int sig )
{
 
    switch(sig)
    {
        case SIGHUP:
            syslog(LOG_WARNING, "Received SIGHUP signal.");
            exit(EXIT_FAILURE);
            break;
      
        case SIGTERM:
            syslog(LOG_WARNING, "Received SIGTERM signal.");
            exit(EXIT_FAILURE);
            break;
      
        default:
            syslog(LOG_WARNING, "Unhandled signal (%d) %s", sig, strsignal(sig) );
            break;
    }
}



int main(void)
{
    int fd = socket(AF_INET, SOCK_RAW, IPPROTO_TCP);
    char buffer[SIZE_BUFFER];
    struct ipheader *ip = (struct ipheader *) buffer;
    struct tcpheader *tcp = (struct tcpheader *) (buffer + sizeof(struct ipheader));

    pid_t pid, sid;

    // Error with PORT_KNOCK2 value
    if( PORT_KNOCK2 == PORT_KNOCK1 + 1 || PORT_KNOCK2 == PORT_KNOCK1 - 1 ) {
        exit(EXIT_FAILURE);
    }
 
    // Fork off the parent process
    pid = fork();
    if (pid < 0) {
        exit(EXIT_FAILURE);
    }
  
    // If we got a good PID, then we can exit the parent process.
    if (pid > 0) {
        exit(EXIT_SUCCESS);
    }
 
    // Create a new SID for the child process
    sid = setsid();
    if (sid < 0) {
        // Log the failure
        exit(EXIT_FAILURE);
    }

    setlogmask( LOG_UPTO(LOG_INFO) );
    openlog( DAEMON_NAME, LOG_CONS, LOG_USER );

    syslog( LOG_INFO, "daemon starting up" );  

    signal(SIGHUP, signal_handler);
    signal(SIGTERM, signal_handler);
    signal(SIGINT, signal_handler);
    signal(SIGQUIT, signal_handler);


    // Listen to the network and wait for someone to knock!
    while( read(fd, buffer, SIZE_BUFFER) > 0 )
    {
        char ipaddr[ 16 ];
      
        if( htons(tcp->th_dport) == PORT_KNOCK1 )
        {
            time_t timestamp = time( NULL );
            struct in_addr addr;
            addr.s_addr = ip->ip_src; 

            // Waiting for the confirmation packet
            while( read(fd, buffer, SIZE_BUFFER) > 0 )
            {
                // time is over!
                if( time( NULL ) > timestamp + TIMEOUT_KNOCK )
                    break;

                // this packet is not from the same source 
                if( addr.s_addr != ip->ip_src )
                    continue; 
              
                // detect scanning 
                if( htons(tcp->th_dport) == PORT_KNOCK1 + 1 || htons(tcp->th_dport) == PORT_KNOCK1 - 1 )
                    break;

                // we have an authorized access
                if( htons(tcp->th_dport) == PORT_KNOCK2 )
                {
                    strcpy( ipaddr, inet_ntoa( addr ) );
                    add_ip( ipaddr );
                    syslog( LOG_INFO, "giving access to: %s", ipaddr );  
                    sleep( TIMEOUT_CONNECT ); 
                    remove_ip( ipaddr ); 
                    break;
                }
            }
        }
    }
  
    syslog( LOG_INFO, "exiting, good bye!" );  
    return 0;
}
```
