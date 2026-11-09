# SSH Remote Server Setup

Project from roadmap.sh: set up a remote Linux server and connect over SSH with two different key pairs.

## Server
- Provider: AWS EC2 (Free plan), Region us-east-2 (Ohio)
- OS: Ubuntu Server LTS, instance type: [e.g. t3.micro]
- Security group: one inbound rule, SSH (port 22) from my IP only

## 1. Create two SSH key pairs
```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
ssh-keygen -t ed25519 -f ~/.ssh/aws_key1 -C "key1"
ssh-keygen -t ed25519 -f ~/.ssh/aws_key2 -C "key2"
```
Each key has a private file (stays on my machine) and a `.pub` file (goes on the server). I set a passphrase on both.

## 2. Import key 1 and launch the instance
```bash
aws ec2 import-key-pair --key-name ssh-project-key1 \
  --public-key-material fileb://$HOME/.ssh/aws_key1.pub \
  --region us-east-2 --profile <profile>
```
Then launched the instance in the EC2 console with `ssh-project-key1` as its key pair.

## 3. Connect with key 1
```bash
ssh -i ~/.ssh/aws_key1 ubuntu@<server-ip>
```

## 4. Add key 2
```bash
ssh-copy-id -f -i ~/.ssh/aws_key2.pub -o IdentityFile=$HOME/.ssh/aws_key1 ubuntu@<server-ip>
```
Without `-f`, `ssh-copy-id` skipped the key. Its pre-check logged in with key 1 and wrongly concluded key 2 was already installed. `-f` skips that check. The key goes into `~/.ssh/authorized_keys` on the server.

```bash
ssh -o IdentitiesOnly=yes -i ~/.ssh/aws_key2 ubuntu@<server-ip>
```

## 5. SSH config aliases
`~/.ssh/config`:
```
Host aws-key1
    HostName <server-ip>
    User ubuntu
    IdentityFile ~/.ssh/aws_key1
    IdentitiesOnly yes

Host aws-key2
    HostName <server-ip>
    User ubuntu
    IdentityFile ~/.ssh/aws_key2
    IdentitiesOnly yes
```
```bash
chmod 600 ~/.ssh/config
ssh aws-key1
ssh aws-key2
```

## 6. Stretch goal: fail2ban
```bash
sudo apt update && sudo apt install -y fail2ban
sudo nano /etc/fail2ban/jail.local
```
```
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5

[sshd]
enabled = true
```
```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```
Result: the `sshd` jail is active, reading from the systemd journal.

## What I learned
- Public key vs private key, and what `authorized_keys` is
- Why the security group should allow SSH from my IP only
- `ssh-copy-id` pre-checks can be misled, so I used `-f`
- The public IP changes if the instance is stopped and started, so the config `HostName` needs updating

Private keys are never committed to this repo.
