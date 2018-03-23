# James_Isaac_server challenge

The following document describes the setting up of an Apache James Server to relay email to external email address. The configuration requires two AWS instances within a VPC, where the Apache James Server could only relay email (i.e. whitelisted) from the other AWS instance.

The configuration steps are:

## AWS Configuration 

### Create VPC - Ensure that:
 * an Internet Gateway is configured
 * the subnets `Auto-assign` Public IP4 addresses
 * the route on the Route Table includes an entry for `Destination` "0.0.0.0/0" to `Target` "Internet Gateway"
 * DNS Hostnames and DNS Resolution are both set to `Yes`
 * the security group includes `inbound` rules for both `SSH` and `SMTP` (you can create seperate groups, as the Mailer instance does no require SMTP inbound connections).

### Create James Instance: 
 * Use the image for Ubuntu 16.04, and attach the newly created VPC
 * When prompted to create new SSH keys, perform this task and store the `.pem` file to local disk.

### Create Mailer instance:
 * Use the Amazon Linux image (any OS is feasible, as long as `sendmail` is available), and attach the newly created VPC

## James Server install:

### Source the software

Download the Binary ZIP file to your local disk via:
http://james.apache.org/download.cgi

Copy the file over to James server via `scp`:

For example:
```scp -i .ssh/<filename>.pem Downloads/james-server-app-3.0.1-app.zip ec2-user@18.219.209.59:/tmp```

### Setup OS (specifically for Ubuntu image):

Install JVM and libc6:

```
apt-get update
apt-get install default-jre
apt-get install libc6-i386 libc6-dev-i386
```

### Install the Apache James software

 * Unpack the ZIP file, and move to desired location:
```
apt-get install unzip
cd <scp destination folder>
unzip james-server-app-3.0.1-app.zip
mv james-server-app-3.0.1-app /usr/local/share/james-server
```

 * Configure the Apache James server using default values:
```
cd /usr/local/share/james-server/conf
rename all *template* files by removing the '-template' value.
```

 * Start James server:
```
cd /usr/local/share/james-server/bin
./james start
```

 * Test connection via telnet to confirm James Server is active:
```telnet localhost 25```

 * Review log files, as required.
 
Logs are located at: `/usr/local/share/james-server/log`. Initial log files to review are: `james-server.log` and `wrapper.log`

## Configure for Whitelist and Mail relay

### Whitelist the Mailer Instance

 * Edit the file `/usr/share/local/james-server/conf/smtpserver.xml`:
 
Change the value (default is 127.0.0.1) of `<authorizedAddresses>` to be the IP address of the Mailer instance.
  
For example:
```<authorizedAddresses>172.32.11.57</authorizedAddresses>```

### Enable Mail Relay

 * Edit the file `/usr/share/local/james-server/conf/mailetcontainer.xml`:
 
 Comment out the following lines:
 ```
 <mailet match="RemoteAddrNotInNetwork=127.0.0.1" class="ToProcessor">
          <processor>relay-denied</processor>
          <notice>550 - Requested action not taken: relaying denied</notice>
       </mailet>
```

### Restart the Apache James Server:
./james restart

## Configure Mailer instance.

### Update /etc/hosts

Add an entry into `/etc/hosts` for James Instance. 

For example:
``172.32.45.179   james-server``

### Configure Sendmail:

Edit `/etc/mail/sendmail.cf`, and add the James instance to the DS value.

For example:
```
# "Smart" relay host (may be null)
DSjames-server
```

### Restart Sendmail:

Run: `service sendmail restart`

### Install telnet and mailx:

Run: `yum install telnet mailx`

### Test connection to James instance via telnet:

For example:
```
telnet james-server 25
Trying 172.32.45.179...
Connected to james-server.
Escape character is '^]'.
220 ip-172-32-45-179 JAMES SMTP Server Server (JAMES SMTP Server ) ready
QUIT
221 2.0.0 ip-172-32-45-179 Service closing transmission channel
Connection closed by foreign host.
```

### Send email to remote email address:

For example:
```
mailx -s "Mail from James challenge" newtondkim@ 

Hello,
Test email from Robin's James Apache challenge
Best regards,
Robin
<CTRL-D>
```
