# weewx
weewx weather station software - weewx.napervilleweather.net

# Background
I have been running weewx weather station software since 2018 in a centos7 VM under virtualbox.  It has been working perfectly.  Now it is time to port it into my kubernetes clusters.  Here's my notes.

# Do it yourself
I was able to install weewx in a Centos7 container using the following:
```
[jkozik@weewx weewx]$ pwd
/home/jkozik/weewx
[jkozik@weewx weewx]$ ls
customizeSettings.sh  Dockerfile  install-input.txt
[jkozik@weewx weewx]$ cat Dockerfile
FROM centos:7
RUN yum -y update && yum -y install python3 wget && \
    pip3 install configobj pillow pyserial pyusb cheetah3
RUN wget https://weewx.com/downloads/weewx-4.5.1.tar.gz && \
    tar xvfz weewx-4.5.1.tar.gz
WORKDIR /weewx-4.5.1

COPY customizeSettings.sh /weewx-4.5.1
COPY install-input.txt /weewx-4.5.1
RUN python3 ./setup.py build
RUN python3 ./setup.py install < install-input.txt

RUN     chmod +x customizeSettings.sh && pwd
RUN  /weewx-4.5.1/customizeSettings.sh

[jkozik@weewx weewx]$ cat install-input.txt
Naperville, IL
702, foot
41.7900009
-88.1200027
n
us
6
serial
/dev/ttyUSB0

[jkozik@weewx weewx]$ cat customizeSettings.sh
echo "Customize weewx.conf"
WEEWXCONF=/home/weewx/weewx.conf

sed -i -e '/station_url =/ c\
    station_url = weewx.napervilleweather.net' $WEEWXCONF

[jkozik@weewx weewx]$
```
This wasn't that hard, but I did have to get some tips from some other weewx repositories:
- [instantlinux/docker-tools](https://github.com/instantlinux/docker-tools) and 
- [tomdotorg/docker-weewx](https://github.com/tomdotorg/docker-weewx)

## Rolling logger errors
But when I tried to run it, I got rolling errors pointing to a missing logger function.
```
[root@d53b8cd9fb22 weewx]# ./bin/weewxd 2>&1 | more
--- Logging error ---
Traceback (most recent call last):
  File "/usr/lib64/python3.6/logging/handlers.py", line 936, in emit
    self.socket.send(msg)
OSError: [Errno 9] Bad file descriptor

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/lib64/python3.6/logging/handlers.py", line 857, in _connect_unixsocket
    self.socket.connect(address)
FileNotFoundError: [Errno 2] No such file or directory

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/lib64/python3.6/logging/handlers.py", line 939, in emit
    self._connect_unixsocket(self.address)
  File "/usr/lib64/python3.6/logging/handlers.py", line 868, in _connect_unixsocket
    self.socket.connect(address)
FileNotFoundError: [Errno 2] No such file or directory
Call stack:
  File "./bin/weewxd", line 264, in <module>
    main()
  File "./bin/weewxd", line 88, in main
    log.info("Initializing weewx version %s", weewx.__version__)
Message: 'Initializing weewx version %s'
Arguments: ('4.5.1',)
--- Logging error ---
Traceback (most recent call last):
...
```
I found a lengthy thread on this subject:
- [[weewx-development] weewx v4 need for syslog daemon](https://www.mail-archive.com/weewx-development@googlegroups.com/msg02573.html)

And the author of the thread posted a work-around on his repository: [vinceskahan/weewx-docker](https://github.com/vinceskahan/weewx-docker)


