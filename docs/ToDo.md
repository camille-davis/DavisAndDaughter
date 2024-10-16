# ToDo

## Speed

Cloudflare freeplan. SSL is a problem. Nginx caches. Haproxy could. It has little else to do.

So far no good:

Moving [Website 3] from CloudFlare to Quattro had no effect.

I tried numerous wonderful plugins:
- Fast velocity minity
- Hummingbird
- Autooptimize
- W3 Total Cache
- WP Performance
- WP Supercache
- Async optimizer
and four or five others.

Not a single one of them made a significant improvement.  Async optimizer helped maybe a tad.

So? Conclusion.

We have fastcgi cache enabled (it's a part of nginx) and enough memory to take good advantage of it. This seems to be doing all the heavy lifting. Despite the scores, things are brisk. Cached pages are fast, and fastcgi caches agressively.

It does make some sense. fascgi cache comes ahead of anything wordpress might do, and, fastcgi cache is, well, fast.

So other than structural changes to the pages, we seem stuck.
