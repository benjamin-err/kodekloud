# Jenkins Setup Notes

Personal notes from the KodeKloud Jenkins course: how I installed Jenkins locally, and why each step was done. These notes grow as the course goes on.

## Contents

1. [Environment](#environment)
2. [Install Java](#1-install-java)
3. [Add the Jenkins package repository](#2-add-the-jenkins-package-repository)
4. [Install, start and unlock Jenkins](#3-install-start-and-unlock-jenkins)
5. [Create an SSH key pair](#4-create-an-ssh-key-pair)
6. [Enable the Jenkins SSH server](#5-enable-the-jenkins-ssh-server)
7. [Add the public key to the Jenkins user](#6-add-the-public-key-to-the-jenkins-user)
8. [Connect to Jenkins over SSH](#7-connect-to-jenkins-over-ssh)
9. [Use the Jenkins CLI](#8-use-the-jenkins-cli)
10. [Troubleshooting](#troubleshooting)
11. [Quick reference](#quick-reference)

---

## Environment

| Component | Value |
|---|---|
| Host OS | Windows 11 |
| Linux | Ubuntu 24.04 LTS on WSL |
| Java | OpenJDK 21 |
| Jenkins | 2.580.2 (LTS, from the official `debian-stable` repo) |
| Jenkins web UI | `http://localhost:8080` |
| Jenkins SSH port | `8088` |
| Working folder | `~/kodekloud` (SSH keys and `jenkins-cli.jar`) |

> **Note:** The course labs use Red Hat-style servers (`yum`, `rpm`). Ubuntu uses `apt` instead, so the lab commands were translated. Both versions are shown where they differ.

---

## 1. Install Java

Jenkins is a Java application, so it needs a Java runtime. Jenkins 2.5xx requires **Java 17 or 21**.

```bash
sudo apt update
sudo apt install -y openjdk-21-jdk
```

| Course (yum) | Ubuntu (apt) | Why |
|---|---|---|
| `java-21-openjdk` + `java-21-openjdk-devel` | `openjdk-21-jdk` | On Ubuntu the JDK package contains both the runtime and the developer tools. |

Check it:

```bash
java -version
```

---

## 2. Add the Jenkins package repository

Ubuntu's own repositories don't include Jenkins. You add the official Jenkins repository, plus the key apt uses to check that its packages are genuine.

```bash
# 1. Create the folder for repository signing keys
sudo install -d -m 0755 /etc/apt/keyrings

# 2. Download the Jenkins signing key
sudo curl -fsSL -o /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

# 3. Register the repository, trusting only that key for it
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" \
  | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# 4. Refresh the package lists so apt sees the new repository
sudo apt update
```

What changed compared with the course commands:

| Course (yum) | Ubuntu (apt) | Why |
|---|---|---|
| `curl … -o /etc/yum.repos.d/jenkins.repo …/redhat-stable/jenkins.repo` | One `deb …` line in `/etc/apt/sources.list.d/jenkins.list`, pointing at `debian-stable` | yum reads `.repo` files. apt reads `deb` lines, and Jenkins has a separate repository for `.deb` packages. |
| `rpm --import …/jenkins.io-2023.key` | Save the key to `/etc/apt/keyrings/` and reference it with `signed-by=` | apt has no global "import key" step. `signed-by` limits the key to this one repository, which is safer. |
| *(not needed)* | Extra `sudo apt update` | yum refreshes repository data on its own. apt only does it when you run `apt update`. |

> ⚠️ **Signing key changed.** The course uses `jenkins.io-2023.key`. Jenkins has since switched to `jenkins.io-2026.key`. With the old key, `apt update` rejects the repository and `apt install jenkins` fails with *"Package 'jenkins' has no installation candidate"*. If this happens again, the current key filename is listed on [pkg.jenkins.io/debian-stable](https://pkg.jenkins.io/debian-stable/).

---

## 3. Install, start and unlock Jenkins

```bash
sudo apt install -y jenkins
sudo systemctl enable --now jenkins
```

- `apt install jenkins` installs Jenkins as a system service that runs as its own `jenkins` user. Its data lives in `/var/lib/jenkins`.
- `systemctl enable --now` starts Jenkins now (`--now`) and on every boot (`enable`).

Check that it's running:

```bash
systemctl status jenkins
```

**Unlock Jenkins.** On first start, Jenkins creates a one-time admin password so that nobody else can take over a fresh install:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Open `http://localhost:8080`, paste the password, install the suggested plugins, and create your admin user (here: `benjamin`).

> **WSL note:** `systemctl` only works when systemd is enabled in WSL. It is on by default in recent Ubuntu WSL installs.

---

## 4. Create an SSH key pair

The key pair lets you log in to Jenkins over SSH without a password. It is used in steps 6–8.

```bash
mkdir -p ~/kodekloud
ssh-keygen -t ed25519 -C jenkins -f ~/kodekloud/jenkins_ed25519 -N ''
chmod 600 ~/kodekloud/jenkins_ed25519
```

| Option | Meaning |
|---|---|
| `-t ed25519` | Key type. Ed25519 is the modern default: short keys, fast, secure. |
| `-C jenkins` | Comment at the end of the public key. It's a label only. |
| `-f …` | Where to save the key. This creates two files: `jenkins_ed25519` and `jenkins_ed25519.pub`. |
| `-N ''` | No passphrase, so the key can be used by tools without anyone typing a password. |
| `chmod 600` | Only you can read the private key. SSH refuses keys that other users can read. |

**Which file is which**

| File | Name | Share it? |
|---|---|---|
| `jenkins_ed25519` | Private key | **Never.** It stays on your machine and proves who you are. |
| `jenkins_ed25519.pub` | Public key | Yes. You give it to the system you want to log in to. |

**What the public key looks like**

```
ssh-ed25519  AAAAC3NzaC1lZDI1NTE5AAAAI…  jenkins
└ key type   └ key data (base64)          └ comment
```

The key data starts with the same characters for every Ed25519 key, because it begins by encoding the key type name. Only the last part is unique to your key.

> **Where the key goes depends on which way the connection runs:**
> - **You → Jenkins** (SSH/CLI login, this guide): the **public** key goes on your Jenkins user.
> - **Jenkins → something else** (GitHub, agents, servers): the **private** key goes into *Manage Jenkins → Credentials* as *SSH Username with private key*, and the public key goes on the other system.

---

## 5. Enable the Jenkins SSH server

Jenkins can run its own SSH server, so you can control it from a terminal. It needs the **SSH server** plugin, and the plugin is **disabled by default**.

1. *Manage Jenkins → Plugins → Available plugins*: install **SSH server**.
2. *Manage Jenkins → Security → SSH Server*: select **Fixed** and set the port to `8088`, then **Save**.

Check it. When the SSH server is running, Jenkins announces its address in an HTTP response header:

```bash
curl -s -D - -o /dev/null http://localhost:8080/login | grep -i x-ssh-endpoint
# X-SSH-Endpoint: localhost:8088
```

`-D -` prints only the response headers, which makes the output easy to `grep`. No output means the SSH server is still disabled.

---

## 6. Add the public key to the Jenkins user

Go to *your user (top right) → Security*.

![Jenkins user Security page before the SSH server plugin is enabled](images/jenkins-user-security.png)

*Before the SSH server is enabled, this page only shows API Token, Password and Session Termination.*

Once the plugin is installed and the port is enabled (step 5), an **SSH Public Keys** box appears on this page. Paste the contents of the public key and click **Save**:

```bash
cat ~/kodekloud/jenkins_ed25519.pub
```

---

## 7. Connect to Jenkins over SSH

```bash
ssh -i ~/kodekloud/jenkins_ed25519 -l benjamin -p 8088 localhost who-am-i
```

| Part | Meaning |
|---|---|
| `-i …` | Which private key to use. Needed because the key isn't in the default `~/.ssh` folder. |
| `-l benjamin` | Jenkins username (shown in the URL of your profile page). |
| `-p 8088` | Jenkins SSH port, not the normal SSH port 22. |
| `localhost` | Jenkins runs on this machine. |
| `who-am-i` | A Jenkins CLI command. `help` lists all of them. |

Expected output:

```
Authenticated as: benjamin
Authorities:
  authenticated
```

> **`jenkins-server` vs `localhost`:** The course labs use the hostname `jenkins-server`, which only exists on the lab network. Locally it fails with *"Could not resolve hostname"*. Use `localhost`, or add an alias:
> ```bash
> echo "127.0.0.1 jenkins-server" | sudo tee -a /etc/hosts
> ```
> WSL may regenerate `/etc/hosts` on restart. To stop that, add `generateHosts = false` under `[network]` in `/etc/wsl.conf`.

---

## 8. Use the Jenkins CLI

`jenkins-cli.jar` is the official command-line client. Every Jenkins server serves its own matching version:

```bash
cd ~/kodekloud
wget http://localhost:8080/jnlpJars/jenkins-cli.jar
java -jar jenkins-cli.jar -s http://localhost:8080 -version
```

It can log in two ways:

**With the SSH key** (uses the SSH port from step 5):

```bash
java -jar jenkins-cli.jar -s http://localhost:8080 \
  -ssh -user benjamin -i ~/kodekloud/jenkins_ed25519 who-am-i
```

It prints some `INFO` log lines first. Add `2>/dev/null` to hide them.

**With an API token** (works over plain HTTP, no SSH server needed):

```bash
java -jar jenkins-cli.jar -s http://localhost:8080 -auth benjamin:<API_TOKEN> who-am-i
```

Create the token under *your user → Security → API Token → Add new token*. It is shown **only once**, so copy it right away.

> Never commit API tokens or private keys to Git.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `yum: command not found` / can't install yum | Ubuntu uses `apt`, not `yum` | Translate the commands (see steps 1–2). |
| `Package 'jenkins' has no installation candidate` | `apt update` rejected the Jenkins repository, usually because the signing key is outdated | Download the current key (step 2), then `sudo apt update`. |
| No **SSH Public Keys** box on the user Security page | SSH server plugin missing or not loaded | Install **SSH server**, enable the port, restart Jenkins if needed (step 5). |
| `curl … \| grep x-ssh-endpoint` prints nothing | SSH port is set to **Disabled** | *Manage Jenkins → Security → SSH Server → Fixed* (step 5). |
| `Could not resolve hostname jenkins-server` | Lab-only hostname | Use `localhost` or add an `/etc/hosts` alias (step 7). |
| `Permission denied (publickey)` | Public key not saved on the Jenkins user, or wrong username | Paste the `.pub` key into the user's Security page and check the username (step 6). |

---

## Quick reference

```bash
# Service
sudo systemctl status jenkins
sudo systemctl restart jenkins

# Initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# Is the SSH server on?
curl -s -D - -o /dev/null http://localhost:8080/login | grep -i x-ssh-endpoint

# SSH login test
ssh -i ~/kodekloud/jenkins_ed25519 -l benjamin -p 8088 localhost who-am-i

# CLI over SSH
java -jar ~/kodekloud/jenkins-cli.jar -s http://localhost:8080 \
  -ssh -user benjamin -i ~/kodekloud/jenkins_ed25519 help
```
