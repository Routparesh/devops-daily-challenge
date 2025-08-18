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
