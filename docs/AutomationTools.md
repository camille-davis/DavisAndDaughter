# Automation Tools

These tools are driven by the [site overview csv file] table or "database." They will work on wordops (ee v3) or ee v4.

## Handy Tools

- `getdn 0` - display the database table.
- `doemall.sh arg` - execute arg for every site in the database. You can do `doemall.sh ./sitemaker.sh` or `doemall.sh "ee site enable"`
- `zapNginxCache.sh` - do whenever a site or its database is loaded or otherwise modified.

## Site create, load, save, and move tools

- `sitemaker.sh` will make a site and set up the database. https sites are created as http sites since the front end terminates ssl. Note, for word-ops, if the site is not already available in a tarball, it will be created with wo supplied DB & credentials. Either the database will need to be made to match, OR the database moved from the WO created one to the one in the database (mysqldump WO-DB > site.sql; mysql Table-DB < site.sql) If the WO database name is created using dashes, it need to be changed. Dashes cause issues. Be sure wp-config.php ends up in var/www/sitename.com/htdocs/wp-config.php.
- `loadFromTB.sh` - loads the site and database tarballs from /[username]/tarballs. If no site tarball is present, it will exit. Tested with ee v4 and wo.
- `saveToTB.sh` - saves tarballs to /[username]/tarballs. Tested only with wo.
- `pushTB.sh` - pushes tarballs to the other backend servers. Works with wo and ee v4.
- `pullTB.sh` - pulls tarballs from the other backend server. Works with wo and ee v4.
- `sitePublish.sh` - this is the main one. Use when a site modification looks solid. This updates the reference, produces a tarball, sends it to the other backend servers, updates them, and updates their reference sites, locks down. Use with caution.
- `siteEmergency_TB_restore.sh` - publishes entire backend from tarball. This is rescue. The idea it to put a undamaged tarball set from an archive in /home/[username]/tarballs/ and do `siteEmergencyTBrestore.sh`
- `siteEmergencyRefRestore.sh` - publishes entire backend from site reference. This is rescue. The idea is to restore from the local reference site. (Sitename.com-r)
- `siteEmergencyRefBackupRestore.sh` - Publishes entire backend from site reference backup. This is rescue. The idea is to restore from the local reference backup site. (Sitename.com-r-bu)
- `siteCommentSync.sh` - self explanatory. Run automatically.

## Lockdown and verification tools

- `siteUnlock.sh` - unlock for site editing.
- `siteUpdateR.sh` - diffs and update the reference if necessary, saving diff results.
- `siteDiffer.sh` (or `siteUpdateR.sh` ?) - diffs only, saving diff results, writing /var/www/sanity.report
- `siteMysqlcheck.sh` - compares database to tarball, writing sanity.report
- `siteLock.sh` - locks the site back up.
- `unlockAll.sh` - unlocks 'em all. Handy for mass plugin updates, etc.
- `updateAllR.sh` - updates all the references. Handy for mass plugin updates, etc.
- `lockAll.sh` - handy for mass plugin updates, etc.
- `surveillance.sh` - diffs the sites and the databases, which will fill up /var/www/sanity.report. Admins are then notified if the sanity report is changed from the previous one. Contacts are synced.

## Do all site tools

These do the indicated operation for all the sites in the DB.

- `lockAll.sh`
- `unlockAll.sh`
- `updateAllR.sh`
- `diffAll.sh`
- `mysqlCheckAll.sh`
- `pushAll.sh`
- `pullAll.sh`
- `publishAll.sh`
- `siteEmergencyTBrestoreAll.sh`
- `siteEmergencyRefRestoreAll.sh`
- `siteEmergencyRefBackupRestoreAll.sh`

sanity.report reports only discrepant sites or databases.

To execute a command on the remote computer:
```
ssh –p[secret port] root@[backend 2 hostname]   /path/command.sh
```
