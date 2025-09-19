# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.boot_timeout = 1800
  config.vm.provider "virtualbox" do |vb|
    vb.cpus = 2
    vb.gui  = true
  end

  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.enabled = false
  end

  linux_common = <<-'SHELL'
    set -euo pipefail
    export DEBIAN_FRONTEND=noninteractive

    sudo apt-get update -y
    sudo apt-get install -y openssh-server apache2 curl ca-certificates net-tools

    sudo systemctl enable --now ssh
    sudo systemctl enable --now apache2

    echo 'vagrant:vagrant' | sudo chpasswd
    sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
    sudo sed -i 's/^#\?KbdInteractiveAuthentication.*/KbdInteractiveAuthentication no/' /etc/ssh/sshd_config
    sudo sshd -t && sudo systemctl restart ssh || sudo systemctl start ssh

    if [ -f /vagrant/index.html ]; then
      sudo cp /vagrant/index.html /var/www/html/index.html
      echo "index.html personalizado aplicado en $(hostname -s)"
    else
      IP_ADDR="$(ip -o -4 addr show scope global | awk '{print $4}' | cut -d/ -f1 | head -n1 || true)"
      HOSTNAME="$(hostname -s)"
      echo "<h1>Hola desde ${HOSTNAME} (${IP_ADDR})</h1>" | sudo tee /var/www/html/index.html >/dev/null
      echo "index.html generado automáticamente"
    fi

    if [ -f /vagrant/openssl.cnf ]; then
      sudo cp /etc/ssl/openssl.cnf /etc/ssl/openssl.cnf.bak || true
      sudo cp /vagrant/openssl.cnf /etc/ssl/openssl.cnf
      echo "openssl.cnf personalizado aplicado en $(hostname -s)"
    else
      echo "Advertencia: /vagrant/openssl.cnf no encontrado, usando el predeterminado"
    fi
  SHELL

  linux_enable_password = <<-'SHELL'
    set -e
    sudo tee /etc/ssh/sshd_config.d/zzz-allow-password.conf >/dev/null <<'EOF'
PasswordAuthentication yes
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no
UsePAM yes
PubkeyAuthentication yes
AuthenticationMethods any
EOF
    echo 'vagrant:vagrant' | sudo chpasswd
    sudo sshd -t && sudo systemctl restart ssh
  SHELL

  config.vm.define "linux1" do |linux|
    linux.vm.box = "caspermeijn/ubuntu-desktop-22.04"
    linux.vm.hostname = "linux1"
    linux.vm.network "private_network", ip: "192.168.56.10"
    linux.vm.provider "virtualbox" do |vb|
      vb.memory = 2048
    end
    linux.vm.provision "shell", inline: linux_common
    linux.vm.provision "shell", run: "always", inline: linux_enable_password
  end

  config.vm.define "linux2" do |linux|
    linux.vm.box = "caspermeijn/ubuntu-desktop-22.04"
    linux.vm.hostname = "linux2"
    linux.vm.network "private_network", ip: "192.168.56.11"
    linux.vm.provider "virtualbox" do |vb|
      vb.memory = 2048
    end
    linux.vm.provision "shell", inline: linux_common
    linux.vm.provision "shell", run: "always", inline: linux_enable_password
  end

  config.vm.define "win" do |win|
    win.vm.box = "gusztavvargadr/windows-11"
    win.vm.hostname = "win"
    win.vm.network "private_network", ip: "192.168.56.20"
    win.vm.guest = :windows
    win.vm.communicator = "winrm"
    win.winrm.username = "vagrant"
    win.winrm.password = "vagrant"
    win.winrm.transport = :negotiate
    win.winrm.timeout = 1800
    win.winrm.retry_limit = 60
    win.winrm.retry_delay = 10
    win.vm.network "forwarded_port", guest: 3389, host: 53389, auto_correct: true
    win.vm.synced_folder ".", "C:/vagrant", disabled: true
    win.vm.provider "virtualbox" do |vb|
      vb.memory = 4096
      vb.cpus   = 2
      vb.customize ["modifyvm", :id, "--vram", "128"]
      vb.customize ["modifyvm", :id, "--graphicscontroller", "vboxsvga"]
      vb.customize ["modifyvm", :id, "--ioapic", "on"]
      vb.customize ["modifyvm", :id, "--nictype2", "82540EM"]
    end

    win.vm.provision "shell",
      name: "network-private-and-icmp",
      privileged: true,
      powershell_elevated_interactive: true,
      inline: <<-'POWERSHELL'
$ErrorActionPreference = "Stop"

try {
  $ip = Get-NetIPAddress -IPAddress 192.168.56.20 -ErrorAction Stop
  if ((Get-NetConnectionProfile -InterfaceIndex $ip.InterfaceIndex).NetworkCategory -ne 'Private') {
    Set-NetConnectionProfile -InterfaceIndex $ip.InterfaceIndex -NetworkCategory Private
  }
} catch {
  Get-NetAdapter | Where-Object {$_.InterfaceDescription -like "*VirtualBox*"} | ForEach-Object {
    Set-NetConnectionProfile -InterfaceAlias $_.Name -NetworkCategory Private -ErrorAction SilentlyContinue
  }
}

Get-NetFirewallRule -Direction Inbound | Where-Object { $_.DisplayName -like "*Echo Request*" } | Enable-NetFirewallRule
New-NetFirewallRule -DisplayName "Allow ICMPv4 Echo In (Vagrant)" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow -Profile Any -ErrorAction SilentlyContinue
      POWERSHELL

  win.vm.provision "shell",
    name: "tools-putty-chrome",
    privileged: true,
    powershell_elevated_interactive: true,
    inline: <<-'POWERSHELL'
  $ErrorActionPreference = "Stop"

  try { Set-Service -Name sshd -StartupType Automatic; Start-Service sshd } catch { }

  if (-not (Get-Command choco.exe -ErrorAction SilentlyContinue)) {
    Set-ExecutionPolicy Bypass -Scope Process -Force
    [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
    Invoke-Expression ((New-Object Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
  }

  choco install putty -y --no-progress

  function Test-ChromeInstalled {
    return (Test-Path "C:\Program Files\Google\Chrome\Application\chrome.exe") -or
          (Test-Path "C:\Program Files (x86)\Google\Chrome\Application\chrome.exe")
  }

  if (-not (Test-ChromeInstalled)) {
    $winget = Get-Command winget.exe -ErrorAction SilentlyContinue
    if ($winget) {
      & $winget install --id Google.Chrome `
        --source winget `
        --silent `
        --accept-source-agreements `
        --accept-package-agreements `
        --disable-interactivity | Out-Null
    } else {
      choco install googlechrome -y --no-progress --ignore-checksums
    }

    $tries = 0
    while (-not (Test-ChromeInstalled) -and $tries -lt 30) {
      Start-Sleep -Seconds 10
      $tries++
    }
  }

  $puttyPath = $null
  try { $puttyPath = (Get-Command putty.exe -ErrorAction Stop).Source } catch {
    $candidates = @(
      "C:\Program Files\PuTTY\putty.exe",
      "C:\Program Files (x86)\PuTTY\putty.exe",
      "C:\ProgramData\chocolatey\lib\putty.portable\tools\PUTTY.EXE"
    )
    foreach ($c in $candidates) { if (Test-Path $c) { $puttyPath = $c; break } }
  }

  $chromePath = "C:\Program Files\Google\Chrome\Application\chrome.exe"
  if (-not (Test-Path $chromePath)) {
    $chromePath = "C:\Program Files (x86)\Google\Chrome\Application\chrome.exe"
  }

  $publicDesktop = Join-Path $Env:Public "Desktop"
  New-Item -ItemType Directory -Force -Path $publicDesktop | Out-Null
  $wsh = New-Object -ComObject WScript.Shell

  if ($puttyPath -and (Test-Path $puttyPath)) {
    $lnk1 = Join-Path $publicDesktop "PuTTY.lnk"
    $s1 = $wsh.CreateShortcut($lnk1)
    $s1.TargetPath = $puttyPath
    $s1.Save()
  }

  if (Test-Path $chromePath) {
    $lnk2 = Join-Path $publicDesktop "Google Chrome.lnk"
    $s2 = $wsh.CreateShortcut($lnk2)
    $s2.TargetPath = $chromePath
    $s2.Save()
  } else {
    Write-Warning "Chrome no se pudo instalar (ni con winget ni con choco). Puedes instalarlo manualmente luego."
  }
  POWERSHELL
  end
end
