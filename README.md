Processing of PCAP files with Snort
https://www.coresentinel.com/processing-pcap-files-snort/
May 1 2013
PCAP files are something which security and network administrators analyse on a regular basis. But how often do you process your packet capture files through an IDS engine to see what alerts it generates?

What you will learn…

This article will explain how to set up a secure drop off point for PCAP files whereby the files you upload will be automatically processed by snort without further intervention.
What you should know…

You should have a basic understanding of how Snort IDS works
You should have a reasonable understanding of CentOS
Prerequisites

Centos 6.x (in my case I am using CentOS 6.4)
A functioning Snort Installation
The lsof package which can be obtained via yum
Recommendations

A snort database within MySQL
A front end IDS interface such as Snorby
Snorts ability to process PCAP files

Wireshark and TCPdump are tools which are used widely for a variety of different purposes. Both will do complete packet captures with the ability to save to .pcap format for further analysis. I can’t remember the amount of times I have been involved in troubleshooting a connection from A to B and performed a packet capture to see what is happening with the traffic. Within linux I usually always use the following basic command syntax to execute a packet dump whilst the traffic in question traverses the interface:

# tcpdump –i eth0 –w traffic.pcap
The above command will dump all traffic from eth0 to a file in pcap format called traffic.pcap by using the –w switch. After the traffic has been captured to a pcap, I usually transfer it across to my workstation, and load it straight into Wireshark for analysis. Wireshark is great for looking at source and destination traffic, ports, and handshake information. But Wireshark has its limitations. Wireshark does not have the ability to identify suspicious traffic patterns by cross referencing traffic to an anomaly signature database such as Snort.

Recently I have gotten heavily involved in a project where we are testing the capabilities of several different IDS sensors and methods of packet capture. One of the features of the Snort command line has is its ability to not only sniff from the wire, but you can also tell it to read a pcap file and process it according to the rules in your snort.conf file. For this I would recommend creating a new snort.conf file specifically for PCAP file reads. An example of the snort syntax used to process PCAP files is as follows:

# snort -c snort_pcap.conf –r traffic.pcap
The above command will read the file traffic.pcap and process it though all of your snort rules according to your snort_pcap.conf file. Fantastic functionality right? But I needed a way to make this functionality easy to use. After all, what average system administrator is going to spend all this time transferring pcap files around and manually running snort commands on them? The function is still great however. So I came up with the idea of setting up a secure FTP file drop off point on the snort box, and using a script which automatically checks to see if a PCAP file has arrived every 10 seconds, and then processes the file if the script is not already busy processing another PCAP you sent it previously. This way all I have to do is remember to sftp my pcaps to the dropoff location, which I can do via any sftp client location.

Setting up the Secure Drop-off Point

This is relatively straight forward and is here for optional added security. All I have done is added the following configuration right at the end of /etc/ssh/sshd_config as follows:

Subsystem sftp internal-sftp
Match Group sftpsecure
    ChrootDirectory %h
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
The above tells sshd to lock down all members of the group ‘sftpsecure’ such that the users cannot redirect ports upon connection and are chrooted to their home directories. One thing you will need to know about this setup is that the root user requires to own the users home folder. This means that the user can only write or upload to a subfolder of their own home folder. Therefore the user which we will call pcap must be set up in the following way which will allow us to sftp files files remotely to the /home/pcap/filedrop directory:

groupadd sftpsecure
useradd –G sftpsecure pcap
passwd pcap
chown root:root /home/pcap
chmod 0755 /home/pcap
mkdir /home/pcap/filedrop
chown pcap:root /home/pcap/filedrop
chmod 755 /home/pcap/filedrop
For Wireshark packet captures, make sure you save the file type as a ‘Modified tcpdump’ for Snort to understand it. And remember, this solution will only process files with a .pcap extension .

 Deploying a script to process files on arrival

The following script will check to see if it is already running by using a lockfile function. If it is running it will exit to avoid overlapping processing. If it is not already running it checks to see if there are any files which have arrived in /home/pcap/filedrop. If any files have arrived, it creates a uniquely named temporary working directory. It then checks to see if the file is open by another program in case it is a large file still being written to. If the file is not still being written to, it moves it from the secure dropoff point into the temporary working directory it has created and processes it. Finally it cleans up after itself and releases the lockfile when it is finished, thereby enabling it to rerun again without overlap.

For the purpose of this example I have placed the script in a directory as follows /root/script/filedrop.sh. The next thing we need to do is set up a cron job so that the script will automatically execute every 10 seconds. As cron’s smallest time increment is 1 minute, I have had to overcome this by adding six different entries into crontab and separating them in 10 second increments by using the sleep command as follows:

# crontab -e
* * * * * /script/filedrop.sh
* * * * * /bin/sleep 10; /script/filedrop.sh
* * * * * /bin/sleep 20; /script/filedrop.sh
* * * * * /bin/sleep 30; /script/filedrop.sh
* * * * * /bin/sleep 40; /script/filedrop.sh
* * * * * /bin/sleep 50; /script/filedrop.sh
What the output looks like when processed

Now you have a secure drop off location which you can use to simply upload your .pcap files to whenever you want them checked against your snort IDS signatures. And within 10-20 seconds they will be processed. In my case I have them appearing with other traditional wire sniffing IDS sensors, so I have given the pcap reader a unique sensor name within barnyard2.

Figure 1: This is what the final result looks like in my IDS dashboard shortly after uploading pcaps:

Snorby dashboard showing snort ids alertsIf you are not using an IDS interface like Snorby, you could always add one more line to the script to email you the alert log instead. Assuming you have a mail server running and were outputting the log to /var/log/snort in alert.log format, then that line would look something like this:

/bin/mail -s "PCAP alert log" "myaddess@domain.com" </var/log/snort/alert.log
Summary

So there you have a secure FTP drop off point which you can use to simply feed .pcap files to, which will then automatically be processed through Snort IDS on arrival. I have found this functionality makes life so much easier when analyzing traffic dumps where I need the convenience to send them somewhere which automatically processes them on arrival without my intervention. Further, as it is not always practical to deploy snort IDS sensors on all of your hosts, you can now always take a packet capture from them and upload it to your drop-off point for inspection. I hope you find this solution useful.

On the Web

https://github.com/firnsy/barnyard2 – This is the barnyard2 spooler for Snort.
https://snorby.org/ – This is the Snorby front end I am using
http://www.snort.org/ – The Snort IDS system
About the author

Steven McLaughlin is an experienced information and network security professional. With both a technical and consulting background, he has been heavily involved in working with global companies developing solutions and delivering large scale projects. He also works in highly specialized teams in order to develop new ideas and patents and bring new products to market.

Originally published in “Processing of PCAP files with Snort”, Hakin9 Magazine, vol. 8, No.5, Issue 05/2013(65) ISSN: 1733-7186, May 2013

Article: https://www.coresentinel.com/processing-pcap-files-snort/

