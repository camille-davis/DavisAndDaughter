# SSL

## How to set up SSL in the first place

This is all done on [frontend 1 hostname] (or [frontend 2 hostname]). Traffic between [frontend 1 hostname] and back end is http://

Reference: https://serversforhackers.com/c/letsencrypt-with-haproxy

Install certbot:
```
add-apt-repository -y ppa:certbot/certbot
apt-get update
apt-get install -y certbot
```

Then modify the haproxy.conf file:

Add this in frontend:
```
# Test URI to see if its a letsencrypt request
    acl letsencrypt-acl path_beg /.well-known/acme-challenge/
    use_backend letsencrypt-backend if letsencrypt-acl
```

And add a new (phony) backend
```
# LE Backend
backend letsencrypt-backend
    server letsencrypt 127.0.0.1:8888
```

If DNS is already pointing to the fronend, we can get a cert. (Otherwise see "Getting a cert in advance of the DNS switch" section.)

Temporarily block 443 (TODO: is this necessary?), then run:
```
certbot certonly --standalone -d website.com –d www.website.com --cert-name website.com --non-interactive --agree-tos --email [my email] --http-01-port=8888
```

Cert elements get concatenated and stored in haproxy/certs:
```
cat /etc/letsencrypt/live/$1/fullchain.pem /etc/letsencrypt/live/$1/privkey.pem > /etc/haproxy/certs/$1/$1.pem
```

In HAProxy, we will force https with
```
# Redirect if HTTPS is *not* used
# redirect scheme https code 301 if !{ ssl_fc } #does all sites
redirect scheme https code 301 if { hdr(Host) -i [backend 1 url] } !{ ssl_fc }
```

See examples in haproxy.cfg

AND - SUPER IMPORTANT -
To make https work, on the backend site, add this in wp-config.php RIGHT AFTER define('WP_DEBUG') for every secure site:
```
if (strpos($_SERVER['HTTP_X_FORWARDED_PROTO'], 'https') !== false)
    $_SERVER['HTTPS']='on';
```
Then add this to backend in haproxy.cfg:
```
http-request add-header X-Forwarded-Proto https if { ssl_fc }
```

We force wp-login, wp-admin onto the [backend 1 hostname] machine with this in the frontend
```
acl is_admin path_beg -i /wp-admin
acl is_admin path_beg -i /wp-login.php
use_backend adminserver if is_admin
```

And a new backend like this:
```
backend adminserver
  mode http
  option log-health-checks
  http-request add-header X-Forwarded-Proto https if { ssl_fc }
  server s1 [internal ip redacted]:80 check
```

## New Cert Setup

On [frontend 1 hostname], try:
```
/etc/haproxy/do-the-cert.sh site_name.com.
```
"Who knows, it just might work." It typically will if the domain is already pointing to [frontend 1 hostname]. Otherwise it will explain how to get it to work.

## Automated Cert Renewal

As of 4/11/20 this is automated to renew on the first of the month. Full trail, notification, and backup implemented and tested.

## Cert Renewal

Use the script in /etc/haproxy

A nice post of redundant HA here: https://community.letsencrypt.org/t/how-to-configure-lets-encrypt-on-floating-ip/18835

## Getting a cert in advance of the DNS switch

Letsencrypt want control over DNS ascertained. This is accepted as true if DNS is already pointing to the site, but if we are moving a site, this would mean some downtime while the DNS records transfer. There is a way to get the cert in advance using a verification method called DNS01. In this method, a dns record is added temporarily. One LE reads it, it will issue a cert.

To move the site, tarball the working https site, move to desired backend, update [site overview csv file], do sitemaker.sh and then loadFromTB.sh. Get the new site working as a non http site served directly from the backend server. There is usually a caching issue as the caches try to preserve the https warnings and error stuff. Log in and check for https in WP settings.

Now, go to the frontend. We want to get a cert for the domain and then switch DNS.

Here is a generic method: https://serverfault.com/questions/750902/how-to-use-lets-encrypt-dns-challenge-validation - This one works, but requires manual stuff.

Enter this and follw the instructions. Namesilo take max 15 minutes to "deploy."
```
certbot -d [domain] -d *.[domain]  --manual --preferred-challenges dns certonly
```

If the DNS records are on DigitalOcean, here is a more automated way, untested: https://certbot-dns-digitalocean.readthedocs.io/en/stable/

An automated method for NameSilo DNS, we have this: https://github.com/ethauvin/namesilo-letsencrypt - I could not get it to work.

