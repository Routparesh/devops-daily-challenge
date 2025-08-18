Task A: Establish Key-Based Login

1. Generate key (if needed):
   bash

```bash
ssh-keygen -t ed25519 -C "you@example.com"
```

2.Install your public key:

```bash
ssh-copy-id user@server
```

3.Verify:

```bash
ssh -vvv user@server "echo OK && id -u && hostname"
```

Ans:

debug2: exec request accepted on channel 0
OK
1000
ip-10-0-1-166

Task B: Harden sshd

Open firewall for new port (2222 shown below).

Edit /etc/ssh/sshd_config to include:

```bash
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers user
MaxAuthTries 3
PermitEmptyPasswords no
ClientAliveInterval 300
ClientAliveCountMax 2
LogLevel VERBOSE
```

Validate and reload:

```bash
sudo systemctl daemon-reload
```

```bash
systemctl restart ssh.socket
```

```bash
sudo sshd -t && sudo systemctl reload ssh
```

Test new port:

```bash
ssh -p 2222 user@server "echo PORT_OK"
```

Ans
ubuntu@ip-10-0-1-166:~/.ssh$ ssh -p 2222 ubuntu@54.90.242.251 "echo PORT_OK"
PORT_OK

Task C: Secure File Transfer Roundtrip

Create a file and upload/download:

```bash
echo "hello-ssh" > hello.txt
```

```bash
scp -P 2222 hello.txt user@server:/tmp/hello.txt
scp -P 2222 user@server:/tmp/hello.txt hello.remote.txt
```

Verify integrity:

```bash
sha256sum hello.txt hello.remote.txt | tee checksums.txt
```

Ans:
ubuntu@ip-10-0-1-166:~/.ssh$ sha256sum hello.txt hello.remote.txt | tee checksums.txt
d7c3ea5c3f06e968ed8e99480092c41967be0c71d971b825d90d95fa152c7a9c hello.txt
d7c3ea5c3f06e968ed8e99480092c41967be0c71d971b825d90d95fa152c7a9c hello.remote.txt

Task D (Optional): SFTP-Only Restricted User

Create group and user (server):

```bash
sudo groupadd -f sftpusers
sudo useradd -m -G sftpusers -s /usr/sbin/nologin sftpuser
```

Prepare chroot:

```bash
sudo mkdir -p /sftp/sftpuser/upload
sudo chown root:root /sftp /sftp/sftpuser
sudo chmod 755 /sftp /sftp/sftpuser
sudo chown sftpuser:sftpusers /sftp/sftpuser/upload
```

Add to /etc/ssh/sshd_config:

```bash
Subsystem sftp internal-sftp
Match Group sftpusers
    ChrootDirectory /sftp/%u
    ForceCommand internal-sftp
    AllowTCPForwarding no
    X11Forwarding no
```

Validate + reload, then test:

```bash
sudo sshd -t && sudo systemctl reload sshd
ssh -p 2222 sftpuser@server   # should fail (no shell)
sftp -P 2222 sftpuser@server  # should succeed

```

Ans:

ubuntu@ip-10-0-1-166:~/.ssh$ sftp -P 2222 sftpuser@54.90.242.251
Connected to 54.90.242.251.
sftp>

ubuntu@ip-10-0-1-166:~/.ssh$ ssh -p 2222 sftpuser@54.90.242.251
This service allows sftp connections only.
Connection to 54.90.242.251 closed.

cat /etc/ssh/sshd_config

# This is the sshd server system-wide configuration file. See

# sshd_config(5) for more information.

# This sshd was compiled with PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games

# The strategy used for options in the default sshd_config shipped with

# OpenSSH is to specify options with their default value where

# possible, but leave them commented. Uncommented options override the

# default value.

Include /etc/ssh/sshd_config.d/\*.conf

# When systemd socket activation is used (the default), the socket

# configuration must be re-generated after changing Port, AddressFamily, or

# ListenAddress.

#

# For changes to take effect, run:

#

# systemctl daemon-reload

# systemctl restart ssh.socket

#

#Port 22
Port 2222
#AddressFamily any
#ListenAddress 0.0.0.0
#ListenAddress ::

#HostKey /etc/ssh/ssh_host_rsa_key
#HostKey /etc/ssh/ssh_host_ecdsa_key
#HostKey /etc/ssh/ssh_host_ed25519_key

# Ciphers and keying

#RekeyLimit default none

# Logging

#SyslogFacility AUTH
LogLevel VERBOSE

# Authentication:

#LoginGraceTime 2m
PermitRootLogin no
#StrictModes yes
MaxAuthTries 3
#MaxSessions 10

PubkeyAuthentication yes

# Expect .ssh/authorized_keys2 to be disregarded by default in future.

#AuthorizedKeysFile .ssh/authorized_keys .ssh/authorized_keys2

#AuthorizedPrincipalsFile none

#AuthorizedKeysCommand none
#AuthorizedKeysCommandUser nobody

# For this to work you will also need host keys in /etc/ssh/ssh_known_hosts

#HostbasedAuthentication no

# Change to yes if you don't trust ~/.ssh/known_hosts for

# HostbasedAuthentication

#IgnoreUserKnownHosts no

# Don't read the user's ~/.rhosts and ~/.shosts files

#IgnoreRhosts yes

# To disable tunneled clear text passwords, change to no here!

PasswordAuthentication no
PermitEmptyPasswords no
AllowUsers ubuntu sftpuser

# Change to yes to enable challenge-response passwords (beware issues with

# some PAM modules and threads)

KbdInteractiveAuthentication no

# Kerberos options

#KerberosAuthentication no
#KerberosOrLocalPasswd yes
#KerberosTicketCleanup yes
#KerberosGetAFSToken no

# GSSAPI options

#GSSAPIAuthentication no
#GSSAPICleanupCredentials yes
#GSSAPIStrictAcceptorCheck yes
#GSSAPIKeyExchange no

# Set this to 'yes' to enable PAM authentication, account processing,

# and session processing. If this is enabled, PAM authentication will

# be allowed through the KbdInteractiveAuthentication and

# PasswordAuthentication. Depending on your PAM configuration,

# PAM authentication via KbdInteractiveAuthentication may bypass

# the setting of "PermitRootLogin prohibit-password".

# If you just want the PAM account and session checks to run without

# PAM authentication, then enable this but set PasswordAuthentication

# and KbdInteractiveAuthentication to 'no'.

UsePAM yes

#AllowAgentForwarding yes
#AllowTcpForwarding yes
#GatewayPorts no
X11Forwarding yes
#X11DisplayOffset 10
#X11UseLocalhost yes
#PermitTTY yes
PrintMotd no
#PrintLastLog yes
#TCPKeepAlive yes
#PermitUserEnvironment no
#Compression delayed
ClientAliveInterval 300
ClientAliveCountMax 2
#UseDNS no
#PidFile /run/sshd.pid
#MaxStartups 10:30:100
#PermitTunnel no
#ChrootDirectory none
#VersionAddendum none

# no default banner path

#Banner none

# Allow client to pass locale environment variables

AcceptEnv LANG LC\_\*

# override default of no subsystems

#Subsystem sftp /usr/lib/openssh/sftp-server

Subsystem sftp internal-sftp
Match Group sftpusers
ChrootDirectory /sftp/%u
ForceCommand internal-sftp
AllowTCPForwarding no
X11Forwarding no

# Example of overriding settings on a per-user basis

#Match User anoncvs

# X11Forwarding no

# AllowTcpForwarding no

# PermitTTY no

# ForceCommand cvs server
