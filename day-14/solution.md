# Linux Permissions Lab – Solution

## Task 1: Users and Groups

### Commands

```bash
for u in riya amit neha; do sudo useradd -m -s /bin/bash "$u" || true; done
sudo groupadd devops || true
for u in riya amit neha; do sudo usermod -aG devops "$u"; done
for u in riya amit neha; do id "$u"; done
```

Output
uid=1004(amit) gid=1004(amit) groups=1004(amit),100(users),1006(devops)
uid=1005(riya) gid=1005(riya) groups=1005(riya),1006(devops)
uid=1006(neha) gid=1006(neha) groups=1006(neha),1006(devops)

## Task 2: Shared Directory with SGID

```bash
sudo mkdir -p /projects/infra
sudo chown root:devops /projects/infra
sudo chmod 2770 /projects/infra

# Test file creation
sudo -u riya bash -c "echo 'Hello from riya' > /projects/infra/test.txt"
sudo -u amit bash -c "echo 'Hello from amit' >> /projects/infra/test.txt"

ls -l /projects/infra

```

Output
-rw-rw-r-- 1 riya devops 32 Aug 19 test.txt
-rw-rw-r-- 1 amit devops 48 Aug 19 test.txt

## Task 3: Fixing Broken App Directory

```bash
sudo mkdir -p /opt/app
sudo chown root:root /opt/app
sudo chmod 600 /opt/app
sudo useradd -m -s /bin/bash appsvc
sudo chown -R appsvc:appsvc /opt/app
sudo find /opt/app -type d -exec chmod 750 {} \;
sudo find /opt/app -type f -exec chmod 640 {} \;

echo "APP_CONFIG=ok" | sudo tee /opt/app/config.cfg

# Verify
sudo -u appsvc cat /opt/app/config.cfg
cat /opt/app/config.cfg
```

Output
APP_CONFIG=ok
cat: /opt/app/config.cfg: Permission denied

## Task 4: Troubleshooting with namei

```bash
# Break permission
sudo chmod a-x /opt
sudo namei -l /opt/app/config.cfg

# Fix permission
sudo chmod a+rx /opt
sudo namei -l /opt/app/config.cfg

```

Output

Before fix
drw-r--r-- root root opt

After fix
drwxr-xr-x root root opt

## Notes & Learnings

The sgid bit (chmod 2770) ensures files created in a group directory remain owned by that group.

File vs directory permissions matter: x on dirs = “traverse”, not “execute”.

useradd vs adduser: Ubuntu prefers adduser, but for scripting useradd is cleaner.

namei -l is excellent for debugging “permission denied” issues in nested paths.

Principle of least privilege: appsvc can read configs, but other users cannot.
