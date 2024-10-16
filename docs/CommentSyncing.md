# Comment Syncing

Any save saveToTB or loadFromTB replenishes comments and commentmeta tables in scratch. These are then compared to the actual database.

This is done in /var/www/sureveillance.sh every 10 minutes. It calls siteContactSync if a difference is noted, it will save current wp_comments and wp_commentmeta to /home/[username]/tarballs, update scratch DB, transmits it to the far end, tarball directory and installs it in scratch and actual database. If it should happen that comments need to be truly merged, various useful mysql lines can be found in the file.
