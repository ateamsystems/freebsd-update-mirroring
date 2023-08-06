# FreeBSD Update Mirroring

Many FreeBSD users and system administrators have to manage multiple machines or environments.
They face various challenges and demands when they need to keep their infrastructure updated with the latest security and software patches.
A FreeBSD Update Server can simplify this process by allowing them to test the updates on some machines before applying them to the rest of the network. 
It also means they can update their systems faster from a local network instead of a slower Internet connection.

This document describes about how to fetch and place FreeBSD OS update related files and patches and use them later 
when using freebsd-update script from other FreeBSD servers.

##	Using modified freebsd-update script

To set up a FreeBSD update server, you need to use a modified version of the freebsd-update script. 
We have called this script freebsd-update-mirror to avoid confusion. It is heavily based on freebsd-update script.
This script downloads and stores the FreeBSD update patches and files in a specific directory.
One of the important options freebsd-update script that can be used with freebsd-update-mirror:

<pre>
--currently-running release
</pre>

This option is very useful when fetching necessary patches and files assuming that the system is running the specified release.

In addition to it, the following option related changes are added to freebsd-update script:

<pre>
-m           -- Mirror mode, download files
</pre>

This option -m can be used together with the following option:

<pre>
-d workdir   -- Store working files in workdir
</pre>

In order to run freebsd-update-mirror using -d, -m option, we need to create working directory first:

<pre>
mkdir 2023Q1
</pre>

This directory then can be used in nginx or apache web servers to serve other FreeBSD servers.


Then sample command can be run using "freebsd-update-mirror fetch" command like:

<pre>
./freebsd-update-mirror fetch -d 2023Q1 --currently-running 13.1-RELEASE -m
</pre>

So above command assumes it is running on 13.1-RELEASE and will store FreeBSD update related files 
into 2023Q1 directory similar to the following:
<pre>
...
drwxr-xr-x  3 root  wheel    512 Oct 13 19:04 13.1-RELEASE
lrwxr-xr-x  1 root  wheel     14 Jan 17 11:13 f465c3739385890c221dff1a05e578c6cae0d0430e46996d319db7439f884336-install -> install.8yvFZk
drwxr-xr-x  2 root  wheel  52736 Oct 13 19:05 files
drwx------  2 root  wheel    512 Jan 17 11:13 install.8yvFZk
-rw-r--r--  1 root  wheel    800 Aug 31 07:06 pub.ssl
-rw-r--r--  1 root  wheel     50 Jan 17 11:13 serverlist
-rw-r--r--  1 root  wheel     50 Jan 17 11:13 serverlist_full
-rw-r--r--  1 root  wheel     25 Jan 17 11:13 serverlist_tried
-rw-r--r--  1 root  wheel    150 Jan 17 11:13 tINDEX.present
-rw-r--r--  1 root  wheel    113 Jan 17 11:13 tag
...
</pre>

In above case, all the update files are downloaded to 2023Q1 directory. 
2023Q1/13.1-RELEASE directory is important and it can be used to serve from nginx or apache web servers to other FreeBSD servers.
The files outside 13.1-RELEASE directory is not important in this case.


##	Configure Nginx web server and copy files to the necessary place

Once we have update files (13.1-RELEASE directory in above case), we can configure nginx to be used as FreeBSD update server. 
Nginx server can be configured on current server or on another FreeBSD server.

The nginx can be installed in FreeBSD using “pkg install nginx” command. 
Then we need to configure nginx, edit /usr/local/etc/nginx/nginx.conf file to containg following:

<pre>
user  www www;
worker_processes  4;

events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;

    keepalive_timeout  65;

    server {
        listen 80;
        server_name  custom-freebsd-update-server.com;
        root /zoo/freebsd-updates;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/local/www/nginx-dist;
        }

        autoindex on;
        autoindex_exact_size off;

        location / {
                root /zoo/freebsd-updates;
        }
    }
}
</pre>

Once configured in above way, put nginx_enable="YES" line in /etc/rc.conf and start nginx:

<pre>
service nginx start
</pre>

Then copy 13.1-RELEASE directory to nginx server root directory, in above case it is /zoo/freebsd-updates/ directory.
The /zoo/freebsd-updates directory should contain 13.1-RELEASE directory.

<pre>
...
drwxr-xr-x  3 root  wheel  3 Oct 13 05:34 13.1-RELEASE
...
</pre>

Then you can open browser and check if nginx is working by using its IP address in browser. 
It should show 13.1-RELEASE directory.


##	Using freebsd-update from other FreeBSD servers

Once nginx configuration is done and working we can change the ServerName of /etc/freebsd-update.conf file using nginx server IP, for instance:

<pre>
ServerName 192.168.1.8
</pre>

Then we can run freebsd-update from the server as usual like in https://docs.freebsd.org/en/books/handbook/cutting-edge/.
For instance, to upgrade 13.0-RELEASE server, run like:

<pre>
freebsd-update -r 13.1-RELEASE upgrade
</pre>

To fetch updates for 13.1-RELEASE server, run like:

<pre>
freebsd-update fetch
</pre>

etc.



## FreeBSD update from cron

If freebsd-update-mirror command needs to be run from cron, then it can be run like:

<pre>
./freebsd-update-mirror cron -d 2023Q1 --currently-running 13.0-RELEASE -m
</pre>

The cron command of freebsd-update is described in freebsd-update manual page as:
<pre>
     cron      Sleep a random amount of time between 1 and 3600 seconds, then
               download updates as if the fetch command was used.  If updates
               are downloaded, an email will be sent (to root or a different
               address if specified via the -t option or in the configuration
               file).  As the name suggests, this command is designed for
               running from cron(8); the random delay serves to minimize the
               probability that a large number of machines will simultaneously
               attempt to fetch updates.

</pre>
