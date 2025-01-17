# Red-Team Oriented Reconnaissance & Enumeration

Redteam members don't follow attack paths that involve throwing exploits around recklessly.

A Redteam member will usually identified misconfiguration or exploit relationship which will take him all the way to domain admin with minimum noise.

In  Blog we are going to cover the following.

* Hunting for users
* (Local) administrator enumeration
* GPO enumeration and abuse
* AD ACLs
* Domain Trusts and more.

**The majority of those from an unpriviledge user's point of view!!**

System Administrators do not seem to realize the amount of information we can pull from AD as a basic domain user.

The two tools we are going to use throughout this module is [Powerview](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1) and [AD Powershell Module](https://learn.microsoft.com/en-us/powershell/module/activedirectory/?view=windowsserver2025-ps).

**Note: You will notice that a large portion of the enumeration activities leverage PowerShell. It is known fact that PowerShell i being heavily monitored and logged nowdays. Another known fact is the attackers as well as redteamer are now leveraging C# and .NET to perform their operations** 

**Powershell can still used for convert opertions(AMSI Bypass, Constrained Language Mode Bypass, Applocker bypass, Logging Bypass etc..)**

The powerShell Module should be install after initial compromise from an elevated shell

```PowerShell
Import-Module ServerManager
Add-WindowsFeature RSAT-AD-PowerShell
```

### DNS using LDAP

We can do DNS lookups using LDAP. We don't have to ask DNS, which has detailed logging and information about what user are querying for are stored.

We can also do reverse lookup like "what's the website" or "What's this computer" related to this IP Address? Even if there are not any pointer records configured in DNS, the lookup will be successfull beacuse all are through AD.

To Identify machine inside the domain or do reverse lookup via LDAP. we would execute the following AD PowerShell module commands inside our domain.

```PowerShell

get-adcomputer -filter * -properties IPV4address | where {$_.IPV4Address} | select name,ipv4address
```

**Reverse Lookup**
```PowerShell
Get-AdComputer -filter {ipv4address -eq 'IP'} -Properties Lastlogondate, passwordlastset, ipv4address
```

### SPN Scanning / Service Discovery

Back in old days we had to perform port scanning to find enterprise services, nowdays we can use something called `SPN scanning`.

SPN Scanning leverage standard LDAP quieres using and looking for Service Principal Names. These are the signposts that are used to identify a service on server that support kerberos authentication. **No port Scanning involved**

A service that support kerberos authentication must register an SPN.

There are number of SPN type like MYSSQLSvc, TERMSERV, WSMan that we can search for.

SPN Format will have the SPN Type, the server name and SQL often has a port number or an instance at the end.

We can get Service related information by asking the AD DC. We will be provided with a list of all the servers, their port number, the service accounts assosicated with them and some additional information.

For a SPN directory list which includes the most common SPN's, we can refer 

https://adsecurity.org/?page_id=183

SPN sacnning is way better way of scanning for services account as opposed to searching for "service" or "scv" in name during service discovery activities.

We can also request all the user account that have Service Principal Name assosicated with them, such as service accounts.

[PowerShell-AD-Recon](https://github.com/PyroTek3/PowerShell-AD-Recon/blob/master/FindPSServiceAccounts)

```PowerShell
Find-PSServiceAccounts
```

If we would like to manually perform SPN scanning, we could use the following using the AD Powershell module

```PowerShell
Get-ADComputer -filter {ServicePrincipalName -Like "*SPN*"} -properties OperatingSystemName, OperatingSystemVersion, OperatingSystemServicePack, Passwordlastset, LastLogonDate, ServicePrincipalName, TrustedForDelegation, TrustedtoAuthForDelegation
```

For more SPN Scanning. [SPN Scanning](https://adsecurity.org/?p=230)


### Group Policies

We can also discover all group policies in an organization. By default, all authenticated user have read access over them.

By analyzing group policies we can see if there's a domain PowerShell Logging Policy, a full auditing policy and configuration like "prevent local account to logon", "add server admin to local administrator group", an EMET configuration, an Applocker configuration etc.

**Powerview**

```PowerShell
Get-NetGPO | Select displayname,name,whenchnaged
```

**Powershell Module**

```PoswerShell
Get-GPO -All -Domain "DOMAIN_NAME"
```

### User Hunting

User hunting activites can be performed with pre-elevated access and post-elevated access.

PowerView leverages a couple of native API Calls `NetWkstaUserEnum` and `NetSessionEnum`. There are 3 or 4 different ways of accessing windows API through PowerShell. Most people tend to use `Add-Type`, but there is a reason we do not want to use this.

Even through using `Add-Type` to embed inline C# so that we compile all functionality in memory is the easiest method, it is not fileless.

`Pinvoke` and the embedded C# code will actually call some compilation artifacts, whenever run from script. To minimize on-disk footprint Powerview utilizes concept like straight reflection for API Interaction through PowerShell.

A great way to understand how this approach works is studying the specifics of [PSReflect](https://github.com/mattifestation/PSReflect) by Matt. At this point, we should remind you that `NetSessionEnum` is essentially what happens under the hood when we type net session on our computers.

with native `net.exe` commands we are unable to "investigate" a remote system, but API call allow us to do this. So, as an unpriviledge user we can ask for all the session on remote system like a DC or a file server.

The result will be who is logged in and from where they are logged in. We should run this call against a high value and high tarffic server. This way we can mao the whereabouts of a large number of logged in users, without being spotted.

When we request the members of particular group, the results of these different nested groups are also grouped themselves. So, we want to unroll everything and figure out what the effective members of these types of groups are. For this we can use the `-Recurse` of PowerView, that will unroll all the nested group memberships and return an effective set of all the groups of users having access right for this particular group.

Under the hood it's essentially LDAP queries and ADSI accelerators. The LDAP queries are optimized in PowerView to suit the red team approach.

We can perform more complex quieres during user hunting. For example, request for all the members of 
"Domain Admins" and then tokenize every display name in order to requery for all users that match that pattern.

**PowerView**

```PowerShell
Get-NetGroupMember -GroupName 'Domain Admins' -FullData | %{ $a=$_.displayname.split(' 
')[0..1] -join ' '; Get-NetUser -Filter "(displayname=*$a*)" } | Select-Object -Property 
displayname,samaccountname
```

**PowerShell Module**

```PowerShell
Get-ADGroupMember -Identify "Domain Admins" | % {$a = $_.name.split('')[0..1] -Join ' '; Get-ADUser -Filter {name -eq "$a"} } | Select name, samaccountname
```

When we dump an AD schema, we try to figure out a linkable pattern for administrators who have multiple accounts. It is not uncommon that someone has an elevated account and non-elevated account.

This is why we want to try and find what are the non-elevated accounts for an Interesting user and then hunt for where they are logged in.

#### Invoke-UserHunter

`Invoke-UserHunter` is a very Interesting `PowerView` command. It queries the domain for all the computer Object and then for each computer it utilizes the native API calls we mentioned previously to enumerate logged users.

**PowerView**

```PowerShell
Invoke-UserHunter -Stealth -ShowAll
```

`Invoke-UserHunter` -Stealth, enumerates all the distributed file systems and DCs and pulls all user object, script path(s), home directories etc. It actually pulls certain type of fields that trend to map where file servers are and user AD schema.

It performs a `Get-NetSession` against those systems. The sessions of those systems can provide us with an almost complete map of the network.

By map we mean "Who is logged in the domain?", "Where are they logged in?" etc

The default Invoke-UserHunter is not safe from a red team perspective, as opposed to Invoke-UserHunter -Stealth. If we are just making LDAP queries to the DC and talking to a handful of servers that everyone talks to, this behavior is quite difficult to get picked up.
 
Also, We can get all the users of an AD forest by simply  querying a single domain controller's Global Catalog, even a child domian's one! Admin priviledge are not required for this operation.

USing PowerView to get the forest's GC

```PowerShell
Get-ForestGlobalCatalog
```

***Not Tested TEST***

### Local Administrators Enumeration

Windows OS allows* any basic domain user(authenticated) to enumerate the members of a local group on remote machine.

[get-localgroup down by default](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/security-policy-settings/network-access-restrict-clients-allowed-to-make-remote-sam-calls)

We can accomplish this in two ways, using:
* WinNT service provider, a service provider we can use with ADSI accelerators that allows enumeratation of local group and users accorss network.
* NetLocalGroupGetMembers Win 32 API call, which doesn't result in the same amount of information, but it tends to much faster, since it leverages native windows functionality.

Retrieve the members of the 'Administrators' local group on  a specific remote machine, using the WinNT service provider.

**PowerShell ADSI**

```PowerShell
([ADSI]'WinNT://computer_name/Administrators').psbase.Invoke('Members') | %{$_.GetType().InvokeMember('Name', 'GetProperty', $null, $_, $null)} 
```

Retrieve more information using `Get-NetLocalGroup` (This command was orginally created to identify RID 500 accounts that are useful against the KB2871997 patch)

**PowerView**

```PowerShell
Get-NetLocalGroup -ComputerName computer_name
```

*Using `NetLocalGroupGetMembers` API call*

```PowerShell
Get-NetLocalGroup -ComputerName computer_name -API
```

*Get the list of effective users who can access a target System*

```PowerShell
Get-NetLocalGroup -ComputerName COMPUTER_NAME -Recurse
```

#### Derivative Local Admin

It's not uncommon to come across a system of heavily delegated local administrator roles. This system increases the difficulty of tracking down user to gain access to a target system, but greatly increases the chances of gaining that access.

**Identifying Administrator Account: RODC Group**

We can also identify administrator accounts indirectly by executing the PowerView, This is a viable administrator identification method since enterprise should be configuring this so that administrator Password are not kept on RODC's.

**PowerView**

```PowerShell
Get-NetGroupGetMembers -GroupName "Denied RODC Password Replication Group" -Recurse
```

**Identifying Administrator Account: AdminCount = 1**

There is good chances that any priviledge groups and account will have the `AdminCount` property set to 1. TO identify protential privileged accounts without any group enumeration using the `AdminCount` property only.

**PowerView**

```PowerShell
Get-NetUser -AdminCount | Select Name,whenCreated,Pwdlastset,LastLogonDate
```

Note:*Gives false positive when using technique*

**Identifying Administrator Account: GPO Enumeration & Abuse**

When machine boot, they determines who can log in to them/what users have administrative rights on them through restrictive groups that are set or through group policy references.

These GPO policies are by architectural design accessible to anyone on the domain. Using GPO we can figure out who can log in to a particular machine or anywhere on the domain, by talking with the DC only.

Even if there is network segmentation, even if we cannot touch/reach specific machines and we want to know who can log in to a machines, we can query the DC and correlate some of the GPOs and computer attributes to get this piece of information.

*This way we can identify an admin without sending a single packet to the target*.

**PowerView**

```PowerShell
Find-GPOLocation -Name USERNAME -Domain DOMAIN_NAME
```

To identify all computer that the specified user has local RDP access right to in the domain.

```PowerShell
Find-GPOLocation -Name USERNAME -LocalGroup RDP
```

We can also do that in reverse:
- A given system has some GPOs applied to it.
- These GPOs have some users linked through restricted groups.

example, The "Desktop Admins" group has administrative rights on that windows machine.

To find the users/groups who can administer a given machine through GPO enumeratation

**PowerView**

```PowerShell
Find-GPOComputerAdmin -ComputerName COMPUTER_NAME
```


**Identifying Administrator Account: GPPs**

```
(\\DOMAIN\SYSVOL\<DOMAIN>\Policies)
```

We can use **PowerSploit**'s `Get-GPPPassword` to identify administrator credentials in SYSVOL. It scan the SYSVOL share on the DC and Identify XML files that have a cpassword attributes (encrypted password string). We can decrypt this string since MS published the decryption key.

MS has a patch for that, KB2962486, which should be installed on every computer used to manage Group.

**Be aware that this patch doesn't delete existing GPP XML files in SYSVOl containing passwords**


**Identifying Active directory group with Local Admin Rights**

organization usually create a group policy "saying" that their workstation admin group in AD should be member of Local administrators for all their workstation.

PowerView can Pull that information out and identify which AD Admin groups or AD groups have admin rights to which computers. To identify which AD group have admin rights to which computers.

**PowerView**

```PowerShell
Get-NetGPOGroup
Get-NetGroupMember -Name "Local Admin"
```

An Alternative path to achieve the same is by targeting a specific OU. Next, We should get a list of what Group Policies apply. Then We recieve list of all computers in that OU.

**PowerView**

```PowerView
Get-NetOU
Find-GPOComputerAdmin -OUName 'OU=X,OU=Y,DC=a,DC=b'
Get-NetComputer -ADSPath 'OU=X,OU=Y,DC=a,DC=b'
```
