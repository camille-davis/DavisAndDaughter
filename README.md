<p align="center"><img width="250px" src="archive/charlie.png" alt=""></p>
<p align="center">Charlie Davis<br>1945 - 2026</p>

# DavisAndDaughter

This is the CDN my dad built. If you like efficiency, redundancy, fine artisanal bash scripts, and not getting hacked, you can use it too.

Servers are hosted on [DigitalOcean](https://www.digitalocean.com/). The basic system has a reserved IP, two redundant frontends running HAProxy, and two redundant backends running any number of PHP websites, with a focus on secure Wordpress websites.

## Features

- Redundancy: multiple live copies of frontend and backend, with automatic cutover if a website goes down.
- Security: multiple custom features at the proxy, backend server, and PHP app level.
- Efficiency: I do not believe in "vibe coding." Every line is at the very least reviewed by a human.
- Divest from Amazon! And you can also pick which datacenter your servers live in.
