# FAX

Kazoo has inbound and outbound faxing to/from PDF.

Faxing has progressed from 3.16 to 3.18 and now 3.19.  Master is ahead of what you will discover here.

There are certainly more parts to faxing on Kazoo then are documented here.  As they say, the best documentation is the source code..

## Fax to Email gateway
* This should work with no problem.  Assign a callflow with a DID number to go into "receive fax".  The "receive fax" application will associate with a user record.  The PDF will be delivered to the users email address.
* There is inbound faxbox funcationality also.  
  * You set this up the same way, just ue the "faxbox" callflow.
  * The faxbox has extra functionality related to the "user app" in kazoo app store within your accounts.
  * There is some code for an online fax document viewer.  I think it's only in monster-ui and not kazoo-ui.

## Email to Fax gateway
* This is _much_ _much_ more complicated to setup.
* This example shows some of the setup required.  Depending on your needs there might be some steps left out..
  * You want to fax a pdf to +15147872030 for example.
  * You send an EMAIL to a faxbox like this
    * +15147872030@4nfk24btd.fax.stormqloud.ca
    * Include your document as an attachment
    * PDF/PS/TIFF should all be ok
    * Subject line, other text goes to /dev/null ( http://en.wikipedia.org/wiki/Null_device )
  * The faxbox has a personal individual unique dns name.
    * Each user of the fax system will need a unique DNS domain record.
    * I repeat, each faxbox requires a unique DNS MX record and name
     * This means 4nfk24btd.fax.stormqloud.ca must resolve to a MX record and is unique per faxbox.
       *  Be RFC compliant, don;'t be lazy and go for A records for email!~

  * Make sure to turn on the specific user "fax notification" checkbox if you want return error/status messages.
  * Carrier flags can select a "fax specific" call path!
    * If your regular SIP trunk provider doesn;t handle T38, you can ship it to second provider path
    
* In my own case, I already restrict spam filtering for the fax domain.  I'm skipping over it but it's needed in a serious setup.  Kazoo isn;t built for spam filtering.  It's done "upstream".   Email me for more information or borrow from the installations that cover it.

* Email always arrives on SMTP port 25.
  * On Kazoo port 25 is running Postfix.
    * Postfix uses the file 
     `/etc/postfix/transport`

* This will require multiple configuration changes.  The shortlist..
  * DNS MX records, get the email to Kazoo server
  * make sure you have gs (ghostscript) installed in /usr/bin/gs
  
* These 2 links have much more information
* https://groups.google.com/forum/#!topic/2600hz-users/IHtSwuuveTQ
* https://github.com/2600hz/kazoo/tree/master/applications/fax/doc


Kazoo inbound fax service should already be running on your Kazoo servers on TCP PORT 19025.  the difficultly is routing the right email to it.

Try this to verify the email to fax service itself is running:

```
[root@kaz ~]# telnet lh 19025
Trying 127.0.0.1...
Connected to lh.
Escape character is '^]'.
220 kaz.prodosec.com Kazoo Email to Fax Server
```


### Option 1: 
  * To get the email flowing into Kazoo you must hijack port 25 on your inbound MX mail server and point it into Kazoo instead of Postfix.
  * The downside is that postfix is no longer running at port 25 on Kazoo servers.  Many programs rely on that for outbound email delivery.  Port 25 is SMTP standard!
 
 This option is discouraged.

### Option 2:
*Use Postfix to send inbound smtp traffic to kazoo and outbound traffic outbound.

*This is Postgres answering the SMTP TCP port 25 in a default Linux setup.

  * So your email arrives at port 25 by SMTP
  * Then Postgres uses the "transport" mechanism to push it to haproxy running on 2525
  * Haproxy does it's thing mapping the email onto port 19025

* Got it? ;-)


```
[root@kaz ~]# telnet lh 25
Trying 127.0.0.1...
Connected to lh.
Escape character is '^]'.
220 kaz.prodosec.com ESMTP Postfix

```

Now you need to make sure your Postgres SMTP server is answering *EXTERNAL* traffic.

```
In /etc/postfix/main.cf
transport_maps = hash:/etc/postfix/transport
```

Edit or create /etc/postfix/transport

```
# contents of /etc/postfix/transport
#
.k6.prodosec.com smtp:[127.0.0.1]:2525
.yourdomain.com smtp:[127.0.0.1]:2525



```

Listening ports for this can be changed in haproxy.cfg


```
#haproxy.cfg
global
log 127.0.0.1 local0
log 127.0.0.1 local1 notice
maxconn 4096
user haproxy
group haproxy
stats socket /tmp/haproxy.sock mode 777

defaults
log global
maxconn 2000
retries 3
timeout connect 6000ms
timeout client 12000ms
timeout server 12000ms

listen kazoo-fax-smtp
       bind *:2525
       mode tcp
       no option http-server-close
       maxconn 50
       log global
       option tcplog
       timeout client 1m
       timeout server 1m
       timeout connect 5s
       balance roundrobin
       server k6.prodosec.com 127.0.0.1:19025 check check-send-proxy send-proxy

listen bigcouch-data 127.0.0.1:15984
balance roundrobin
    mode http
    option httplog
    server k6.prodosec.com 127.0.0.1:5984

listen bigcouch-mgr 127.0.0.1:15986
balance roundrobin
    mode http
    option httplog
    server k6.prodosec.com 127.0.0.1:5986

listen haproxy-stats 127.0.0.1:22002
mode http
stats uri /

```
SIP Trunks stormqloud.ca
