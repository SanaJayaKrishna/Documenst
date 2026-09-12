# Windows → Ubuntu Server: Passwordless SSH + Simple Hostname-Based RDP

This guide configures a Windows client so that you can access an Ubuntu
server using a short hostname instead of repeatedly entering the IP
address and username.

The target setup is:

``` text
SSH:
    ssh myserver

RDP:
    mstsc /v:myserver
```

The examples below use:

-   Windows username: `sanaj`
-   Ubuntu username: `jk`
-   Ubuntu server IP: `172.16.88.46`
-   Desired hostname/alias: `myserver`

Replace these values if your environment is different.

------------------------------------------------------------------------

# Part 1 --- Passwordless SSH Login

## 1. Check whether an SSH key already exists

Open **Command Prompt** or PowerShell and run:

``` cmd
dir "%USERPROFILE%\.ssh"
```

Look for:

``` text
id_ed25519
id_ed25519.pub
```

The files mean:

  File               Purpose
  ------------------ ----------------------------------------
  `id_ed25519`       Private key --- keep this secret
  `id_ed25519.pub`   Public key --- safe to copy to servers

If these files already exist and you want to use them, do not generate
another key.

------------------------------------------------------------------------

## 2. Generate an SSH key if required

If `id_ed25519` does not exist, run:

``` cmd
ssh-keygen -t ed25519
```

When prompted for the file location, press **Enter** to accept the
default:

``` text
C:\Users\sanaj\.ssh\id_ed25519
```

When asked for a passphrase:

-   Leave it empty if you want completely passwordless use of the key.
-   Use a passphrase if you want stronger protection of the private key.
    In that case, configure `ssh-agent` to avoid entering the passphrase
    repeatedly.

The resulting files should be:

``` text
C:\Users\sanaj\.ssh\id_ed25519
C:\Users\sanaj\.ssh\id_ed25519.pub
```

------------------------------------------------------------------------

## 3. Copy the public key to the Ubuntu server

Because the current Windows shell is Command Prompt, use:

``` cmd
type "%USERPROFILE%\.ssh\id_ed25519.pub" | ssh jk@172.16.88.46 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

The first time, Ubuntu will ask for the existing password:

``` text
jk@172.16.88.46's password:
```

Enter the Ubuntu user's password.

Only the public key is copied to Ubuntu.

**Never copy `id_ed25519` (the private key) to the server.**

------------------------------------------------------------------------

## 4. Fix SSH permissions on Ubuntu

SSH into the server and run:

``` bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

These permissions are important because SSH may reject an
`authorized_keys` file that is too permissive.

------------------------------------------------------------------------

## 5. Test passwordless SSH

From Windows:

``` cmd
ssh jk@172.16.88.46
```

If configured correctly, SSH should log in without asking for the Ubuntu
account password.

------------------------------------------------------------------------

# Part 2 --- Use a Hostname Instead of `user@IP`

Windows OpenSSH supports an SSH configuration file.

The file is:

``` text
C:\Users\sanaj\.ssh\config
```

If it does not exist, create it.

------------------------------------------------------------------------

## 6. Create the SSH config

Open it with Notepad:

``` cmd
notepad "%USERPROFILE%\.ssh\config"
```

Add:

``` sshconfig
Host myserver
    HostName 172.16.88.46
    User jk
    IdentityFile ~/.ssh/id_ed25519
```

Save the file.

Now the following:

``` cmd
ssh myserver
```

is equivalent to:

``` cmd
ssh jk@172.16.88.46
```

SSH automatically knows:

-   server IP: `172.16.88.46`
-   username: `jk`
-   private key: `id_ed25519`

------------------------------------------------------------------------

## 7. Verify the SSH configuration

Run:

``` cmd
ssh myserver
```

If it works, your normal SSH workflow is now simply:

``` cmd
ssh myserver
```

------------------------------------------------------------------------

# Part 3 --- One SSH Key for Multiple Servers

The same public key can be installed on multiple Ubuntu servers.

For example:

``` text
Windows
  |
  +-- id_ed25519
  |
  +-- Server 1 → authorized_keys
  +-- Server 2 → authorized_keys
  +-- Server 3 → authorized_keys
```

Only the public key is copied to the servers.

The private key remains on Windows.

For multiple servers, add separate entries to:

``` text
C:\Users\sanaj\.ssh\config
```

Example:

``` sshconfig
Host server1
    HostName 172.16.88.46
    User jk
    IdentityFile ~/.ssh/id_ed25519

Host server2
    HostName 172.16.88.47
    User jk
    IdentityFile ~/.ssh/id_ed25519

Host gpu-server
    HostName 172.16.88.48
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
```

You can then use:

``` cmd
ssh server1
ssh server2
ssh gpu-server
```

For a personal/development environment, sharing one key across several
servers is convenient. For higher-security environments, separate keys
provide better isolation.

------------------------------------------------------------------------

# Part 4 --- Configure a Windows Hostname for SSH and RDP

SSH already has its own hostname/alias mechanism through:

``` text
C:\Users\sanaj\.ssh\config
```

For RDP, a simple Windows hostname alias can be created using the
Windows `hosts` file.

The hosts file is:

``` text
C:\Windows\System32\drivers\etc\hosts
```

------------------------------------------------------------------------

## 8. Open the Windows hosts file as Administrator

Start **Notepad as Administrator**.

Then open:

``` text
C:\Windows\System32\drivers\etc\hosts
```

If Notepad does not show the file, change the file filter from:

``` text
Text Documents (*.txt)
```

to:

``` text
All Files (*.*)
```

------------------------------------------------------------------------

## 9. Add the server hostname

Add:

``` text
172.16.88.46    myserver
```

Save the file.

This tells Windows:

``` text
myserver → 172.16.88.46
```

You can verify it with:

``` cmd
ping myserver
```

or:

``` cmd
nslookup myserver
```

Note that `ping` may fail if ICMP is blocked by the server firewall;
that does not necessarily mean the hostname mapping is wrong.

A useful check is:

``` cmd
ssh myserver
```

because SSH should resolve the same hostname.

------------------------------------------------------------------------

# Part 5 --- Configure RDP Saved Credentials

RDP does not use the SSH configuration file.

Windows RDP credentials are normally stored through **Windows Credential
Manager**.

This is separate from the `.rdp` configuration.

------------------------------------------------------------------------

## 10. Store the RDP credential

You can open:

``` text
Control Panel
    → Credential Manager
    → Windows Credentials
```

Then select:

``` text
Add a Windows credential
```

For the hostname-based connection, use:

``` text
Internet or network address:
myserver

User name:
jk

Password:
<Ubuntu/RDP password>
```

Save the credential.

------------------------------------------------------------------------

## 11. Store the credential using CMD

An alternative is:

``` cmd
cmdkey /generic:TERMSRV/myserver /user:jk /pass:YOUR_PASSWORD
```

For example:

``` cmd
cmdkey /generic:TERMSRV/myserver /user:jk /pass:MyPassword
```

Be aware that putting a password directly into a command can expose it
through command history or process inspection. Using Credential Manager
interactively is preferable when possible.

To inspect stored credentials:

``` cmd
cmdkey /list
```

You should see an entry similar to:

``` text
Target: TERMSRV/myserver
```

------------------------------------------------------------------------

# Part 6 --- RDP Configuration and the "Insecure Connection" Warning

If you see a dialog similar to:

``` text
Continue With Insecure Connection?

This Remote Desktop connection is insecure.
To secure this connection, enable RDTLS Security in your client
by saving the connection settings in your client as an RDP file
and set "use redirection server name:i:1" in it.
```

this is a separate issue from the username/password.

The RDP client is warning about the security/redirection configuration
used by the Ubuntu RDP server.

The message specifically asks for:

``` text
use redirection server name:i:1
```

------------------------------------------------------------------------

# Part 7 --- Creating an RDP File

The most reliable way to keep server-specific RDP settings is to create
a dedicated `.rdp` file.

## 12. Open Remote Desktop Connection

Press:

``` text
Win + R
```

Enter:

``` text
mstsc
```

Click:

``` text
Show Options
```

Enter:

``` text
Computer: myserver
User name: jk
```

------------------------------------------------------------------------

## 13. Save the RDP configuration

Click:

``` text
Save As...
```

Save it as:

``` text
C:\Users\sanaj\Documents\myserver.rdp
```

You can also save it on the Desktop if preferred.

------------------------------------------------------------------------

## 14. Edit the RDP file

Open:

``` text
C:\Users\sanaj\Documents\myserver.rdp
```

with Notepad.

Find:

``` text
use redirection server name:i:0
```

Change it to:

``` text
use redirection server name:i:1
```

If the setting does not exist, add:

``` text
use redirection server name:i:1
```

Save the file.

------------------------------------------------------------------------

## 15. Recommended contents of the RDP file

A minimal server-specific configuration can contain:

``` text
full address:s:myserver
username:s:jk
use redirection server name:i:1
prompt for credentials:i:0
```

Other RDP settings can remain in the file as generated by Remote Desktop
Connection.

Do not put the password in the `.rdp` file.

Windows should obtain the password from Credential Manager.

------------------------------------------------------------------------

# Part 8 --- About `Default.rdp`

Windows may use a default RDP configuration file called:

``` text
Default.rdp
```

Typically this is associated with the user's Documents folder:

``` text
C:\Users\sanaj\Documents\Default.rdp
```

However, the file is not necessarily present until Remote Desktop
Connection creates/saves it, and behavior can vary depending on how the
RDP client is being invoked.

If this command:

``` cmd
notepad "%USERPROFILE%\Documents\Default.rdp"
```

says:

``` text
Cannot find the C:\Users\sanaj\Documents\Default.rdp file.
```

do not assume that something is broken.

You can create/save an RDP file using:

``` text
mstsc
→ Show Options
→ Save As...
```

For a specific server, a dedicated file such as:

``` text
myserver.rdp
```

is preferable because it keeps server-specific settings together.

------------------------------------------------------------------------

# Part 9 --- If You Specifically Want `mstsc /v:myserver`

If your preferred workflow is exactly:

``` cmd
mstsc /v:myserver
```

rather than double-clicking an `.rdp` file, then there are two separate
requirements:

1.  Windows must resolve `myserver` to `172.16.88.46`.
2.  Windows must have the RDP credential saved for `TERMSRV/myserver`.

The hostname mapping is provided by:

``` text
C:\Windows\System32\drivers\etc\hosts
```

with:

``` text
172.16.88.46    myserver
```

The credential can be stored with:

``` cmd
cmdkey /generic:TERMSRV/myserver /user:jk /pass:YOUR_PASSWORD
```

Then:

``` cmd
mstsc /v:myserver
```

will use the hostname and saved credential.

However, the RDP setting:

``` text
use redirection server name:i:1
```

is an RDP-file setting. If the Ubuntu RDP server requires that setting
to avoid the security warning, a dedicated `.rdp` file is the cleaner
solution.

------------------------------------------------------------------------

# Part 10 --- Recommended Final Setup

For your particular requirement, the cleanest setup is:

## SSH

### File

``` text
C:\Users\sanaj\.ssh\config
```

### Configuration

``` sshconfig
Host myserver
    HostName 172.16.88.46
    User jk
    IdentityFile ~/.ssh/id_ed25519
```

### Usage

``` cmd
ssh myserver
```

No IP address, no username, and no password.

------------------------------------------------------------------------

## RDP

### Windows hosts file

``` text
C:\Windows\System32\drivers\etc\hosts
```

Add:

``` text
172.16.88.46    myserver
```

### Windows Credential Manager

Store:

``` text
Target:
TERMSRV/myserver

Username:
jk

Password:
<your password>
```

### RDP configuration

Prefer:

``` text
C:\Users\sanaj\Documents\myserver.rdp
```

with:

``` text
full address:s:myserver
username:s:jk
use redirection server name:i:1
prompt for credentials:i:0
```

### Usage

Either double-click:

``` text
myserver.rdp
```

or, if the warning is no longer applicable to the default client
configuration:

``` cmd
mstsc /v:myserver
```

------------------------------------------------------------------------

# Part 11 --- Troubleshooting

## SSH still asks for a password

Run:

``` cmd
ssh -v myserver
```

Look for lines indicating whether the private key is being offered and
accepted.

Also verify:

``` cmd
dir "%USERPROFILE%\.ssh"
```

and confirm:

``` text
id_ed25519
id_ed25519.pub
config
```

exist.

On Ubuntu verify:

``` bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

Recommended permissions:

``` text
~/.ssh                 700
~/.ssh/authorized_keys 600
```

------------------------------------------------------------------------

## SSH says the key is not being used

Check the configuration:

``` cmd
ssh -G myserver
```

This displays the effective SSH configuration.

Verify that it shows approximately:

``` text
hostname 172.16.88.46
user jk
identityfile ~/.ssh/id_ed25519
```

------------------------------------------------------------------------

## RDP asks for the password even though "Remember me" was checked

Check:

``` cmd
cmdkey /list
```

Look for:

``` text
TERMSRV/myserver
```

If you previously stored the credential against the IP address, you may
have:

``` text
TERMSRV/172.16.88.46
```

while connecting to:

``` text
myserver
```

These are different targets.

Remove stale credentials if necessary:

``` cmd
cmdkey /delete:TERMSRV/myserver
```

or:

``` cmd
cmdkey /delete:TERMSRV/172.16.88.46
```

Then add the correct credential again.

------------------------------------------------------------------------

## RDP still shows the "Insecure Connection" warning

If you are using an `.rdp` file, open it and verify:

``` text
use redirection server name:i:1
```

Make sure the RDP client is actually launching that file.

If you are using:

``` cmd
mstsc /v:myserver
```

remember that a server-specific `.rdp` file and the default `mstsc`
invocation are not necessarily equivalent.

For the most predictable behavior, use:

``` text
myserver.rdp
```

with the required RDP settings explicitly saved inside it.

------------------------------------------------------------------------

# Final Result

Once configured, your Windows machine can have:

``` text
C:\Users\sanaj\.ssh\config
C:\Users\sanaj\.ssh\id_ed25519
C:\Users\sanaj\.ssh\id_ed25519.pub

C:\Windows\System32\drivers\etc\hosts

Windows Credential Manager
    └── TERMSRV/myserver

C:\Users\sanaj\Documents\myserver.rdp
```

Your daily commands become:

``` cmd
ssh myserver
```

and:

``` cmd
mstsc /v:myserver
```

or, for the most reliable RDP configuration:

``` text
Double-click myserver.rdp
```

The SSH private key remains only on the Windows machine, while the SSH
public key is installed on the Ubuntu server.
