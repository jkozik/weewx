# Configure weewx weather software for my Davis Vantage Pro2
weewx weather station software - weewx.napervilleweather.net

## Background
I have been running weewx weather station software since 2018 in a centos7 VM under virtualbox.  It has been working perfectly.  Now it is time to port it into my kubernetes clusters.  In this repository, I have a backup of the weewx.sdb file that is holding 3 years of weather telemetry data. 

Here's my notes.

# Deploy tomdotorg/docker-weewx's repository in my kubernetes cluster
The [tomdotorg/docker-weewx](https://github.com/tomdotorg/docker-weewx) repository has a weewx image that can be configured to run in kubernetes.  My setup:
- [Davis Vantage Pro2 weather station](https://www.davisinstruments.com/pages/vantage-pro2)
- Davis [Wireless Weather Envoy](https://www.davisinstruments.com/products/wireless-weather-envoy?pr_prod_strat=description&pr_rec_pid=6661299241121&pr_ref_pid=6661301665953&pr_seq=uniform) USB data logger
- Kubernetes cluster spread across two dell servers. 

The Weather station transmits weather telemetry to the Envoy.  The Envoy receives it and relays the data over a USB connection.  My kubernetes node called kworker1 has a USB port defined, called /dev/ttyUSB0.  Only node kworker1 can see this port. The deployment file as a node selector setup to enforce this. 

The weewx application requires a configuration file, weewx.conf.  I customize that file to fit my setup.  The weewx application archives all of the weather data in a SQLite database, weewx.sdb. I choose to put these files on external PersistantVolumes.  I want these files to persist even if the weewx appliation gets trashed, reset or upgraded.  

## Setup weewx user and directory 
I want to make to Persistent Volumes using NFS.  To help self document this, I am setting up a weewx user, setting up a simple directory structure and sharing two of the directories in NFS.
```
[root@dell2 ~]# tree ~weewx
/home/weewx
|
└── nfs
    ├── archive
    │   |
    │   └── weewx.sdb
    └── conf
        └── weewx.conf
        
[root@dell2 ~]# cat /etc/exports # add the following two lines to nfs exports file
[...]
/home/weewx/nfs/conf 192.168.100.0/24(rw,sync,no_root_squash,no_all_squash,no_subtree_check,insecure)
/home/weewx/nfs/archive 192.168.100.0/24(rw,sync,no_root_squash,no_all_squash,no_subtree_check,insecure)

[root@dell2 ~]# service nfs-server restart
Redirecting to /bin/systemctl restart nfs-server.service
[root@dell2 ~]#
```

These are both made into PVs and PVCs.
```
[jkozik@dell2 weewx]$ kubectl apply -f weewx-conf-archive-pv.yaml
persistentvolume/weewx-conf created
persistentvolume/weewx-archive created
[jkozik@dell2 weewx]$ kubectl apply -f weewx-conf-archive-pvc.yaml
persistentvolumeclaim/weewx-conf created
persistentvolumeclaim/weewx-archive created
[jkozik@dell2 weewx]$ kubectl get pv,pvc
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                  STORAGECLASS   REASON   AGE
persistentvolume/weewx-archive                              1Gi        RWO            Retain           Bound    default/weewx-archive                  nfs                     142m
persistentvolume/weewx-conf                                 1Gi        RWO            Retain           Bound    default/weewx-conf                     nfs                     142m

NAME                                                 STATUS    VOLUME                         CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/weewx-archive                  Bound     weewx-archive                  1Gi        RWO            nfs            142m
persistentvolumeclaim/weewx-conf                     Bound     weewx-conf                     1Gi        RWO            nfs            142m
```
The weewx server scans the stream from the USB data logger.  It saves the data into the weewx.sdb and every 10 minutes or so generates a set of images and html files. This runs in a container with volume mounts pointing the the configuration files and stores the weewx.sdb in an archive volume. Note also, this container gets access to the /dev/ttyUSB0 serial port. I have no source code for this; I am just using the image [mitct02/weewx](https://hub.docker.com/r/mitct02/weewx) image. 

Also in this POD there is an apache web server that renders the images and htmls files generates by the weewx server. Again, I have no source for this, I am just pulling in the apache [httpd](https://hub.docker.com/_/httpd) image.  Please look at the weewx-deployment.yaml file for details.  
```
[jkozik@dell2 weewx]$ kubectl apply -f weewx-deploy.yaml
deployment.apps/weewx created

[jkozik@dell2 weewx]$ kubectl get pods
NAME                                               READY   STATUS      RESTARTS   AGE
weewx-85ddcf7954-5jd9v                             2/2     Running     0          72m

[jkozik@dell2 weewx]$ kubectl exec -it weewx-85ddcf7954-5jd9v  -c weewx -- /bin/bash
root@weewx-85ddcf7954-5jd9v:/# cd /home/weewx
root@weewx-85ddcf7954-5jd9v:~# ls
archive  bin  conf  docs  examples  LICENSE.txt  public_html  README.md  skins  tmp  util  weewx.conf  weewx.conf.4.5.1
root@weewx-85ddcf7954-5jd9v:/# cd /home/weewx/public_html
root@weewx-85ddcf7954-5jd9v:~/public_html# ls
celestial.html    daytempfeel.png  favicon.ico         monthrx.png        monthwind.png     telemetry.html     weektempfeel.png  yearbarometer.png  yeartempin.png
daybarometer.png  daytempin.png    font                monthtempdew.png   monthwindvec.png  weekbarometer.png  weektempin.png    yearhumin.png      yeartemp.png
dayhumin.png      daytemp.png      index.html          monthtempfeel.png  NOAA              weekhumin.png      weektemp.png      yearhum.png        yearuv.png
dayhum.png        dayuv.png        monthbarometer.png  monthtempin.png    rss.xml           weekhum.png        weekuv.png        yearradiation.png  yearvolt.png
dayradiation.png  dayvolt.png      monthhumin.png      monthtemp.png      seasons.css       weekradiation.png  weekvolt.png      yearrain.png       yearwinddir.png
dayrain.png       daywinddir.png   monthhum.png        monthuv.png        seasons.js        weekrain.png       weekwinddir.png   yearrx.png         yearwind.png
dayrx.png         daywind.png      monthradiation.png  monthvolt.png      statistics.html   weekrx.png         weekwind.png      yeartempdew.png    yearwindvec.png
daytempdew.png    daywindvec.png   monthrain.png       monthwinddir.png   tabular.html      weektempdew.png    weekwindvec.png   yeartempfeel.png
root@weewx-85ddcf7954-5jd9v:~/public_html#
root@weewx-85ddcf7954-5jd9v:~# cd archive
root@weewx-85ddcf7954-5jd9v:~/archive# ls
bckweewx.sdb092021  weewx.sdb
root@weewx-85ddcf7954-5jd9v:~/archive# exit
exit
```
Note: from above, the weewx pod has two containers.  Exec into the weewx container and verify that the public_html directory gets populated.  It takes a few minutes to generate. 

## Apply Service and Ingress
And finally, a service needs to be configured to expose the web port from the apache httpd.  Then an ingress needs to be applyed to map this service to the URL http://weewx.napervilleweather.net.  
```
[jkozik@dell2 weewx]$ kubectl apply -f weewx-svc.yaml -f weewx-ingress-tls.yaml
service/weewx unchanged
ingress.networking.k8s.io/weewx-ingress configured
```

# Here's notes from when I tried to "Do it yourself"
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

# Use image from tomdotorg
Rather than trying to create my own weewx image, I discovered that there are a few images that I can use straight from the dockerhub repository. I focused on [tomdotorg
/
docker-weewx](https://github.com/tomdotorg/docker-weewx)

The following worked for me:
```
[jkozik@weewx weewx]$ docker run -d --name weewxtdo --device=/dev/ttyUSB0 --volume /home/jkozik/weewx/weewx.conf:/home/weewx/weewx.conf mitct02/weewx:4.5.1
[jkozik@weewx weewx]$ docker logs weewxtdo| more
*** Running /etc/my_init.d/00_regen_ssh_host_keys.sh...
*** Running /etc/my_init.d/10_syslog-ng.init...
*** Booting runit daemon...
*** Runit started as PID 22
Sep 18 14:58:28 ec6a73630431 syslog-ng[14]: syslog-ng starting up; version='3.13.2'
using
Sep 18 14:58:29 ec6a73630431 weewx[29] INFO __main__: Initializing weewx version 4.5.1
Sep 18 14:58:29 ec6a73630431 weewx[29] INFO __main__: Using Python 3.6.9 (default, Jan 26 2021, 15:33:00)
[GCC 8.4.0]
Sep 18 14:58:29 ec6a73630431 weewx[29] INFO __main__: Platform Linux-3.10.0-1160.21.1.el7.x86_64-x86_64-with-Ubuntu-18.04-bionic
Sep 18 14:58:29 ec6a73630431 weewx[29] INFO __main__: Locale is 'en_US.UTF-8'
Sep 18 14:58:29 ec6a73630431 weewx[29] INFO __main__: Using configuration file /home/weewx/weewx.conf
Sep 18 14:58:29 ec6a73630431 weewx[29] INFO __main__: Debug is 0
Sep 18 14:58:29 ec6a73630431 weewx[29] INFO weewx.engine: Loading station type Vantage (weewx.drivers.vantage)
Sep 18 14:58:29 ec6a73630431 cron[26]: (CRON) INFO (pidfile fd = 3)
Sep 18 14:58:29 ec6a73630431 cron[26]: (CRON) INFO (Running @reboot jobs)
Sep 18 14:58:30 ec6a73630431 weewx[29] INFO weewx.engine: StdConvert target unit is 0x1
Sep 18 14:58:30 ec6a73630431 weewx[29] INFO weewx.manager: Created and initialized table 'archive' in database 'weewx.sdb'
Sep 18 14:58:30 ec6a73630431 weewx[29] INFO weewx.manager: Created daily summary tables
Sep 18 14:58:30 ec6a73630431 weewx[29] INFO weewx.engine: Archive will use data binding wx_binding
Sep 18 14:58:30 ec6a73630431 weewx[29] INFO weewx.engine: Record generation will be attempted in 'hardware'
Sep 18 14:58:30 ec6a73630431 weewx[29] ERROR weewx.engine: The archive interval in the configuration file (300) does not match the station hardware interval (600).
Sep 18 14:58:30 ec6a73630431 weewx[29] INFO weewx.engine: Using archive interval of 600 seconds (specified by hardware)
Sep 18 14:58:30 ec6a73630431 weewx[29] INFO weewx.restx: StationRegistry: Registration not requested.
Sep 18 14:58:30 ec6a73630431 weewx[29] INFO weewx.restx: Wunderground: Posting not enabled.
Sep 18 14:58:30 ec6a73630431 weewx[29] INFO weewx.restx: PWSweather: Posting not enabled.
Sep 18 14:58:30 ec6a73630431 weewx[29] INFO weewx.restx: CWOP: Posting not enabled.
Sep 18 14:58:30 ec6a73630431 weewx[29] INFO weewx.restx: WOW: Posting not enabled.
Sep 18 14:58:30 ec6a73630431 weewx[29] INFO weewx.restx: AWEKAS: Posting not enabled.
Sep 18 14:58:30 ec6a73630431 weewx[29] INFO __main__: Starting up weewx version 4.5.1
Sep 18 14:58:30 ec6a73630431 weewx[29] INFO weewx.engine: Clock error is -3598.31 seconds (positive is fast)
Sep 18 14:58:30 ec6a73630431 weewx[29] INFO weewx.drivers.vantage: Clock set to 2021-09-18 14:58:31 EDT (1631991511)
Sep 18 14:58:30 ec6a73630431 weewx[29] INFO weewx.engine: Using binding 'wx_binding' to database 'weewx.sdb'
Sep 18 14:58:30 ec6a73630431 weewx[29] INFO weewx.manager: Starting backfill of daily summaries
Sep 18 14:58:30 ec6a73630431 weewx[29] INFO weewx.manager: Empty database
Sep 18 14:58:32 ec6a73630431 weewx[29] INFO weewx.manager: Added record 2021-08-31 19:20:00 EDT (1630452000) to database 'weewx.sdb'
Sep 18 14:58:32 ec6a73630431 weewx[29] INFO weewx.manager: Added record 2021-08-31 19:20:00 EDT (1630452000) to daily summary in 'weewx.sdb'
Sep 18 14:58:32 ec6a73630431 weewx[29] INFO weewx.manager: Added record 2021-08-31 19:30:00 EDT (1630452600) to database 'weewx.sdb'
Sep 18 14:58:32 ec6a73630431 weewx[29] INFO weewx.manager: Added record 2021-08-31 19:30:00 EDT (1630452600) to daily summary in 'weewx.sdb'
Sep 18 14:58:32 ec6a73630431 weewx[29] INFO weewx.manager: Added record 2021-08-31 19:40:00 EDT (1630453200) to database 'weewx.sdb'
Sep 18 14:58:32 ec6a73630431 weewx[29] INFO weewx.manager: Added record 2021-08-31 19:40:00 EDT (1630453200) to daily summary in 'weewx.sdb'
Sep 18 14:58:32 ec6a73630431 weewx[29] INFO weewx.manager: Added record 2021-08-31 19:50:00 EDT (1630453800) to database 'weewx.sdb'
Sep 18 14:58:32 ec6a73630431 weewx[29] INFO weewx.manager: Added record 2021-08-31 19:50:00 EDT (1630453800) to daily summary in 'weewx.sdb'
Sep 18 14:58:32 ec6a73630431 weewx[29] INFO weewx.manager: Added record 2021-08-31 20:00:00 EDT (1630454400) to database 'weewx.sdb'
Sep 18 14:58:32 ec6a73630431 weewx[29] INFO weewx.manager: Added record 2021-08-31 20:00:00 EDT (1630454400) to daily summary in 'weewx.sdb'
Sep 18 14:58:32 ec6a73630431 weewx[29] INFO weewx.manager: Added record 2021-08-31 20:10:00 EDT (1630455000) to database 'weewx.sdb'
Sep 18 14:58:32 ec6a73630431 weewx[29] INFO weewx.manager: Added record 2021-08-31 20:10:00 EDT (1630455000) to daily summary in 'weewx.sdb'
Sep 18 14:58:32 ec6a73630431 weewx[29] INFO weewx.manager: Added record 2021-08-31 20:20:00 EDT (1630455600) to database 'weewx.sdb'
Sep 18 14:58:32 ec6a73630431 weewx[29] INFO weewx.manager: Added record 2021-08-31 20:20:00 EDT (1630455600) to daily summary in 'weewx.sdb'
Sep 18 14:58:32 ec6a73630431 weewx[29] INFO weewx.manager: Added record 2021-08-31 20:30:00 EDT (1630456200) to database 'weewx.sdb'
Sep 18 14:58:32 ec6a73630431 weewx[29] INFO weewx.manager: Added record 2021-08-31 20:30:00 EDT (1630456200) to daily summary in 'weewx.sdb'
Sep 18 14:58:32 ec6a73630431 weewx[29] INFO weewx.manager: Added record 2021-08-31 20:40:00 EDT (1630456800) to database 'weewx.sdb'
Sep 18 14:58:32 ec6a73630431 weewx[29] INFO weewx.manager: Added record 2021-08-31 20:40:00 EDT (1630456800) to daily summary in 'weewx.sdb'
Sep 18 14:58:32 ec6a73630431 weewx[29] INFO weewx.manager: Added record 2021-08-31 20:50:00 EDT (1630457400) to database 'weewx.sdb'
Sep 18 14:58:32 ec6a73630431 weewx[29] INFO weewx.manager: Added record 2021-08-31 20:50:00 EDT (1630457400) to daily summary in 'weewx.sdb'
Sep 18 14:58:32 ec6a73630431 weewx[29] INFO weewx.manager: Added record 2021-08-31 21:00:00 EDT (1630458000) to database 'weewx.sdb'
Sep 18 14:58:32 ec6a73630431 weewx[29] INFO weewx.manager: Added record 2021-08-31 21:00:00 EDT (1630458000) to daily summary in 'weewx.sdb'
REC:    2021-08-31 19:20:00 EDT (1630452000) altimeter: None, appTemp: 79.3725275155127, barometer: 29.788, cloudbase: 4275.124104653878, consBatteryVoltage: None, dateTime: 1630452000, dewpoi
nt: 60.52585393952294, ET: 0.0, forecastRule: 192, heatindex: 75.96300000000001, highOutTemp: 76.0, highRadiation: 0.0, highUV: 0.0, humidex: 84.01621446348804, inDewpoint: 53.1716602790168, i
nHumidity: 55.0, inTemp: 70.1, interval: 10, lowOutTemp: 75.7, maxSolarRad: 94.7508724517392, outHumidity: 59.0, outTemp: 75.9, pressure: None, radiation: 0.0, rain: 0.0, rainRate: 0.0, rxChec
kPercent: 92.25, txBatteryStatus: None, usUnits: 1, UV: 0.0, windchill: 75.9, windDir: None, windGust: 1.0, windGustDir: 67.5, windrun: 0.0, wind_samples: 216.0, windSpeed: 0.0
REC:    2021-08-31 19:30:00 EDT (1630452600) altimeter: None, appTemp: 79.50397560207841, barometer: 29.79, cloudbase: 3952.429721280259, consBatteryVoltage: None, dateTime: 1630452600, dewpoi
nt: 61.645709226366854, ET: 0.0, forecastRule: 192, heatindex: 75.774, highOutTemp: 75.7, highRadiation: 0.0, highUV: 0.0, humidex: 84.46060928531574, inDewpoint: 53.26452553650931, inHumidity
: 55.0, inTemp: 70.2, interval: 10, lowOutTemp: 75.4, maxSolarRad: 66.36687714846124, outHumidity: 62.0, outTemp: 75.6, pressure: None, radiation: 0.0, rain: 0.0, rainRate: 0.0, rxCheckPercent
: 90.96875, txBatteryStatus: None, usUnits: 1, UV: 0.0, windchill: 75.6, windDir: None, windGust: 4.0, windGustDir: 225.0, windrun: 0.0, wind_samples: 213.0, windSpeed: 0.0
```
More generally, the following sets up weewx exposing a public_html file
```
[jkozik@weewx weewx]$ docker run -d --name weewxtdo --device=/dev/ttyUSB0 --volume /home/jkozik/weewx/weewx.conf:/home/weewx/weewx.conf --volume /tmp/public_html:/home/weewx/public_html  --volume /home/jkozik/weewx/archive:/home/weewx/archive mitct02/weewx:4.5.1
23b804446512585992ba35e73b23717bca0cafb080a62e908ff5c03ebdd7cf6f

[jkozik@weewx weewx]$ docker logs weewxtdo| more
*** Running /etc/my_init.d/00_regen_ssh_host_keys.sh...
*** Running /etc/my_init.d/10_syslog-ng.init...
*** Booting runit daemon...
*** Runit started as PID 22
Sep 18 17:59:13 23b804446512 syslog-ng[14]: syslog-ng starting up; version='3.13.2'
using
Sep 18 17:59:15 23b804446512 weewx[29] INFO __main__: Initializing weewx version 4.5.1
Sep 18 17:59:15 23b804446512 weewx[29] INFO __main__: Using Python 3.6.9 (default, Jan 26 2021, 15:33:00)
[GCC 8.4.0]
Sep 18 17:59:15 23b804446512 weewx[29] INFO __main__: Platform Linux-3.10.0-1160.21.1.el7.x86_64-x86_64-with-Ubuntu-18.04-bionic
Sep 18 17:59:15 23b804446512 weewx[29] INFO __main__: Locale is 'en_US.UTF-8'
Sep 18 17:59:15 23b804446512 weewx[29] INFO __main__: Using configuration file /home/weewx/weewx.conf
Sep 18 17:59:15 23b804446512 weewx[29] INFO __main__: Debug is 0
Sep 18 17:59:15 23b804446512 weewx[29] INFO weewx.engine: Loading station type Vantage (weewx.drivers.vantage)
Sep 18 17:59:15 23b804446512 cron[26]: (CRON) INFO (pidfile fd = 3)
Sep 18 17:59:15 23b804446512 cron[26]: (CRON) INFO (Running @reboot jobs)
Sep 18 17:59:20 23b804446512 weewx[29] INFO weewx.engine: StdConvert target unit is 0x1
Sep 18 17:59:20 23b804446512 weewx[29] INFO weewx.manager: Created and initialized table 'archive' in database 'weewx.sdb'
Sep 18 17:59:20 23b804446512 weewx[29] INFO weewx.manager: Created daily summary tables
Sep 18 17:59:20 23b804446512 weewx[29] INFO weewx.engine: Archive will use data binding wx_binding
Sep 18 17:59:20 23b804446512 weewx[29] INFO weewx.engine: Record generation will be attempted in 'hardware'
Sep 18 17:59:20 23b804446512 weewx[29] ERROR weewx.engine: The archive interval in the configuration file (300) does not match the station hardware interval (600).
Sep 18 17:59:20 23b804446512 weewx[29] INFO weewx.engine: Using archive interval of 600 seconds (specified by hardware)
Sep 18 17:59:20 23b804446512 weewx[29] INFO weewx.restx: StationRegistry: Registration not requested.
Sep 18 17:59:20 23b804446512 weewx[29] INFO weewx.restx: Wunderground: Posting not enabled.
Sep 18 17:59:20 23b804446512 weewx[29] INFO weewx.restx: PWSweather: Posting not enabled.
Sep 18 17:59:20 23b804446512 weewx[29] INFO weewx.restx: CWOP: Posting not enabled.
Sep 18 17:59:20 23b804446512 weewx[29] INFO weewx.restx: WOW: Posting not enabled.
Sep 18 17:59:20 23b804446512 weewx[29] INFO weewx.restx: AWEKAS: Posting not enabled.
Sep 18 17:59:20 23b804446512 weewx[29] INFO __main__: Starting up weewx version 4.5.1
Sep 18 17:59:20 23b804446512 weewx[29] INFO weewx.engine: Clock error is -30.92 seconds (positive is fast)
Sep 18 17:59:20 23b804446512 weewx[29] INFO weewx.drivers.vantage: Clock set to 2021-09-18 17:59:21 EDT (1632002361)
Sep 18 17:59:20 23b804446512 weewx[29] INFO weewx.engine: Using binding 'wx_binding' to database 'weewx.sdb'
Sep 18 17:59:20 23b804446512 weewx[29] INFO weewx.manager: Starting backfill of daily summaries
Sep 18 17:59:20 23b804446512 weewx[29] INFO weewx.manager: Empty database
Sep 18 17:59:22 23b804446512 weewx[29] INFO weewx.manager: Added record 2021-08-31 22:20:00 EDT (1630462800) to database 'weewx.sdb'
Sep 18 17:59:22 23b804446512 weewx[29] INFO weewx.manager: Added record 2021-08-31 22:20:00 EDT (1630462800) to daily summary in 'weewx.sdb'
Sep 18 17:59:22 23b804446512 weewx[29] INFO weewx.manager: Added record 2021-08-31 22:30:00 EDT (1630463400) to database 'weewx.sdb'
Sep 18 17:59:22 23b804446512 weewx[29] INFO weewx.manager: Added record 2021-08-31 22:30:00 EDT (1630463400) to daily summary in 'weewx.sdb'
Sep 18 17:59:22 23b804446512 weewx[29] INFO weewx.manager: Added record 2021-08-31 22:40:00 EDT (1630464000) to database 'weewx.sdb'
Sep 18 17:59:22 23b804446512 weewx[29] INFO weewx.manager: Added record 2021-08-31 22:40:00 EDT (1630464000) to daily summary in 'weewx.sdb'
Sep 18 17:59:22 23b804446512 weewx[29] INFO weewx.manager: Added record 2021-08-31 22:50:00 EDT (1630464600) to database 'weewx.sdb'
Sep 18 17:59:22 23b804446512 weewx[29] INFO weewx.manager: Added record 2021-08-31 22:50:00 EDT (1630464600) to daily summary in 'weewx.sdb'
Sep 18 17:59:22 23b804446512 weewx[29] INFO weewx.manager: Added record 2021-08-31 23:00:00 EDT (1630465200) to database 'weewx.sdb'
Sep 18 17:59:22 23b804446512 weewx[29] INFO weewx.manager: Added record 2021-08-31 23:00:00 EDT (1630465200) to daily summary in 'weewx.sdb'
Sep 18 17:59:22 23b804446512 weewx[29] INFO weewx.manager: Added record 2021-08-31 23:10:00 EDT (1630465800) to database 'weewx.sdb'
Sep 18 17:59:22 23b804446512 weewx[29] INFO weewx.manager: Added record 2021-08-31 23:10:00 EDT (1630465800) to daily summary in 'weewx.sdb'
Sep 18 17:59:22 23b804446512 weewx[29] INFO weewx.manager: Added record 2021-08-31 23:20:00 EDT (1630466400) to database 'weewx.sdb'
Sep 18 17:59:22 23b804446512 weewx[29] INFO weewx.manager: Added record 2021-08-31 23:20:00 EDT (1630466400) to daily summary in 'weewx.sdb'
Sep 18 17:59:22 23b804446512 weewx[29] INFO weewx.manager: Added record 2021-08-31 23:30:00 EDT (1630467000) to database 'weewx.sdb'
Sep 18 17:59:22 23b804446512 weewx[29] INFO weewx.manager: Added record 2021-08-31 23:30:00 EDT (1630467000) to daily summary in 'weewx.sdb'
Sep 18 17:59:22 23b804446512 weewx[29] INFO weewx.manager: Added record 2021-08-31 23:40:00 EDT (1630467600) to database 'weewx.sdb'
Sep 18 17:59:22 23b804446512 weewx[29] INFO weewx.manager: Added record 2021-08-31 23:40:00 EDT (1630467600) to daily summary in 'weewx.sdb'
Sep 18 17:59:23 23b804446512 weewx[29] INFO weewx.manager: Added record 2021-08-31 23:50:00 EDT (1630468200) to database 'weewx.sdb'
Sep 18 17:59:23 23b804446512 weewx[29] INFO weewx.manager: Added record 2021-08-31 23:50:00 EDT (1630468200) to daily summary in 'weewx.sdb'
Sep 18 17:59:23 23b804446512 weewx[29] INFO weewx.manager: Added record 2021-09-01 00:00:00 EDT (1630468800) to database 'weewx.sdb'
Sep 18 17:59:23 23b804446512 weewx[29] INFO weewx.manager: Added record 2021-09-01 00:00:00 EDT (1630468800) to daily summary in 'weewx.sdb'
Unable to wake up console... sleeping
Unable to wake up console... retrying
REC:    2021-08-31 22:20:00 EDT (1630462800) altimeter: None, appTemp: 76.63108596774251, barometer: 29.833, cloudbase: 2341.3781612084267, consBatteryVoltage: None, dateTime: 1630462800, dewp
oint: 64.63433609068292, ET: 0.0, forecastRule: 75, heatindex: 72.06299999999999, highOutTemp: 71.5, highRadiation: 0.0, highUV: 0.0, humidex: 82.48273913687328, inDewpoint: 53.26452553650931,
 inHumidity: 55.0, inTemp: 70.2, interval: 10, lowOutTemp: 71.4, maxSolarRad: 0.0, outHumidity: 79.0, outTemp: 71.5, pressure: None, radiation: 0.0, rain: 0.0, rainRate: 0.0, rxCheckPercent: 8
9.26041666666666, txBatteryStatus: None, usUnits: 1, UV: 0.0, windchill: 71.5, windDir: None, windGust: 1.0, windGustDir: 67.5, windrun: 0.0, wind_samples: 209.0, windSpeed: 0.0
REC:    2021-08-31 22:30:00 EDT (1630463400) altimeter: None, appTemp: 76.77286265469326, barometer: 29.833, cloudbase: 2342.041067322306, consBatteryVoltage: None, dateTime: 1630463400, dewpo
int: 64.73141930378185, ET: 0.0, forecastRule: 75, heatindex: 72.17299999999999, highOutTemp: 71.6, highRadiation: 0.0, highUV: 0.0, humidex: 82.6551123683511, inDewpoint: 53.26452553650931, i
nHumidity: 55.0, inTemp: 70.2, interval: 10, lowOutTemp: 71.5, maxSolarRad: 0.0, outHumidity: 79.0, outTemp: 71.6, pressure: None, radiation: 0.0, rain: 0.0, rainRate: 0.0, rxCheckPercent: 91.
82291666666666, txBatteryStatus: None, usUnits: 1, UV: 0.0, windchill: 71.6, windDir: None, windGust: 1.0, windGustDir: 315.0, windrun: 0.0, wind_samples: 215.0, windSpeed: 0.0
...
```
After a few minutes, the public_html directory gets populated with weewx web weather content
```
[jkozik@weewx weewx]$ docker exec -it weewxtdo /bin/bash
root@23b804446512:/# cd /home/weewx
root@23b804446512:~# ls
archive  bin  docs  examples  LICENSE.txt  public_html  README.md  skins  tmp  util  weewx.conf  weewx.conf.4.5.1
root@23b804446512:~# cd public_html
root@23b804446512:~/public_html# ls
root@23b804446512:~/public_html# ls
...
root@23b804446512:~# ls public_html
celestial.html    daytempfeel.png  favicon.ico         monthrx.png        monthwind.png     telemetry.html     weektempfeel.png  yearbarometer.png  yeartempin.png
daybarometer.png  daytempin.png    font                monthtempdew.png   monthwindvec.png  weekbarometer.png  weektempin.png    yearhumin.png      yeartemp.png
dayhumin.png      daytemp.png      index.html          monthtempfeel.png  NOAA              weekhumin.png      weektemp.png      yearhum.png        yearuv.png
dayhum.png        dayuv.png        monthbarometer.png  monthtempin.png    rss.xml           weekhum.png        weekuv.png        yearradiation.png  yearvolt.png
dayradiation.png  dayvolt.png      monthhumin.png      monthtemp.png      seasons.css       weekradiation.png  weekvolt.png      yearrain.png       yearwinddir.png
dayrain.png       daywinddir.png   monthhum.png        monthuv.png        seasons.js        weekrain.png       weekwinddir.png   yearrx.png         yearwind.png
dayrx.png         daywind.png      monthradiation.png  monthvolt.png      statistics.html   weekrx.png         weekwind.png      yeartempdew.png    yearwindvec.png
daytempdew.png    daywindvec.png   monthrain.png       monthwinddir.png   tabular.html      weektempdew.png    weekwindvec.png   yeartempfeel.png
```
Here's another container that serves the previously generaged content on a web server
```
[jkozik@weewx weewx]$ docker run -d --name=weewx-webserver --restart=always -p 8080:80 -v /tmp/public_html:/usr/local/apache2/htdocs httpd
Unable to find image 'httpd:latest' locally
latest: Pulling from library/httpd
a330b6cecb98: Pull complete
14e3dd65f04d: Pull complete
fe59ad2e7efe: Pull complete
2cb26220caa8: Pull complete
3138742bd847: Pull complete
Digest: sha256:af1199cd77b018781e2610923f15e8a58ce22941b42ce63a6ae8b6e282af79f5
Status: Downloaded newer image for httpd:latest
5b9b1d3f1f43efc32ac9026fae3148c1f161bc7c9ca90f53655b6b1cbed4631c
[jkozik@weewx weewx]$
```
The weewx web page is on http://192.168.100.170:8080

## USB
The definitive article on how to get the /dev/ttyUSB0 visible to docker containers: 
- [Docker - a way to give access to a host USB or serial device?](https://stackoverflow.com/questions/24225647/docker-a-way-to-give-access-to-a-host-usb-or-serial-device)

I am running a Davis Envoy USB data logger.  It is plugged into my Dell Server.  My containers are running in virtualbox VMs configured as nodes in a kubernete3s cluster.  USB is a little tricky.  Also, since my nodes span multiple machines, I need to label the node and use a node-selector.




