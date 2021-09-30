Debian Tools
===
This is warehouse of scripts I use on a regular basis.  Feel free to use as you wish.

The initial load is copied from https://github.com/ChrisDeFreitas/ElasticStack/.  I'll be adding as needs arise.


# Commands and Scripts

```Bash
# free memory	
$ free -h   

# find process details by pid (with command line)
$ sudo ps -ax |grep 19414

# display top 10 cpu consumers
$ sudo ps -e --sort -pcpu -o "%cpu %mem pri f stat pid ppid class rtprio ni pri psr start user comm time tty" | head -n 10

# display top 10 memory consumers
$ sudo ps -e --sort -pmem -o "%cpu %mem pri f stat pid ppid class rtprio ni pri psr start user comm time tty" | head -n 10

# display 30 recently started apps
# note: to get startup commandline add "command" to headers argument (after tty and before last ")
$ sudo ps -e --sort -start -o "%cpu %mem pri f stat pid ppid class rtprio ni pri psr start user comm time tty" | tail -n 30  
```

```Bash
# list last 10 logins  
$ sudo last -10 -F -i

# of the last 30 logins, exclude userName  
$ sudo last -30 -F -i | grep -v userName  
```

```Bash
#Status of all services
$ sudo systemctl  
$ sudo systemctl --type=service 
$ sudo systemctl --type=service --state=active
# list startup services only (https://www.linux.com/topic/desktop/cleaning-your-linux-startup-process/)
$ sudo systemctl list-unit-files --type=service | grep enabled

#Disable service
$ sudo systemctl stop SERVICENAME
$ sudo systemctl disable SERVICENAME
```  

```Bash
#Service ports
# requires: $ sudo apt install net-tools
# better to use "ss" because pre-installed in Debian
$ sudo netstat -ltup | grep java
$ sudo netstat -ltup | grep kibana

# ss switches
# -H = hide column headers  
# -n  = do not resolve hostnames or port numbers (use 22 instead of ssh)
# -o  = display timer data
# state established = only display established connections
# exclude established = exclude established connections
# sport > :1024 =  dislay source ports above 1024
# -l, state listening = report on listening ports
# -p, --process = include pid, requires root access
# -tu, --query=inet, --tcp --udp = display data for tcp and udp connections (all equal)

# summary of port activity 
$ ss -s

# all listening internet ports with pid
$ sudo ss -ltup 

# tcp connections by port number
#   dport = destination/remote port
#   sport = source/local port
$ ss -t '( dport = :22 or sport = :22 )'

# count established connections by source port
# based on ElasticStack ports
$ ss sport = :9200 or sport = :9300  |wc -l

# count established connections by pid
# based on current pid of Kibana = 5992
$ sudo ss -tup  |grep pid=5992  |wc -l

# list processes by number of established tcp connections
$ sudo ss -Hptu |awk '{print $7}' |sort |uniq -c -w25 |sort -r
```

## login.sh
```Bash
#!/bin/bash

## login.sh
## call from ~/.profile to view at start of each login

echo 
echo ss --summary:
ss -sH

echo ss --listening --query=inet:
ss -ltuH

echo 
echo Last 10 logins with reboots etc:
last -10 -F -i -x
```

## login_remote.sh
```Bash
#!/bin/bash

## login_remote.sh
## list network stats from remote system
## assume: ssh server configured in ~/.ssh/config

remoteHost=$1

if [ -z "$remoteHost" ];
then
	#remoteHost=192.168.0.255
  echo "  Error: arg1 must be server name or ip (arg1 = $remoteHost)."
  exit -1
fi

echo Network stats for $remoteHost - 
echo 

ssh $remoteHost '\
echo ss --summary:; ss -sH; \
echo ss --listening --query=inet:; ss -ltuH; \
echo ;\
echo Last 10 logins with reboots etc:; last -10 -F -i -x ;\
'
```

## recentApps.sh
```Bash
#!/bin/bash
echo
echo 30 recently started apps:
echo
echo start time user comm tty pid ppid %cpu %mem pri class pri psr 
sudo ps -e --sort=-start -o "start time user comm tty pid ppid %cpu %mem pri class pri psr " | tail -n 30  | sort -r
```  
	
## recentApps_remote.sh
```Bash
#!/bin/bash

## recentApps_remote.sh
## list 30 recently started apps on remote system
## assume: ssh server configured in ~/.ssh/config

remoteHost=$1

if [ -z "$remoteHost" ];
then
	#remoteHost=192.168.0.255
  echo "  Error: arg1 must be server name or ip (arg1 = $remoteHost)."
  exit -1
fi

echo 30 recently started apps on $remoteHost - 
echo 

ssh -t $remoteHost '\
sudo echo ;\
echo start time user comm tty pid ppid %cpu %mem pri class pri psr  ;\
sudo ps -e --sort=-start -o "start time user comm tty pid ppid %cpu %mem pri class pri psr " | tail -n 30  | sort -r ;\
'
```  
	
## tcpUsage.sh
```Bash
#!/bin/bash

echo
echo List processes and count of established connections:
sudo ss -Hptu |awk '{print $7}' |sort |uniq -c -w25 |sort -r
```

## tcpUsage_remote.sh
```Bash
#!/bin/bash

## tcpUsage_remote.sh
## list processes and count of established connections on remote system
## assume: ssh server configured in ~/.ssh/config

remoteHost=$1

if [ -z "$remoteHost" ];
then
	#remoteHost=192.168.0.255
  echo "  Error: arg1 must be server name or ip (arg1 = $remoteHost)."
  exit -1
fi

echo List processes and count of established connections on $remoteHost - 
echo 

ssh -t $remoteHost 'sudo ss -Hptu |awk "{print $7}" |sort |uniq -c -w25 |sort -r '
```


## services_remote.sh
```Bash
#!/bin/bash

## services_remote.sh
## list enable services/daemons on remote system
## assume: ssh server configured in ~/.ssh/config

remoteHost=$1

if [ -z "$remoteHost" ];
then
        remoteHost="xfce"
  #echo "  Error: arg1 must be server name or ip (arg1 = $remoteHost)."
  #exit -1
fi

echo Enabled services on $remoteHost -
echo

ssh -t $remoteHost 'sudo systemctl list-unit-files --type=service | grep enabled'
```

## backup.sh
- backup remote folder to local folder
- then copy a local backed up folder to a different host
- use SSH, not rsync
- designed for node.js projects, so skip node_modules and "." folders
```Bash
#!/bin/bash
#
# gjbackup.sh: GuitarJoe backup
#
# 1. copy files from remote folder, $HOST:$SRC, to local folder with date stamp, $DEST/$DATE
# 2. copy local folder contents, $PUB/*, to remote folder, $RHOST:$RDEST
# assume: ssh config is used
# note: create local folders if missing
# note: . folders (like .git) are automatically excluded due to "ssh ... ls $SRC"
#
PROGNAME=${0##*/}
VERSION="0.1"


HOST=chris@estack
#SRC=~/tmp/gateway
SRC=/var/www/guitarjoe

DEST=~/tmp/tmp/guitarjoe		#backup root folder
#DEST=/cygdrive/c/Users/chris/Backups/estack/$date

DATE=$(date +"%Y%m%d")					# backup folder name
#DATE=$(date +"%Y%m%d-%H%M%S")	# include time


#PUB=""
PUB=$DEST/$DATE/docs
RHOST=chris@xfce
RDEST=/var/www/html/guitarjoe



echo
echo Start $PROGNAME v$VERSION: $(date)
echo


FILES=$(ssh $HOST ls "$SRC")
#echo Found: $FILES


if [ -d "$DEST" ]; then
	echo DEST Exists: $DEST
else
	mkdir "$DEST"
	echo DEST Created: $DEST
fi
if [ -d "$DEST"/$DATE ]; then
	echo DEST/DATE Exists: $DEST/$DATE/
else
	mkdir "$DEST"/$DATE/
	echo DEST/DATE Created: $DEST/$DATE/
fi


for FILE in $FILES; do
  if [[ "$FILE" == "node_modules" || "$FILE" == ".git" ]]; then
    echo skip $FILE
    continue
  fi

	echo Copying $SRC/$FILE...
	scp -Cpr $HOST:"$SRC"/"$FILE" "$DEST"/$DATE/
done


if [[ -z "$PUB" ]]; then
	echo PUB is empty -- nothing to upload
else
	if [[ -e "$PUB" ]]; then
		echo Uploading [$PUB/*] to [$RHOST:"$RDEST"]...
		scp -Cpr "$PUB"/* $RHOST:"$RDEST"
	else
		echo Error, $PUB does not exist
	fi
fi


echo
echo Done $PROGNAME v$VERSION: $(date)
```
