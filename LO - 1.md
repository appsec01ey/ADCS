### Learning Objective - 1 : 

• Compromise the web application on cb-webapp1.
• Privilege Escalate using CertPotato to gain admin access on cb-webapp1.

#### Solution : 

- We have a compromised machine  -> Run Invisi shell -> Load Powershell AD-Module > Get-ADComputer -Filter * -Properties Name | ft Name,DNSHostname,IPV4Address -A
- We get a list of computers including cb-webapp1.certbulk.cb.corp > Run Nmap scan to identify port 443 is open
- A web app with no auth and file upload functionality is present on https://cb-webapp1.certbulk.cb.corp
- File Upload is vuln to RCE via Malicious File Upload (Uploading a cmd.aspx shell)
- Run : whomai we get : iis apppool\defaultapppool

- What is this iis apppool\defaultapppool
```
When IIS (Internet Information Services) runs a website, it does so inside a worker process called w3wp.exe.
Each website or group of sites runs in an Application Pool, which isolates one app from another — preventing one crashed or compromised site from affecting others.

Each Application Pool needs a Windows account to run under.
In modern IIS (since Windows Server 2008), that account is a virtual account of the form: IIS APPPOOL\<AppPoolName>

Example:
Default app pool → IIS APPPOOL\DefaultAppPool
A custom app pool called “MyApp” → IIS APPPOOL\MyApp
```

```
Virtual accounts are a special Windows security feature.
They:
Don’t exist as normal user accounts in Active Directory or Local Users.
Are automatically managed by Windows — no passwords, no need to create them manually.
Have their own unique SID (Security Identifier), which allows precise ACL (permission) configuration.
Are local to the machine — you won’t see them in AD or net user.
```

```
Why They Are Created

They exist for isolation and security.
Each app pool gets its own identity.
If one site gets compromised, the attacker’s code can only access what that app pool’s account has permissions for.
This is least privilege by design — apps run under a minimal-privilege account instead of a shared one like Network Service.
```

- Now we need to break out of this virtual account to a privileged account say administrator
- Start a smbserver on compromised machine using : smbserver.py -smb2support testshare .
- Try to connect to the share via obtained shell using: dir \\hostname\testshare$ we can see that authentication happens using cb-webapp1$ [$ means its a machine account] domain account instead of iis-apppool virtual account
- Thus we can try to use certpotato to perform a priv esc to Administrator
- 
