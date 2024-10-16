HOW TO BRING UP THE QUATTRO

Put two droplets up as [backend 1 url] (primary) and [backend 2 url] (backup).
They should have live in the same data (sub)center, Ubuntu 18.04, have private ip enabled, ssh key authentication (select all the keys)

These are the servers, all non-ssl. The HAPROXY will take care of ssl

RAW SERVERS – DO FIRST
On both of these,

------lockdown------
Open ports
  ufw allow 22
  ufw allow [secret port]
  ufw allow 80
  ufw enable
Once balancing is set up, we will remove general port 80 and only allow eth1 access from the internal ip of the balancers.  This is done with
  ufw  allow in on eth1 from lb-ip to any port 80

Edit /etc/ssh/sshd_config:
  Make sure #Port 22 is commented and under it add
  Port [secret port]
  Make sure the file has PasswordAuthentication no
Restart ssh with
  service ssh restart
  log out and verify that port [secret port] works and port 22 doesn’t
  remove port 22 with
  ufw delete allow 22

SYSTEM TIME
ntp is pre-installed, set timezone with
  timedatectl set-timezone America/Los_Angeles
  Check it with
    date
 command

Generate Keys
  ssh-keygen –t rsa, accept defaults but use [passphrase] for passphrase.
     Add keys to the other servers ~/.ssh/authorized_keys file
To avoid having to enter the passphrase all the time:
   apt-get install keychain
Then in .bashrc add
  keychain id_rsa
  . ~/.keychain/`uname -n`-sh

IF YOU WANT TO ADD A SWAPFILE
 This is a good idea on 1GB RAM machines. Visit
https://linuxize.com/post/how-to-add-swap-space-on-ubuntu-18-04/


NOTE EE v4 abandoned 11/16/19 Now using wordops, a fork of ee v3. Continue appropriately.

<del>
Install the v4 easyengine with
  wget -qO ee rt.cx/ee4 && sudo bash ee

Install identical wordpresses on both (or whatever) with
    Ee site create name.com –type=wp
    Log in to them and change the weird password to something sensible or better, set them up sensibly from the start with https://easyengine.io/commands/site/create/

Install database tools – the docker DB backup stuff won’t otherwise work.
  apt install mysql-client-5.7
  apt install mariadb-client-10.1
These both must be installed


Disable Docker management of iptables, so that ufw will work
IMPORTANT-undo back to add a wp site or manage plugins-IMPROTANT
  echo "{ \"iptables\": false }" > /etc/docker/daemon.json

discussed in detail here
https://www.mkubaczyk.com/2017/09/05/force-docker-not-bypass-ufw-rules-ubuntu-16-04/

But this method does not appear to work.

Another method is found here https://stackoverflow.com/questions/30383845/what-is-the-best-practice-of-docker-ufw-under-ubuntu
Scroll down to “Rollback” and do the /etc/ufw/after.rules mod. This seems to allow docker to do its thing but ufw is respected as well.
 The after.rules mod consists of adding these line at the end of the file, then restarting ufw.


# BEGIN UFW AND DOCKER
*filter
:ufw-user-forward - [0:0]
:DOCKER-USER - [0:0]
-A DOCKER-USER -j RETURN -s [internal ip redacted]/8
-A DOCKER-USER -j RETURN -s [ip 1 redacted]/12
-A DOCKER-USER -j RETURN -s [ip 2 redacted]/16

-A DOCKER-USER -j ufw-user-forward

-A DOCKER-USER -j DROP -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d [ip 2 redacted]/16
-A DOCKER-USER -j DROP -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d [internal ip redacted]/8
-A DOCKER-USER -j DROP -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d [ip 1 redacted]/12
-A DOCKER-USER -j DROP -p udp -m udp --dport 0:[port redacted] -d [ip 2 redacted]/16
-A DOCKER-USER -j DROP -p udp -m udp --dport 0:[port redacted] -d [internal ip redacted]/8
-A DOCKER-USER -j DROP -p udp -m udp --dport 0:[port redacted] -d [ip 1 redacted]/12

-A DOCKER-USER -j RETURN
COMMIT
# END UFW AND DOCKER
This may have some weird side effects and should be removed eventually

Now we can restrict port 80 access to internal ip’s with
  ufw  allow in on eth1 from lp-ip to any port 80

Find the right docker container with
  Docker container ps
Get into the appropriate one and fool around with
  docker container exec -it [container id]  /bin/bash
There is no editor. Files can be copied out, edited and back in using this sort of thing. Or install an editor in the container.
  docker  cp [container id]:/etc/nginx/conf.d/default.conf ./

WP files can be edited directly by following nano /opt/easyengine/sites/site.com/app etc

Databases can be manipulated with
# Backup
docker exec CONTAINER /usr/bin/mysqldump -u root --password=[root password] DATABASE > backup.sql

# Restore
cat backup.sql | docker exec -i CONTAINER /usr/bin/mysql -u root --password=[root password] DATABASE

Which does not appear to work, however, this actually works:
docker exec -i [container id] mysql -uroot -p[password] --database=[database] < /home/[username]/tarballs/[database].sql

That password is in [location redacted], but

docker exec -i [container id] mysql -uroot -p[password] --database=[database] -e "SHOW tables;"

A nice BU script is to be found here.
https://community.easyengine.io/t/full-sql-and-file-backup-of-all-sites-on-v4-how-im-doing-it-using-cron-via-script/11789
Get root access to the DB with this
cd /opt/easyengine/services && docker-compose exec global-db bash -c 'mysql -uroot -p[root password]'
</del>
EE v4 abandoned 11/16. Way too unstable.

WORDOPS

Use wordops,  install with wget -qO wo wops.cc && sudo bash wo

Copy /var/www/*.sh, /var/www/sh/[private dir]/*, and /var/www/[site overview csv file] from [backend 1 hostname] or [backend 2 hostname].
Create user [username], and tarballs directory off of [username] home. Tarballs all live here. Move appropriate ones in.
Be sure the correct credentials are in the [site overview csv file] spreadsheet.
Establish sites with /var/www/[private dir]/sitemaker.sh NickName (from database)
For ecommerce sites, disable fastcache via wo site update sitename.com --wp




ON TO THE LOAD BALANCERS

Reference is https://www.digitalocean.com/community/tutorials/how-to-set-up-highly-available-haproxy-servers-with-keepalived-and-floating-ips-on-ubuntu-14-04.

A newer one: https://bryceandy.com/posts/creating-highly-available-load-balancers-with-haproxy-and-keepalived-on-digitalocean  seems to be the same.
Except sanity check is pgrep haproxy instead of pidof haproxy






Droplets on same (sub)data center with internal IP as well as floating IP
Names are [frontend 1 url] and [frontend 2 url]

Perform all the modification up to the loading of easy engine.

Install haproxy:
apt-get install haproxy and edit /etc/default/haproxy file adding
# Set ENABLED to 1 if you want the init script to start haproxy. ENABLED=1 # Add extra flags here. #EXTRAOPTS="-de -m 16"
Get the “anchor IP from
curl [ip redacted]/metadata/v1/interfaces/public/0/anchor_ipv4/address && echo



Improvements are possible, but the above switches on Database dead, site disables, etc.

IMPORTANT: install nginx helper plug in in back ends, and make sure it is activated AND enables. This prevents caching at various spots. Otherwise totally goofy.

IMPORTANT: haproxy uses [backend 1 url] as a default check server. It needs to be a wp site, and on both servers, with a page created called [secure wordpress page title]. Fastcgi caching can be enabled, but if so, this must go into the locations block under index. Otherwise if the site crashes, nginx cache keeps loading valid pages and haproxy never figures it out. And vice-versa when the site recovers.  (nginx really shouldn’t cache error pages!)

# cache by default
set $skip_cache 0;

# Don't cache uris containing the following segments
if ($request_uri ~* "/[secure wordpress page title]|/wp-admin/|/xmlrpc.php|wp-.*.php|/feed/|index.php|sitemap(_index)?.xml") {
        set $skip_cache 1;
}

Here we are exempting page loads for various site urls. /[secure wordpress page title] is for haproxy httpchk test. The others were in an article here: https://guides.wp-bullet.com/improving-nginx-proxy-fastcgi-page-caching-skip-cache-reasons/ These additional changes made it magically possible to do an admin login on a https site. This was failing before, necessitating a circumvention of haproxy.

Should we automate this? Maybe only the important site have Kustom Kutover.

Kustom Kutover
Any site that is have a Kustom fast Kutover feature will need to have a Kustom backend and this Kode added.
The other dummy site, [backend 2 url] is arranged to be an example of a Kustom backend.  Note that the site must have a page whose title is [secure wordpress page title]

PROBABLY ecommerce site should have caching completely disabled.
Enable: wo site update sitename.com –wpfc
Disable wo site update sitename.com –wp



Problems with keepalived

The symptom is that when [frontend 1 hostname] is alive, it is supposed to rule, but we usually find it on [frontend 2 hostname].

When I reboot [frontend 1 hostname], [frontend 2 hostname] immediately takes over. When [frontend 1 hostname] recovers, it takes it back. This is all fine.

BUT if I force it to [frontend 2 hostname], it stays there. This is not right.
If I now reboot [frontend 1 hostname], it takes it back, which is OK.
I force it to [frontend 2 hostname], it stays, but if I restart keepalived, it takes it back.

Now if I use pidof, it does not switch if I reboot. I force [frontend 2 hostname] to take, reboot [frontend 1 hostname] and it does NOT take it back.
pidof does not work. pgrep sort of works. Why?????

Seems many bugs and the cutover scheme is clouded in mystery.

We would want a front end to do this:
Variables:
Curl W2 works.
Curl W1 works.
Is my haproxy working
Can I talk to the other guy.
Is my haproxy working.
Am I the leader.
Does the other guy agree.

So keepalived is disabled
....and presenting

FEXTEND

This is in c, written as a daemon, and is found on both front-end servers at /etc/fextend/. The executable is in that directory, identical for both servers, and pointed to in /usr/bin/.  /etc/fextend/fextend.conf differentiates the two.

The two daemons exchange information via sockets over the private (eth1)

There is start/stop code for it in /etc/init.d/, thus the systemctl (or service) process will monitor it, restart it if needed, etc.   service fextend start|stop|restart all work. Reload does not.

There are two ancillary bash scripts, am_master.sh and become_master.sh, the former a query and the latter a digital ocean master switcher.

To determine viability, [website 1].com (W1) and [website 2].com (W2) are curled with the -X option, a return code of 200 indicates success. These site are chose as they only run on [backend 1 hostname] and [backend 3 hostname].

The curl goes out on the eth0 port, which is otherwise unused. It is presumed that if the front-end can curl so can anyone else.

If  W1 can curl and W2 cannot, or vice versa, it is presumed that one of the back-ends is out ant text messages are sent. It neither can curl, then fextend will take various actions. See code for specifics.




SAFELIST

Haproxy reads [safelist path]. If on the safelist access to /website.com/wp_admin/ is allowed.

The list cleverly accepts straight IPs as well as duckdns entries.

When an IP changes, safelist updates and reloads haproxy.

Sometimes reloading haproxy biffs [website 1] for a minute or so. Not good.

Hence all duckdns URSs that might frequently change were expunged from the safelist.

For these, there is an alternative. Said expunged user may do this:

curl [secret url]

This will seemingly fail, but will put the associated IP into a haproxy “stick table” with a life of 1 hour, during which the user can access the back end admin pages. This has been tested.

Note the code in the curl is also used for cloud connections to the Raspberry Pis.





COMMUNICATIONS
mqtt from anywhere: mqtt pub  -h [cloud url] -s  -p [port] -u [username] -pw [password] -t testTopic -q 1  -m "$1 `date`"  -r

And email can be done with a curl: curl -X POST https://api.forwardemail.net/v1/emails   -u [username] -d "from=[my email]" -d "to=[example]" -d "subject=test"   -d "text=test-x"

And can text message to xxxxx@[carrier]

Anveo will send sms for all but T-Mo: curl -s "https://www.anveo.com/api/v1.asp?apikey=[api key]&action=sms&from=[number 1]&destination=[number 2]&message=[text]"


Here is a one line login to the tmated sites: ssh `ssh -p [secret port] root@[frontend 1 hostname]  grep "ssh\ session:"  [tmate file location] | awk '{print $4}'`







SSL

OPERATION
 New Cert Setup
 On [frontend 1 hostname], try /etc/haproxy/do-the-cert.sh site_name.com. “Who knows, it just might work.” It typically will if the domain is already pointing to [frontend 1 hostname]. Otherwise it will explain how to get it to work.

AUTOMATED CERT RENEWAL
As of 4/11/20 This is automated to renew on the first of the month. Full trail, notification, and backup implemented and tested.


HOW TO SET UP SSL IN THE FIRST PLACE

This is all done on [frontend 1 hostname] (or [frontend 2 hostname]). Traffic between [frontend 1 hostname] and back end is  http://

using https://serversforhackers.com/c/letsencrypt-with-haproxy

Install certbot

add-apt-repository -y ppa:certbot/certbot
apt-get update
apt-get install -y certbot

Then modify the haproxy.conf file

Add this in frontend
# Test URI to see if its a letsencrypt request
    acl letsencrypt-acl path_beg /.well-known/acme-challenge/
    use_backend letsencrypt-backend if letsencrypt-acl

And add a new (phony) backend
# LE Backend
backend letsencrypt-backend
    server letsencrypt 127.0.0.1:8888

If DNS is already pointing to the fronend, we can get a cert: (Otherwise see next section)

Temporarily block 443, then (doesn’t seem to be necessary)
Then this…..

certbot certonly --standalone -d website.com –d www.website.com --cert-name website.com --non-interactive --agree-tos --email [my email] --http-01-port=8888

Cert elements get concatenated and stored in haproxy/certs
cat /etc/letsencrypt/live/$1/fullchain.pem     /etc/letsencrypt/live/$1/privkey.pem   >   /etc/haproxy/certs/$1/$1.pem

In haproxy, we will force https with
# Redirect if HTTPS is *not* used
#     redirect scheme https code 301 if !{ ssl_fc } #does all sites
     redirect scheme https code 301 if { hdr(Host) -i [backend 1 url] } !{ ssl_fc }

See examples in haproxy.cfg

AND – SUPER IMPORTANT –
To make https work, on the backend site, add this in wp-config.php RIGHT AFTER define(‘WP_DEBUG) for every secure site

if (strpos($_SERVER['HTTP_X_FORWARDED_PROTO'], 'https') !== false)
    $_SERVER['HTTPS']='on';

(If it is put at the end things do not works right.) Then add this to backend in haproxy.cfg

     http-request add-header X-Forwarded-Proto https if { ssl_fc }

We force wp-login, wp-admin onto the [backend 1 hostname] machine with this in the frontend
     acl is_admin path_beg -i /wp-admin
     acl is_admin path_beg -i /wp-login.php
     use_backend adminserver if is_admin

And a new backend like this

backend adminserver
     mode http
     option log-health-checks
     http-request add-header X-Forwarded-Proto https if { ssl_fc }
     server s1 [internal ip redacted]:80 check

CERT RENEWAL
Use the script in /etc/haproxy



A nice post of redundant HA here https://community.letsencrypt.org/t/how-to-configure-lets-encrypt-on-floating-ip/18835



GETTING A CERT IN ADVANCE OF THE DNS SWITCH
Letsencrypt want control over DNS ascertained.  This is accepted as true if DNS is already pointing to the site, but if we are moving a site, this would mean some downtime while the DNS records transfer. There is a way to get the cert in advance using a verification method called DNS01. In this method, a dns record is added temporarily. One LE reads it, it will issue a cert.


So to move the site.
Tarball the working https site, move to desired backend, update [site overview csv file], do sitemaker.sh and then loadFromTB.sh.  Get the new site working as a non http site served directly from the backend server. This is usually  caching issue as the caches try to preserve the https warnings and error stuff. Log in and check for https in WP settings.

Now, go to the frontend. We want to get a cert for the domain and then switch DNS
Here is a generic method https://serverfault.com/questions/750902/how-to-use-lets-encrypt-dns-challenge-validation  This one works, but requires manual stuff

Enter this and follw the instructions. Namesilo take max 15 minutes to “deploy.”

certbot -d [domain] -d *.[domain]  --manual --preferred-challenges dns certonly

If the DNS records are on digitalocean, here is a more automated want, untested. https://certbot-dns-digitalocean.readthedocs.io/en/stable/

An automated method for namesilo dns, we have this https://github.com/ethauvin/namesilo-letsencrypt I could not get it to work.




SETTING UP THE REDUNDANT LB

Reference is https://www.digitalocean.com/community/tutorials/how-to-set-up-highly-available-haproxy-servers-with-keepalived-and-floating-ips-on-ubuntu-14-04, however, The keepalived part doesn’t work, do read on,

Duplicate the primary and fixup the following:
    1. apt-get update and upgrade [frontend 1 hostname].
    2. make sure the [frontend 2 hostname] subdirectories on [frontend 1 hostname]:/etc/haproxy and [frontend 1 hostname]:/etc/keepalived are up to date and have correct differences for [frontend 2 hostname]

          (These are the differences between [frontend 1 hostname] and [frontend 2 hostname] if you need to set it manually
              a. Stop haproxy and keepalived
              b. haproxy.cfg. Change   bind [internal ip a redacted]:80  bind [internal ip a redacted]:80
              c. keepalived.conf Change state BACKUP, priority 100 and swap the ip address
              d, ufw switch the eth1 ip to the [frontend 1 hostname] server
              e. hosts, add [backend 1 hostname], [backend 2 hostname], [frontend 1 hostname]
              f. Start ‘em up.  )


    3. snapshot [frontend 1 hostname] and redo [frontend 2 hostname] with resulting image
    4. After [frontend 2 hostname] reboot (automatic) quickly log on and stop the haproxy and keepalived services. The reboot usually causes probs since [frontend 2 hostname] boots up looking like [frontend 1 hostname]. MAKE SURE [frontend 1 hostname] kept the floating IP. [frontend 2 hostname] is likely to grab it and it does not seem to restore to [frontend 1 hostname] after the service stop. Rebooting [frontend 1 hostname] is the fastest way to fix this.
    5. add a ufw rule: ufw allow in on eth1 from [frontend 1 internal ip] to any port 80
    6. and delete the wrong one: ufw delete allow in on eth1 from [frontend 2 internal ip] to any port 80
    7. /etc/hosts file gets slicked. Copy server ips from [frontend 1 hostname].
    8. Copy the haproxy and keepalived configs from [frontend 2 hostname]/
    9. Restart keepalived then haproxy
    10. Test by stopping haproxy on [frontend 1 hostname]. Should cutover and certs should be ok.


Install the KEEPALIVED

apt-get install build-essential libssl-dev

Do this to get keepalive installed and running as a service

apt-get install keepalived
service keepalived start


In /etc/keepalived/keepalived.conf do this
vrrp_script chk_haproxy {
    #wont work anymore script "pidof haproxy"
    script "pgrep haproxy"
    interval 2
}

vrrp_instance VI_1 {
    interface eth1
    state MASTER
    priority 200

    virtual_router_id [id number]
    unicast_src_ip primary_private_IP
    unicast_peer {
        secondary_private_IP
    }

    authentication {
        auth_type PASS
        auth_pass [password]
    }

    track_script {
        chk_haproxy
    }

    notify_master /etc/keepalived/master.sh
}
…and for secondary, this..
vrrp_script chk_haproxy {
    #wont work anymore script "pidof haproxy"
    script "pgrep haproxy"
    interval 2
}

vrrp_instance VI_1 {
    interface eth1
    state BACKUP
    priority 100

    virtual_router_id [id number]
    unicast_src_ip secondary_private_IP
    unicast_peer {
        primary_private_IP
    }

    authentication {
        auth_type PASS
        auth_pass [password]
    }

    track_script {
        chk_haproxy
    }

    notify_master /etc/keepalived/master.sh
}

We will do this to get the address change script
    cd /usr/local/bin
    sudo curl -LO http://do.co/assign-ip

Upstart no longer works. Do two things to make address change script work:

Make DO export token an env variable
export DO_TOKEN=[do token]

Do apt-get install python-requests.

Then this the  will make that LB the active master

/etc/keepalived/master.sh
.
It should be executable and look like this

export DO_TOKEN='[do token]'
IP='[floating ip]'
ID=$(curl -s http://[ip redacted]/metadata/v1/id)
HAS_FLOATING_IP=$(curl -s http://[ip redacted]/metadata/v1/floating_ip/ipv4/active)

if [ $HAS_FLOATING_IP = "false" ]; then
    n=0
    while [ $n -lt 10 ]
    do
        python /usr/local/bin/assign-ip $IP $ID && break
        n=$((n+1))
        sleep 3
    done
fi






AUTOMATION TOOLS

These tools are driven by the [site overview csv file] table or “database.” They will work on wordops (eev3) or ee v4.


HANDY TOOLS
On [backend 1 hostname] or [backend 2 hostname], getdn 0 will display the database table

/var/www/doemall.sh arg  This will execute arg for every site in the database.
  You can do doemall.sh  ./sitemaker.sh  or doemall.sh “ee site enable”

/var/www/zapNginxCache.sh do whenever a site or its database is loaded or otherwise modified.

SITE CREATE, LOAD, SAVE, AND MOVE TOOLS
/var/www/sitemaker.sh
/var/www/sitemaker.sh will make a site and set up the database. https sites are created as http sites since the front end terminates ssl. Note, for word-ops, if the site is not already available in a tarball, it will be created with wo supplied DB & credentials. Either the database will need to be made to match, OR the database moved from the WO created one to the one in the database (mysqldump WO-DB > site.sql; mysql Table-DB < site.sql) If the WO database name is created using dashes, it need to be changed. Dashes cause issues. Be sure wp-config.php ends up in var/www/sitename.com/htdocs/wp-config.php.

/var/www/loadFromTB.sh
/var/www/loadFromTB.sh will load the site and database tarballs from /[username]/tarballs. If no site tarball is present, it will exit. Tested with ee v4 and wo.

/var/www/saveToTB.sh This will save tarballs to /[username]/tarballs. Tested only with wo.

/var/www/pushTB.sh This pushes tarballs to the other backend servers. Works with wo and ee v4.

/var/www/pullTB.sh This pulls tarballs from the other backend server. Works with wo and ee v4.

/var/www/sitePublish.sh  THIS IS THE MAIN ONE
/var/www/sitePublish.sh Use when a site modification looks solid. This updates the reference, produces a tarball, sends it to the other backend servers, updates them, and updates their reference sites, locks down. Use with caution.

/var/www/siteEmergencyTBrestore.sh
/var/www/siteEmergency_TB_restore.sh. Publishes entire backend from tarball. This is rescue. The idea it to put a undamaged tarball set from an archive in /home/[username]/tarballs/ and do  siteEmergencyTBrestore,sh.

/var/www/siteEmergencyRefRestore.sh
/var/www/siteEmergencyRefRestore.sh. Publishes entire backend from site reference. This is rescue. The idea is to restore from the local reference site. (Sitename.com-r)

/var/www/siteEmergencyRefBackupRestore.sh
/var/www/siteEmergencyRefBackupRestore.sh. Publishes entire backend from site reference backup. This is rescue. The idea is to restore from the local reference backup site. (Sitename.com-r-bu)

/var/www/siteCommentSync.sh Self explanatory. Run automatically


LOCKDOWN AND VERIFICATION TOOLS
/var/www/siteUnlock.sh
/var/www/siteUnlock.sh Unlock for site editing.

/var/www/siteUpdateR.sh
/var/www/siteUpdateR.sh Diffs and update the reference if necessary, saving diff results.

/var/www/siteDiffer.sh
/var/www/siteUpdateR.sh Diffs only, saving diff results, writing /var/www/sanity.report

/var/www/siteMysqlcheck.sh
/var/www/siteSqlCheck compares database to tarball, writing sanity.report

/var/www/siteLock.sh
/var/www/siteLock.sh Locks the site back up.

/var/www/unlockAll.sh
/var/www/unlock.sh  Unlocks ‘em all. Handy for mass plugin updates, etc.

/var/www/updateRefAll.sh
/var/www/updateR.sh  Updates all the references. ‘em all. Handy for mass plugin updates, etc.

/var/www/lockdown.sh Handy for mass plugin updates, etc.
/var/www/lockdown.sh

/var/www/surveillance.sh
/var/www/surveillance.sh diffs the sites and the databases, which will fill up /var/www/sanity.report. Admins are then notified if the sanity report is changed from the previous one.  Contacts are synced..


DO ALL SITE TOOLS
These do the indicated operation for all the sites in the DB
/var/www/lockAll.sh
/var/www/unlockAll.sh
/var/www/updateAllR.sh
/var/www/diffAll.sh
/var/www/mysqlCheckAll.sh
/var/www/pushAll.sh
/var/www/pullAll.sh
/var/www/publishAll.sh
/var/www/siteEmergencyTBrestoreAll.sh
/var/www/siteEmergencyRefRestoreAll.sh
/var/www/siteEmergencyRefBackupRestoreAll.sh

/var/www/sanity.report reports only discrepant sites or databases.

To execute a command on the remote computer

ssh –p[secret port] root@[backend 2 hostname]   /path/command.sh



LOCKDOWN AND SECURITY

Currently we have these ip addresses

[backend 1 ip] [backend 1 hostname]
[backend 2 ip]   [backend 2 hostname]
[backend 2 ip]   [alias name] (another name for [backend 2 hostname])

[floating ip]  [frontend 1 hostname]f     #floating IP
[frontend 1 ip]  [frontend 1 hostname]
[frontend 2 ip]   [frontend 2 hostname]

Firewalls.

Format for a complex rule
ufw allow in on eth0 to any port 80 from [example ip]

Frontend
Frontend must allow 80, 443 in from everyone. However, we can block wp-login and wp-admin in the haproxy file
[frontend 1 hostname] allows from [frontend 2 hostname] internal and vice versa
ufw allow in on eth1 to any port 80 from [frontend 2 internal ip]
and vice versa
ufw allow in on eth1 to any port 80 from [frontend 1 internal ip]

Backend
Allow 80 on eth0 from known people.
ufw allow in on eth0 to any port 80 from [my ip]

Allow 80 on eth1 from either of the front end servers
ufw allow in on eth1 to any port 80 from [frontend 1 internal ip]
ufw allow in on eth1 to any port 80 from [frontend 2 internal ip]

Permission lockdown,

TBD review our current practice and consider separate groups for each site. See
https://www.digitalocean.com/community/tutorials/how-to-host-multiple-websites-securely-with-nginx-and-php-fpm-on-ubuntu-14-04
Investigate referencing the database as well.

These suggestions implemented
https://linuxconfig.org/basic-php-7-and-nginx-configuration-on-ubuntu-16-04-linux


Fail2ban on back end only saw stream from the LB and banned that for 10 minutes.

We could run fail2ban on the fe, mentioned here:
https://serverfault.com/questions/853806/blocking-ips-in-haproxy


GETTING FAIL2BAN TO WORK ON BACKEND

WE ARE NOT RUNNING FAIL2BAN as of 5/21/20
Found here http://blog.sbelyea.net/articles/nginx-and-fail2ban-behind-a-proxy

This is needed independent of fail2ban
1.“disable”  /etc/nginx/conf.d/cloudflare.conf by renaminy it and commenting out the cron
2. Add to /etc/nginx.conf  in http section arount the#proxy head area
  set_real_ip_from     [frontend 2 internal ip]; # enter your proxy's IP here.
    set_real_ip_from     [frontend 1 internal ip]; # enter your proxy's IP here.
    real_ip_header               X-Forwarded-For;

This stuff the true header into the access log for all the sites.


DEALING WITH wp-cron.php

This kloogey thing looks like traffic and fires whenever a page loads, which goes back through the loadbalancer and it sets off overload stuff.

If can be prolonged with
define('WP_CRON_LOCK_TIMEOUT', 5400); (Place it above “happy editing”)
in wp-config.php
It can be disabled with
define('DISABLE_WP_CRON', true);,
and if this is done, a linux cron job needs to call it periodically.
Note: With it turned off, it still runs on W2.  One a minute. Woo Commerce, we suppose.


SITE ISOLATION
See: https://www.journaldev.com/26097/php-fpm-nginx Create a new group and user with
Create pools. Copy and modify www.conf. Use same user and group as www.conf. Change pool title at top to [Nickname], Name sock with nickname and at bottom, for open_basedir, change /var/www/ to [basedir location]

reboot and verify if this shows up in /var/run/php/
then in the sites-available file,  comment out include common/wpfc-php73-conf or similar. To another name and create a file in /common the references the sock, i.e.

isolate.sh does it all



SPEED

Cloudflare freeplan. SSL is a problem. Nginx caches. Haproxy could. It has little else to do.

So far no good:

Moving [Website 3] from CloudFlare to Quattro had no effect.
I tried numerous wonderful plugins:
    • Fast velocity minity
    • Hummingbird
    • Autooptimize
    • W3 Total Cache
    • WP Performance
    • WP Supercache
    • Async optimizer
and four or five others.
Not a single one of them made a significant improvement.  Async optimizer helped maybe a tad.
So? Conclusion.
We have fastcgi cache enabled (it's a part of nginx) and enough memory to take good advantage of it. This seems to be doing all the heavy lifting. Despite the scores, things are brisk. Cached pages are fast, and fastcgi caches agressively.
It does make some sense. fascgi cache comes ahead of anything wordpress might do, and, fastcgi cache is, well, fast.
So other than structural changes to the pages, we seem stuck.
Site	Page Speed	Y-slow	Load time




COMMENT SYNCING

Any save saveToTB or loadFromTB replentishes comments and commentmeta tables in scratch. These are then compared to the actual database.

This is done in /var/www/sureveillance.sh every 10 minutes. It calls siteContactSync if a difference is noted, it will
Save current wp_comments and wp_commentmeta to /home/[username]/tarballs, update scratch DB, transmits it to the far end, tarball directoryand and installs it in scratch and actual database.  If it should happen that comments need to be truly merged,  various useful mysql line can be found in the file.



Task	Comments	Done
[site overview csv file]	Has site, type, nickname, dbname, dbuser, db password	√
getdn	getdb will fetch entries in the table. Type getdn ? for info	√
sitemaker.sh	In vw. Sitemaker sitename.com will make appropriate site iaw [site overview csv file] entries	√
doemall.sh	Doemall.sh command will do ‘command site.com’ for all of them	√
Site lock, unlock, update	These all must be modified to work with new structure; should be driven from table	√
Pull/push from other server	 NO. 1. Save TB, 2. Move TB,3. Load TB.
Save     Move   Load √	√
Pull/push from local reference	Push to local reference should update TB, no pull from reference.	X
Pull/Push from tarballs	tar -xf /home/[username]/tarballs/I.tgz  --strip-components=4 -C /opt/easyengine/sites/[domain]/app/htdocs/	√
Daily site tarballs, server BU.	Should run from same sheet. Sheet stored centrally ([backend 1 hostname]) and fetched every time. Server BU is now 7d+12m, make 7d+9w+10m
Comment syncer	Mark commentable sites in DB	√
Local reference for db also		√
DavisNet update	Add commentable, controlled admin,	√
Stress testing	Reboot fails if three or more sites enabled. This patched with “graceful” sh files	X
Stress testing	Ee v4 failed. Not doen for wo, as we have a lot of experience on ee v3	X
	Somebody has written some e tools, for v4 and wo as well.

	FUTURE STUFF
Is cloudflare useful for us?
Should be add varnish as additional be servers?	nginx fastcgi seem to be quite fast.	√
	It might be better to put the cache preventer in /var/www/conf/nginx, because wo may overwrute















