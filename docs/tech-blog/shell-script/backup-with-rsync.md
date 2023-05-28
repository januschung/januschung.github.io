
![floppy disks](../../assets/tech-blog/shell-script/backup.avif)
Here are the scripts which I have been using to generate backup for file and mysql database. They  can be run manually but preparbaly with cron jobs for automation.

Note that for the mysql backup script, there is a setting for `rotation_day`. This is used to keep the number of copy of the database backup, which will be recyled in the final commnad:

`find $backup_path/ -mtime +$rotation_days -exec rm {} \;`

The backup file is also making use of `gpg` for encryption.

## Backup file with rsync
``` bash
#!/bin/bash
source_path="[CHANGE ME]"
destination_path="[CHANGE ME]"
 # prefix for log file, eg "photobackup-"
prefix="[CHANGE ME]"
logfile=$prefix"_"`date +%d"_"%m"_"%Y"__"%H`.log

umask 177
 
 # run rsync and exclude the archive folder
/usr/bin/rsync -avz --log-file=$destination_path/$logfile --exclude archive $source_path $destination_path
```

## Backup mysql database biweekly with encryption

``` bash
#!/bin/bash
user="[CHANGE ME]"
password="[CHANGE ME]"
gpgrcp="[CHANGE ME]"
DATABASE="[CHANGE ME]"
#set daily backup dir
backup_path="[CHANGE ME]"
prefix="p[CHANGE ME]"
suffix="_"`date +%d"_"%m"_"%Y"__"%H`.sql.gz
rotation_days=14;
umask 177
 
FILENAME=$prefix$DATABASE$suffix
# dump and gzip databases
/usr/bin/mysqldump -u$user -p$password $DATABASE \
           --events --ignore-table=mysql.event | \
           /bin/gzip > $backup_path/$FILENAME
# encrypt files
/usr/bin/gpg -r $gpgrcp -e $backup_path/$FILENAME
# delete only gzipped files
/bin/rm $backup_path/$FILENAME

# recycle files
find $backup_path/ -mtime +$rotation_days -exec rm {} \;
```