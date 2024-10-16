# Lockdown and security

## Firewalls

Format for a complex rule:
```
ufw allow in on eth0 to any port 80 from [example ip]
```

## Frontend

Frontend must allow 80, 443 in from everyone. However, we can block wp-login and wp-admin in the HAProxy file.

[frontend 1 hostname] allows from [frontend 2 hostname] internal:
```
ufw allow in on eth1 to any port 80 from [frontend 2 internal ip]
```
And vice versa:
```
ufw allow in on eth1 to any port 80 from [frontend 1 internal ip]
```

## Backend

Allow 80 on eth0 from known people:
```
ufw allow in on eth0 to any port 80 from [my ip]
```
Allow 80 on eth1 from either of the front end servers:
```
ufw allow in on eth1 to any port 80 from [frontend 1 internal ip]
ufw allow in on eth1 to any port 80 from [frontend 2 internal ip]
```

## Permission lockdown

TODO: review our current practice and consider separate groups for each site. See: https://www.digitalocean.com/community/tutorials/how-to-host-multiple-websites-securely-with-nginx-and-php-fpm-on-ubuntu-14-04 (Check if this is already implemented in "Site isolation" section)

Investigate referencing the database as well.

These suggestions implemented: https://linuxconfig.org/basic-php-7-and-nginx-configuration-on-ubuntu-16-04-linux

## Dealing with wp-cron.php

This kloogey thing looks like traffic and fires whenever a page loads, which goes back through the loadbalancer and it sets off overload stuff.

It can be prolonged in wp-config.php (Place it above “happy editing”):
```
define('WP_CRON_LOCK_TIMEOUT', 5400);
```
It can be disabled with:
```
define('DISABLE_WP_CRON', true);
```

If this is done, a linux cron job needs to call it periodically.

Note: With it turned off, it still runs on W2.  One a minute. Woo Commerce, we suppose.

## Site isolation

See: https://www.journaldev.com/26097/php-fpm-nginx

Create a new group and user. Create pools. Copy and modify www.conf. Use same user and group as www.conf. Change pool title at top to [Nickname], Name sock with nickname and at bottom, for open_basedir, change /var/www/ to [basedir location]

Reboot and verify if this shows up in /var/run/php/

Then in the sites-available file, comment out `include common/wpfc-php73-conf` or similar. To another name and create a file in /common the references the sock, i.e.

`isolate.sh` does it all.
