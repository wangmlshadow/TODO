# Automatic repository and web server synchronization on commit

> [Automatic repository and web server synchronization on commit | Code Capsule](https://codecapsule.com/2010/10/26/automatic-repository-web-server-synchronization-commit/)

When developing a web site or web application, it is always a hassle to first commit the code to the repository, and then use FTP or SCP to update the code on the web server. Various IDEs allow to automatize this “code synchronization” process with a simple button to click, however this is still one additional step on the developer’s mind. We all went through these moments where, developing late in the night, we were trying to fix a bug and realized that we were working on the wrong version as we had forgotten to update the code on the web server. What a waste of time! For the development of a recent project, that includes sharing code with other developers, I decided to put an end to this code synchronization issue, by making it fully automatic. In this article, I first present the problem and my design of a configuration-independent solution. Then, I apply this solution to the specific context of my project, which uses Python as a web server, BitBucket/Mercurial as a repository solution, and Apache/PHP to handle the commit notifications.

## Motivation

Manually updating the files on a web server after a commit is a simple and straightforward step. However, it can be the source of various errors, such as:

1. Accidentally uploading the files in a wrong location on the web server
2. Forgetting to update the code on the web server and waste time debugging a deprecated version
3. Being distracted by thinking about updating the code on the web server, and losing focus on the development flow
4. Wasting time, as even if updating manually takes only a few seconds, these seconds will add up dramatically

The actual step of committing the code to the repository can hardly be automatized, as only the developer can say when the code is ready to be pushed. On the contrary, updating the code on the web server could easily be avoided. A simple system would be to get the web server to update its files and restart automatically whenever a commit is made to the repository. Figure 1 below describes the design of this automatic synchronization system.

![](https://i0.wp.com/codecapsule.com/wp-content/uploads/2010/10/autopull.jpg?resize=710%2C522 "Automatic synchronization of web server and repository on commit")
Figure 1 - Automatic synchronization of web server and repository on commit

## Application: Pylons, Mercurial and PHP

For a recent project, I am using Mercurial on [BitBucket](http://www.bitbucket.org/) to share code with other developers (step 1 on Figure 1). To test the web application that we are developing using Pylons/Python, I have a little web server at home. The idea would be to update and restart this web server every time someone commit to the repository. I could use hooks with Mercurial, but there is a simpler way to do that. Indeed, BitBucket offers many possibilities of Service Integration, including a POST request. With this, every time a commit is made on the repository, a HTTP/POST request is sent to a specific URL (step 2 on Figure 1). Note that although I am using BitBucket here, this HTTP/POST solution can be easily adapted to other repository services such as GitHub. All what remains to be done from there is a script, in my case a PHP script, that receives this POST request, pulls the code from the repository and restart the server if necessary (steps 3 and 4 on Figure 1). From there, the web server is serving the last version of the code committed to the repository (step 5 on Figure 1).

I am not going to discuss the details of setting a BitBucket account, installing Mercurial on your computer or installing Pylons on the server. Many documentations sources are already doing that very well. I am only going to give the details of the steps and scripts that I am using to get the whole system to work.

### 1. Web server daemon

Considering that my web server is running on Ubuntu Server 10.04, here is what I did to get Pylons to run as a daemon. First, I created a file named **pylons** in the **/etc/init.d/** directory, with the following content:

```
#!/bin/sh -e

user=www-data
dir_project="/home/www-data/pylons"
file_pid="$dir_project/paster.pid"
file_log="$dir_project/paster.log"
file_ini="$dir_project/development.ini"

case "$1" in
    start)
        cd $dir_project
        sudo -u $user paster serve --daemon --pid-file=$file_pid --log-file=$file_log $file_ini start
        ;;
    stop)
        cd $dir_project
        sudo -u $user paster serve --daemon --pid-file=$file_pid --log-file=$file_log $file_ini stop
        ;;
    restart)
        cd $dir_project
        sudo -u $user paster serve --daemon --pid-file=$file_pid --log-file=$file_log $file_ini restart
        ;;
    *)
    echo $"Usage: $0 {start|stop|restart}"
    exit 1
esac
```

(I modified an existing daemon script by Max Ischenko and Mike Orr, available at: [http://wiki.pylonshq.com/display/pylonscookbook/Scripts+for+paster+serve](http://wiki.pylonshq.com/display/pylonscookbook/Scripts+for+paster+serve))

Then, I made this script executable:

```
$ sudo chmod 755 /etc/init.d/pylons
```

and I added a link in the **/etc/rc2.d/** directory:

```
$ cd /etc/rc2.d
$ sudo ln -s ../init.d/pylons S20pylons
```

Depending on the configuration of your server, you can repeat this operation for **rc3.d** and  **rc4.d** . Also, I added a cron job that tries to start the Pylons server every 15 minutes. If the server is still up, it won’t change anything, but if the server has crashed, it will automatically restart it. The line I added to the **/etc/crontab** file is:

```
*/15 *  * * *   root    /etc/init.d/pylons start
```

Finally, as you might have noticed, the script is running Pylons as the **www-data** user. I am doing this because on my server I already have Apache, and **www-data** is the user running all web related services, so for simplicity and coherence, I kept the same logic and decided to run the Pylons daemon and do all the code update operations with the **www-data** user. Feel free to use whatever user or logic that fits your needs better.

### 3. Create the SSH key for the **www-data** user

The BitBucket repository service uses SSH to authenticate connections. As my **www-data** user is used by Apache, its home directory is  **/var/www/** . I therefore created a **www-data** directory in  **/home/** , in which I included a **.ssh** directory. So now I have the new directory path:  **/home/www-data/.ssh** . I created a new SSH key pair for the **www-data** user with the command:

```
$ sudo -u www-data ssh-keygen -t rsa
```

I chose **/home/www-data/.ssh/id_rsa** as the path of my private key (the corresponding public key should be saved at the same location with the name  **id_rsa.pub** ). Finally, I added the public key to the BitBucket account that I am using.

### 4. Clone the repository

On my server, I have the Pylons server running in /home/www-data/pylons/. This is also the code that I am sharing on BitBucket, so I simply clone the repository structure from BitBucket:

```
$ sudo -u www-data hg clone --ssh '/usr/bin/ssh -o StrictHostKeyChecking=no -i /home/www-data/.ssh/id_rsa' ssh://hg@bitbucket.org/user/myproject
```

From now on, all I will to do is to pull code in the **/home/www-data/pylons/myproject/** directory.

### 5. Set up the notification PHP script

As I have Apache/PHP already running on my server, I am going to use it to receive the notification POST request from the BitBucket notification service. Note that I could use any technology, server, or language, but given that I am going to restart the Pylons server, I prefer to have the notification script running with a different server (Apache in my case). Feel free to use whatever solution you want, Apache is just an example.

In my **/var/www/** directory, the one from which Apache serves web pages on my server, I added a **myproject** directory. In this directory, I created the following files:

**commit_tracker.php**

```
<?php
    exec('./run');
?>
```

**run**

```
#!/bin/sh
nohup ./update 1>/dev/null 2>&1 &
```

**update**

```
#!/bin/sh
cd /home/www-data/pylons/myproject
hg pull -u --ssh '/usr/bin/ssh -o StrictHostKeyChecking=no -i /home/www-data/.ssh/id_rsa' ssh://hg@bitbucket.org/user/myproject

dir_server="/home/www-data/pylons"
file_pid="$dir_server/paster.pid"
file_log="$dir_server/paster.log"
file_ini="$dir_server/development.ini"

cd $dir_server
paster serve --daemon --pid-file=$file_pid --log-file=$file_log $file_ini restart
```

Finally I made the **run** and **update** files executable:

```
$ chmod 755 run
$ chmod 755 update
```

### 6. Add the service integration

The only thing that remains to be done is the integration of this PHP notification script with the BitBucket repository. In the “Admin” panel of my project in the BitBucket interface, I went in the “Services Administration” menu, and from there, I set the “POST” service with the URL **http://1.2.3.4/myproject/commit_tracker.php** (1.2.3.4 being the IP address of my web server).

With this little setup, whenever a commit is made, the URL **http://1.2.3.4/myproject/commit_tracker.php** will be hit, and the web server will use Mercurial to pulls the code from the BitBucket repository, and restart the Pylons server based on that code. That might seem to be nothing, but it is a huge step ahead in the development process.

Note that if you are using GitHub, the same service is available, in the “Admin” panel, and then in the “Service Hooks” and “Post-Receive URLs” menus.

## Conclusion

The solution that I have presented is not perfect, but it is cheap, light, and easy to set up. But more importantly, it is doing great for the environment I am working in, and with the limited resources I have. If you want to use it for your own projects, here is a list of the possible problems and improvements:

1. If a malicious person discovers about your commit tracking script, he can call the URL repetitively to interfere with your work by forcing your server to restart. A simple check on the host name in the script could avoid this kind of problems.
2. If a malicious person spoof the IP of your repository server, he could possibly upload to your server whatever code he wants, which can have serious consequences. Make sure you always talk to your repository server.
3. The code in the **pylons** and **update** files are partly the same. It would be good to somehow factorize the common parts.
4. As the code is automatically updated on the web server at every commit, this can crash the web server in case the code contain bugs. Therefore, there should be some sort of mechanism on the web server to detect such crashes and revert the code to the last working version.
5. The system is right now designed to handle only one repository, but it would be easy to modify it to handle multiple repositories.
