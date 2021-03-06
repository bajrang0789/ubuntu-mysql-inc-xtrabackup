STEPS for creating Automated Percona Xtrabackup:

1.) CREATE a backup user 'xtrabackup' on the MySQL server which needs to be backup up.
2.) GRANT the following Access Privileges. 
``` GRANT RELOAD, PROCESS, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'xtrabackup'@'%' ```
3.) Create a backup.cnf on '/etc/mysql/backup.cnf' in the server where Percona-xtrabackup-24 is installed.
```
[client]
user=xtrabackup
password=password
```
4.) Configuring System Backup User and Assigning Permissions:
	i.) On Ubuntu Systems, a backup user and corresponding backup group is already available. Confirm this by checking the /etc/passwd and /etc/group files with the following command

	The first line from the /etc/passwd file describes the backup user, while the second line from the /etc/group file defines the backup group.

``` grep backup /etc/passwd /etc/group ```
Output: 
```
/etc/passwd:backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
/etc/group:backup:x:34:root,ubuntu
/etc/group:mysql:x:1001:backup
````
	ii.) Type the following commands to add the backup user to the mysql group and your sudo user to the backup group: (make sure you add root user into backup group, also ubuntu if needed)

```
sudo usermod -aG mysql backup
sudo usermod -aG backup ${USER}
```
	iii.) The new group isn't available in our current session automatically. To re-evaluate the groups available to our sudo user, either log out and log back in, or type:

``` exec su - ${USER} ```

	iv.) You will be prompted for your sudo user's password to continue. Confirm that your current session now has access to the backup group by checking our user's groups again:

``` id -nG ```
Output:
``` root backup ```

Our sudo user will now be able to take advantage of its membership in the backup group.

Next, we need to make the /var/lib/mysql directory and its subdirectories accessible to the mysql group by adding group execute permissions. Otherwise, the backup user will be unable to enter those directories, even though it is a member of the mysql group.
	
To give the mysql group access to the MySQL data directories, type:

```
sudo find /var/lib/mysql -type d -exec chmod 750 {} \;

```
Our backup user now has the access it needs to the MySQL directory.
Given here the data-dir is : /var/lib/mysql (you can change accordingly)


5.) Cretion of Backup Assests:

	a.) Remeber I created a backup.cnf config file in STEP-3 on the server where xtrabackup is installed.
	Now we need give the respective permissions to it.

```
sudo chown backup /etc/mysql/backup.cnf
sudo chmod 600 /etc/mysql/backup.cnf
```
	b.) Create a backup root directory (where the backup EBS volume will be mounted)
```
sudo mkdir -p /xtrabackup/data/
sudo chown backup:mysql /xtrabackup/data/
```
Volume create and mount point:
```
mkfs -t ext4 /dev/xvdb
mount -t ext4 /dev/xvdb /xtrabackup/data/
```
	c.) Creation of Encryption Key to secure backup : 
```
printf '%s' "$(openssl rand -base64 24)" | sudo tee /xtrabackup/data//encryption_key && echo
```
Output:
```
root@msr-c1:~/db_scripts/percona_xtrabackup# printf '%s' "$(openssl rand -base64 24)" | sudo tee /xtrabackup/data/encryption_key && echo
N//4YRI4WZsmxi8KnEM6LJPbg3WILjBJ
```
	d.) Restrict access to the encryption key 
	It is very important to restrict access to this file as well. Again, assign ownership to the backup user and deny access to all other users:
```
sudo chown backup:backup /backups/mysql/encryption_key
sudo chmod 600 /backups/mysql/encryption_key
```
	This key will be used during the backup process and any time you need to restore from a backup.


6.) Creating the Backup and Restore Scripts

We now have everything we need to perform secure backups of the running MySQL instance.

In order to make our backup and restore steps repeatable, we will script the entire process. We will create the following scripts:

backup-mysql.sh: This script backs up the MySQL databases, encrypting and compressing the files in the process. It creates full and incremental backups and automatically organizes content by day. By default, the script maintains 3 days worth of backups.
extract-mysql.sh: This script decompresses and decrypts the backup files to create directories with the backed up content.
prepare-mysql.sh: This script "prepares" the back up directories by processing the files and applying logs. Any incremental backups are applied to the full backup. Once the prepare script finishes, the files are ready to be moved back to the data directory.

You can view the scripts in the repository for this tutorial on GitHub at any time. If you do not want to copy and paste the contents below, you can download them directly from GitHub by typing:
```
cd /tmp
curl -LO https://raw.githubusercontent.com/bajrang0789/ubuntu-mysql-inc-xtrabackup/master/backup-mysql.sh
curl -LO https://raw.githubusercontent.com/bajrang0789/ubuntu-mysql-inc-xtrabackup/master/extract-mysql.sh
curl -LO https://raw.githubusercontent.com/bajrang0789/ubuntu-mysql-inc-xtrabackup/master/prepare-mysql.sh
```

Be sure to inspect the scripts after downloading to make sure they were retrieved successfully and that you approve of the actions they will perform. If you are satisfied, mark the scripts as executable and then move them into the /usr/local/bin directory by typing:

```
chmod +x /tmp/{backup,extract,prepare}-mysql.sh
sudo mv /tmp/{backup,extract,prepare}-mysql.sh /usr/local/bin
```

Making sure them to be executable :
```
sudo chmod +x /usr/local/bin/backup-mysql.sh
sudo chmod +x /usr/local/bin/extract-mysql.sh
sudo chmod +x /usr/local/bin/prepare-mysql.sh
```

7.) Performing a Full backup: 

Creating a screen and executing the full backup script:
```
screen -S xtrabackup
sudo -u backup backup-mysql.sh
```
This creates a full backup on a day dir, so every first invocation of the backup script will be a full backup and then on the same day will be an incremental backup.
Once the backup is completed you can check with : 
```
cd /xtrabackup/data/"$(date +%a)"
ls
```

You can check the checkpoints and the backup-progress.log to get more details.

Similar is the case for incremental backups too. 

8.) Extract the Backups: 
For extraction you will need a compression library called qpress, if that is not installed while setting up percona you can download it from here : 
# To Download Qpress on ubuntu 14.04
wget http://repo.percona.com/apt/pool/main/q/qpress/qpress_11-1.trusty_amd64.deb

for Extraction go into the backup dir /xtrabackup/data/"$(date +%a)"
We can extract the backups by passing the .xbstream backup files to the extract-mysql.sh script. Again, this must be run by the backup user:
```
sudo -u backup extract-mysql.sh *.xbstream
```
Output:
``` Extraction complete! Backup directories have been extracted to the "restore" directory. ```

If we move into the restore directory, directories corresponding with the backup files we extracted are now available:

```
cd /xtrabackup/data/"$(date +%a)"/restore
ls -F
```

9.) Prepare the Final Backup:

```
sudo -u backup prepare-mysql.sh
```

If the Data-Dir is not the default one, you can mention this on my.cnf as Option group: xtrabackup 

```
[xtrabackup]
#target_dir             = /xtrabackup/data/Mon/restore/full-12-10-2018_21-53-15
datadir                 = /data2/data
port                    = 3310
```


