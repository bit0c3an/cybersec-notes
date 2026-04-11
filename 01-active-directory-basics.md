# Active Directory Basics

> Notes from TryHackMe — Active Directory learning path  
> Module: Active Directory Basics

---

## What is Active Directory and Why Does It Exist?

Imagine managing 5 computers manually — doable. Now imagine 157 computers across 4 offices with 320 users. That's where Active Directory comes in.

A **Windows Domain** is a group of users and computers under the administration of a business. The idea is to centralize everything in one place called **Active Directory (AD)**. The server that runs AD is called a **Domain Controller (DC)**.

### Main advantages
- **Centralized identity management** — configure all users from one place
- **Security policy management** — apply policies across the entire network from AD

A real example: when you log into any computer at school/university with the same credentials, that's AD authenticating you centrally. The policies that stop you from accessing the Control Panel on campus machines? Also AD.

---

## Core AD Objects

The AD Domain Service (AD DS) acts as a catalogue of all **objects** on the network.

### Users
- Most common object type
- **Security principals** — can be authenticated and assigned privileges
- Two types:
  - **People** — employees that need network access
  - **Services** — accounts for services like IIS or MSSQL (only have privileges needed for their specific service)

### Machines
- Every computer that joins the domain gets a machine object
- Also security principals — assigned their own account
- Machine account name = computer name + `$` sign (e.g. `DC01$`)
- Passwords are auto-rotated and are 120 random characters
- Generally shouldn't be accessed by anyone except the computer itself

### Security Groups
- Used to assign access rights to resources (files, printers, shares) to entire groups instead of individual users
- Can contain users, machines, and other groups
- A user can belong to **many groups**

| Group | Description |
|---|---|
| Domain Admins | Full admin over the entire domain including all DCs |
| Server Operators | Can administer DCs but can't change admin group memberships |
| Backup Operators | Can access any file regardless of permissions — used for backups |
| Account Operators | Can create or modify other accounts |
| Domain Users | All existing user accounts |
| Domain Computers | All existing computers |
| Domain Controllers | All existing DCs |

---

## Organizational Units (OUs)

OUs are **containers** used to classify and organize users and machines. Main use: applying different policies to different departments.

- A user can only belong to **one OU at a time**
- Typical structure mirrors the business org chart (Sales OU, IT OU, Marketing OU, etc.)

### Default containers created by Windows
| Container | Contents |
|---|---|
| Builtin | Default groups available to any Windows host |
| Computers | Where machines go by default when joining the domain |
| Domain Controllers | Default OU for DCs |
| Users | Default users and groups |
| Managed Service Accounts | Accounts used by services |

### OUs vs Security Groups — Key Difference
- **OUs** → apply policies to users/computers (a user can only be in one OU)
- **Security Groups** → grant permissions over resources (a user can be in many groups)

---

## Delegation

Delegation lets you give specific users control over OUs **without making them Domain Admins**.

Most common use case: giving IT support the ability to reset passwords for users in specific OUs.

Example: delegating password reset rights over the Sales OU to a helpdesk user called Phillip.

Once delegated, Phillip can reset Sophie's password via PowerShell:
```powershell
Set-ADAccountPassword sophie -Reset -NewPassword (Read-Host -AsSecureString -Prompt 'New Password') -Verbose
```

Force password change on next logon:
```powershell
Set-ADUser -ChangePasswordAtLogon $true -Identity sophie -Verbose
```

---

## Machine Organization Best Practice

By default all machines go to the "Computers" container. Better practice is to separate them into OUs:

1. **Workstations** — daily use machines for regular users. Never have privileged users logged in.
2. **Servers** — provide services to users or other servers.
3. **Domain Controllers** — most sensitive devices. Contain hashed passwords for all domain accounts.

---

## Group Policy Objects (GPOs)

GPOs are collections of settings applied to OUs. They can target users or computers.

- GPOs apply to the linked OU **and all sub-OUs** under it
- Distributed via a network share called **SYSVOL** stored on the DC (`C:\Windows\SYSVOL\sysvol\`)
- Changes can take up to **2 hours** to propagate — force sync with:
```powershell
gpupdate /force
```

### Common GPO examples
- **Restrict Control Panel access** — applied to Sales, Marketing, Management OUs (not IT)
- **Auto lock screen after 5 min inactivity** — applied to root domain so all machines inherit it

---

## Authentication Protocols

### Kerberos (default in modern Windows)
Users receive **tickets** as proof of authentication instead of sending credentials each time.

**Flow:**
1. User sends username + encrypted timestamp to the **KDC (Key Distribution Center)** on the DC
2. KDC responds with a **TGT (Ticket Granting Ticket)** + Session Key
3. When user wants to access a service, they send the TGT to the KDC requesting a **TGS (Ticket Granting Service)**
4. KDC sends back a TGS encrypted with the Service Owner's hash
5. User presents TGS to the service — connection established

> Key point: the TGT is encrypted with the **krbtgt account's password hash** — user can't read its contents. This is why attacking krbtgt (Golden Ticket attack) is so powerful.

### NetNTLM (legacy, kept for compatibility)
Uses a **challenge-response** mechanism — password never transmitted over the network.

**Flow:**
1. Client sends auth request to server
2. Server sends a random **challenge** number
3. Client combines password hash + challenge to generate a **response**
4. Server forwards challenge + response to DC for verification
5. DC recalculates and compares — if match, authenticated
6. Result sent back to client

> Key point: if a local account is used instead of a domain account, the server can verify the response itself using the local SAM — no DC needed.

---

## Forests, Trees and Trust Relationships

### Trees
When a company expands, you can split the domain into subdomains that share the same namespace.

Example: `thm.local` splits into `uk.thm.local` and `us.thm.local` — this forms a **Tree**.

- Each subdomain has its own DC and manages its own resources
- IT teams manage their own branch independently
- **Enterprise Admins** group has admin privileges over ALL domains in the enterprise

### Forests
When multiple companies merge, their separate domain trees form a **Forest** — union of trees with different namespaces.

Example: `thm.local` tree + `mht.local` tree = one forest.

### Trust Relationships
Trusts allow users from one domain to access resources in another domain.

| Type | Description |
|---|---|
| One-way trust | If AAA trusts BBB, users from BBB can access resources in AAA (not the other way) |
| Two-way trust | Both domains mutually authorize each other's users — default when joining a tree/forest |

> Important: a trust relationship doesn't automatically grant access to everything. It just allows you to authorize users across domains — what's actually authorized is still up to you.

---

## Attacker Perspective — Why This Matters

| AD Concept | Why attackers care |
|---|---|
| Domain Admins | Primary target — full control over domain |
| Domain Controllers | Most sensitive — contain all password hashes |
| SYSVOL share | Sometimes contains credentials in GPO scripts |
| krbtgt account | Compromising it = Golden Ticket attack = unlimited persistence |
| Trust relationships | Cross-domain attacks — compromise one domain to pivot to another |
| Delegation | Misconfigured delegation = privilege escalation path |
| Machine accounts | Often overlooked — can be used for lateral movement |

---

*Notes by [@bit0c3an](https://github.com/bit0c3an) — TryHackMe Active Directory path*
