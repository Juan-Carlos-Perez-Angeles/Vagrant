# Vagrant Multi-VM Infrastructure (Ubuntu Desktop + Windows 11)

This project creates a simple multi-VM environment with **2 Ubuntu Desktop 22.04 VMs** and **1 Windows 11 VM** using **Vagrant** and **VirtualBox**.  
Each Linux VM is pre-configured with SSH, Apache, and OpenSSL. The Windows VM installs **PuTTY** and **Google Chrome**, enables ICMP ping, and provides RDP access.

---

## 📋 Requirements

Make sure the following are installed on your host machine:

- [Vagrant](https://www.vagrantup.com/downloads)
- [VirtualBox](https://www.virtualbox.org/wiki/Downloads)
- [Git](https://git-scm.com/downloads)

### Vagrant Plugins

You’ll need these plugins:

```bash
vagrant plugin install vagrant-reload
```

### Vagrant Boxes

Download the required base boxes:

```bash
vagrant box add caspermeijn/ubuntu-desktop-22.04
vagrant box add gusztavvargadr/windows-11
```

---

## 📂 Project Structure

```
project/
├── Vagrantfile
├── README.md
├── openssl.cnf        # Custom OpenSSL configuration
└── index.html         # Custom Apache welcome page
```

- **openssl.cnf** → will be copied into `/etc/ssl/openssl.cnf` on both Linux VMs.
- **index.html** → will be copied into `/var/www/html/index.html` on both Linux VMs.

---

## 🚀 Usage

### 1. Initialize project
If you cloned this repo, skip this step.  
Otherwise, create a new Vagrant environment:

```bash
vagrant init
```

Replace the generated `Vagrantfile` with the one provided in this project.

---

### 2. Start all VMs

```bash
vagrant up
```

This will:
- Start **linux1** at `192.168.56.10`
- Start **linux2** at `192.168.56.11`
- Start **win** at `192.168.56.20`

---

### 3. Access the machines

- **Linux 1**
  ```bash
  vagrant ssh linux1
  ```
- **Linux 2**
  ```bash
  vagrant ssh linux2
  ```
- **Windows 11**
  ```bash
  vagrant ssh linux2
  ```
  
  or

  - Via RDP: connect to `127.0.0.1:53389`
  - Username: `vagrant`
  - Password: `vagrant`

---

## 🔧 Customization

- Replace `index.html` to change the default Apache page.
- Update `openssl.cnf` to adjust SSL/TLS configuration.
- Edit `Vagrantfile` to change VM resources (CPU, memory, IP addresses).

---

## 🛑 Stopping and Cleaning Up

- Suspend VMs:
  ```bash
  vagrant suspend
  ```
- Halt VMs:
  ```bash
  vagrant halt
  ```
- Destroy everything:
  ```bash
  vagrant destroy -f
  ```

---

## ✅ Notes

- Windows box setup may take longer since Chocolatey installs PuTTY and Chrome.  
