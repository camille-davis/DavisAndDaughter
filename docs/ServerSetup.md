# Server setup (doc in progress)

We are using [DigitalOcean](https://www.digitalocean.com/) droplets.

## Backends

Put two droplets up as [backend 1 url] and [backend 2 url].

They should have live in the same data (sub)center, Ubuntu 18.04, have private ip enabled, ssh key authentication (select all the keys).

Servers are all non-ssl. The HAProxy will take care of SSL.

### Lockdown

Open ports:
```
ufw allow 22
ufw allow [secret port]
ufw allow 80
ufw enable
```

Once balancing is set up, we will remove general port 80 and only allow eth1 access from the internal ip of the balancers. This is done with:
```
ufw  allow in on eth1 from lb-ip to any port 80
```

Edit /etc/ssh/sshd_config:
- Make sure #Port 22 is commented and under it add: `Port [secret port]`
- Make sure the file has `PasswordAuthentication no`

Restart ssh:
```
service ssh restart
```

Log out and verify that port [secret port] works and port 22 doesn’t. Then remove port 22:
```
ufw delete allow 22
```

### System time

ntp is pre-installed, set timezone with:
```
timedatectl set-timezone America/Los_Angeles
```
Check it with `date` command.

### Generate keys
```
ssh-keygen –t rsa
```
Accept defaults but use [passphrase] for passphrase.

Add keys to the other servers ~/.ssh/authorized_keys file

To avoid having to enter the passphrase all the time:
```
apt-get install keychain
```
Then in .bashrc add
```
keychain id_rsa
. ~/.keychain/`uname -n`-sh
```

### If you want to add a swapfile

This is a good idea on 1GB RAM machines. Visit https://linuxize.com/post/how-to-add-swap-space-on-ubuntu-18-04/

### WordOps (for Wordpress websites)

Install WordOps:
```
wget -qO wo wops.cc && sudo bash wo
```

Copy /var/www/*.sh, /var/www/sh/[private dir]/*, and /var/www/[site overview csv file] from [backend 1 hostname] or [backend 2 hostname].

Create user [username], and tarballs directory off of [username] home. Tarballs all live here. Move appropriate ones in.

Be sure the correct credentials are in the [site overview csv file] spreadsheet.

Establish sites with:
```
/var/www/[private dir]/sitemaker.sh NickName (from database)
```

For ecommerce sites, disable fastcache via:
```
wo site update sitename.com --wp
```

IMPORTANT: install Nginx Helper Plugin in backends, and make sure it is activated AND enabled. This prevents caching at various spots. Otherwise totally goofy.

## Frontends (load balancers)

References:
- https://www.digitalocean.com/community/tutorials/how-to-set-up-highly-available-haproxy-servers-with-keepalived-and-floating-ips-on-ubuntu-14-04
- A newer one: https://bryceandy.com/posts/creating-highly-available-load-balancers-with-haproxy-and-keepalived-on-digitalocean - seems to be the same except sanity check is pgrep haproxy instead of pidof haproxy

Get 2 droplets in the same (sub)data center with internal IP as well as floating IP.

Names are [frontend 1 url] and [frontend 2 url].

Perform all the modifications up to the loading of easy engine.

Install HAProxy:
```
apt-get install haproxy
```

Edit /etc/default/haproxy file, adding:
```
Set ENABLED to 1 if you want the init script to start HAProxy.
ENABLED=1
# Add extra flags here.
#EXTRAOPTS="-de -m 16"
```
Get the anchor IP:
```
curl [ip redacted]/metadata/v1/interfaces/public/0/anchor_ipv4/address && echo
```

Improvements are possible, but the above switches on Database dead, site disables, etc.

IMPORTANT: HAProxy uses [backend 1 url] as a default check server. It needs to be a wp site, and on both servers, with a page created called [secure wordpress page title]. FastCGI caching can be enabled, but if so, this must go into the locations block under index. Otherwise if the site crashes, Nginx cache keeps loading valid pages and HAProxy never figures it out. And vice-versa when the site recovers. (Nginx really shouldn’t cache error pages!)

```
# cache by default
set $skip_cache 0;

# Don't cache uris containing the following segments
if ($request_uri ~* "/[secure wordpress page title]|/wp-admin/|/xmlrpc.php|wp-.*.php|/feed/|index.php|sitemap(_index)?.xml") {
        set $skip_cache 1;
}
```

Here we are exempting page loads for various site urls. /[secure wordpress page title] is for HAProxy httpchk test. The others were in an article here: https://guides.wp-bullet.com/improving-nginx-proxy-fastcgi-page-caching-skip-cache-reasons/

These additional changes made it magically possible to do an admin login on a https site. This was failing before, necessitating a circumvention of HAProxy.

Should we automate this? Maybe only the important site have Kustom Kutover.

Any site that is have a Kustom fast Kutover feature will need to have a Kustom backend and this Kode added.

The other dummy site, [backend 2 url] is arranged to be an example of a Kustom backend. Note that the site must have a page whose title is [secure wordpress page title]

PROBABLY ecommerce site should have caching completely disabled.

Enable:
```
wo site update sitename.com –wpfc
```
Disable:
```
wo site update sitename.com –wp
```

### Problems with Keepalived

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
- Curl W2 works.
- Curl W1 works.
- Is my haproxy working
- Can I talk to the other guy.
- Is my haproxy working.
- Am I the leader.
- Does the other guy agree.

So keepalived is disabled.
....and presenting:

### Fextend

This is in C, written as a daemon, and is found on both front-end servers at /etc/fextend/. The executable is in that directory, identical for both servers, and pointed to in /usr/bin/. /etc/fextend/fextend.conf differentiates the two.

The two daemons exchange information via sockets over the private eth1.

There is start/stop code for it in /etc/init.d/, thus the systemctl (or service) process will monitor it, restart it if needed, etc. service fextend start|stop|restart all work. Reload does not.

There are two ancillary bash scripts, am_master.sh and become_master.sh, the former a query and the latter a digital ocean master switcher.

To determine viability, [website 1] and [website 2] are curled with the -X option, a return code of 200 indicates success. These sites are chosen as they only run on [backend 1 hostname] and [backend 3 hostname].

The curl goes out on the eth0 port, which is otherwise unused. It is presumed that if the front-end can curl so can anyone else.

If W1 can curl and W2 cannot, or vice versa, it is presumed that one of the back-ends is out ant text messages are sent. It neither can curl, then fextend will take various actions. See code for specifics.

### Safelist

Haproxy reads [safelist path]. If on the safelist access to /website.com/wp_admin/ is allowed.

The list cleverly accepts straight IPs as well as duckdns entries.

When an IP changes, safelist updates and reloads haproxy.

Sometimes reloading HAProxy biffs [website 1] for a minute or so. Not good.

Hence all duckdns URLs that might frequently change were expunged from the safelist.

For these, there is an alternative. Said expunged user may do this:

curl [secret url]

This will seemingly fail, but will put the associated IP into a HAProxy "stick table" with a life of 1 hour, during which the user can access the back end admin pages. This has been tested.

### Setting up the redundant load balancer

Reference: https://www.digitalocean.com/community/tutorials/how-to-set-up-highly-available-haproxy-servers-with-keepalived-and-floating-ips-on-ubuntu-14-04 - however, the keepalived part doesn’t work, so read on:

Duplicate the primary and fixup the following:
1. apt-get update and upgrade [frontend 1 hostname].
2. Make sure the [frontend 2 hostname] subdirectories on [frontend 1 hostname]:/etc/haproxy and [frontend 1 hostname]:/etc/keepalived are up to date and have correct differences for [frontend 2 hostname]

(These are the differences between [frontend 1 hostname] and [frontend 2 hostname] if you need to set it manually
- Stop haproxy and keepalived
- haproxy.cfg - change `bind [internal ip a redacted]:80` to `bind [internal ip b redacted]:80`
- keepalived.conf - change state BACKUP, priority 100 and swap the IP address
- ufw - switch the eth1 ip to the [frontend 1 hostname] server
- hosts - add [backend 1 hostname], [backend 2 hostname], [frontend 1 hostname]

3. Snapshot [frontend 1 hostname] and redo [frontend 2 hostname] with resulting image.
4. After [frontend 2 hostname] reboot (automatic) quickly log on and stop the haproxy and keepalived services. The reboot usually causes probs since [frontend 2 hostname] boots up looking like [frontend 1 hostname]. MAKE SURE [frontend 1 hostname] kept the floating IP. [frontend 2 hostname] is likely to grab it and it does not seem to restore to [frontend 1 hostname] after the service stop. Rebooting [frontend 1 hostname] is the fastest way to fix this.
5. Add a ufw rule: `ufw allow in on eth1 from [frontend 1 internal ip] to any port 80`
6. And delete the wrong one: `ufw delete allow in on eth1 from [frontend 2 internal ip] to any port 80`
7. /etc/hosts file gets slicked. Copy server ips from [frontend 1 hostname]
8. Copy the haproxy and keepalived configs from [frontend 2 hostname]
9. Restart keepalived then haproxy
10. Test by stopping haproxy on [frontend 1 hostname]. Should cutover and certs should be ok.

#### Install the Keepalived

```
apt-get install build-essential libssl-dev
```

Do this to get keepalive installed and running as a service:
```
apt-get install keepalived
service keepalived start
```

In /etc/keepalived/keepalived.conf do this:
```
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
```

And for secondary, this..
```
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
```

We will do this to get the address change script:
```
cd /usr/local/bin
sudo curl -LO http://do.co/assign-ip
```

Upstart no longer works. Do two things to make address change script work:

Make DO export token an env variable:
```
export DO_TOKEN=[do token]
```
```
apt-get install python-requests
```

Then this will make that LB the active master:
```
/etc/keepalived/master.sh
```

It should be executable and look like this:
```
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
```

