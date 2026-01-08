# VMware (ESXi 8.x) Debian Template for OpenWrt Builds

This guide consolidates the VMware + OpenWrt build VM workflow in a clear, step-by-step format. It covers:

- Debian 12 template creation on ESXi 8.x
- Partitioning for OpenWrt builds
- Cloud-init and vSphere Guest Customization
- Optional seed.iso (NoCloud)
- OpenWrt build steps on the VM
- PowerCLI automation hooks

## 1. VM Creation (ESXi 8.x)

1) Create a new VM

- Guest OS: Debian 12 (64-bit)
- Firmware: UEFI
- Network: VMXNET3
- Controller: PVSCSI (or LSI SAS)
- Disk: size per your build needs

2) Attach Debian ISO and install

- Minimal install (no GUI)
- Select only: "SSH server" + "standard system utilities"

## 2. Partitioning (Recommended for OpenWrt Builds)

Use a separate build volume to avoid filling the OS disk.

Example layout:

- Disk 1 (OS)
  - `/` 40-60G (ext4)
  - `swap` 4-8G (optional)
- Disk 2 (build)
  - `/work` or `/openwrt` 200G+ (ext4 or xfs)

Mount build disk (example):

```bash
sudo mkfs.ext4 /dev/sdb
sudo mkdir -p /work
sudo mount /dev/sdb /work
```

Persist in `/etc/fstab`:

```bash
/dev/sdb  /work  ext4  defaults  0  2
```

## 3. Base Packages and Tools

Install core tools and OpenWrt build dependencies:

```bash
sudo apt-get update
sudo apt-get install -y \
  open-vm-tools cloud-init sudo ca-certificates \
  binutils bison build-essential bzip2 clang coreutils diffutils \
  file findutils flex g++ g++-multilib gawk gcc-multilib gettext git grep gzip \
  libncurses5-dev libssl-dev patch perl python3 python3-distutils \
  python3-pyelftools python3-setuptools rsync subversion swig tar \
  unzip util-linux wget which zlib1g-dev
```

## 4. Create a Build User

```bash
sudo useradd -m -s /bin/bash builder
sudo usermod -aG sudo builder
```

## 5. SSH and Timezone

Timezone:

```bash
sudo timedatectl set-timezone Asia/Shanghai
```

SSH hardening (optional but recommended):

- Edit `/etc/ssh/sshd_config`
  - `PermitRootLogin no`
  - `PasswordAuthentication yes` or `no` if using keys only

Restart SSH:

```bash
sudo systemctl restart ssh
```

## 6. Cloud-init + vSphere Guest Customization

### 6.1 How it works

- vSphere Guest Customization injects hostname/network/user settings.
- cloud-init reads VMware/OVF data and applies those settings on first boot.

### 6.2 Ensure VMware datasource is enabled

```bash
sudo sh -c 'cat > /etc/cloud/cloud.cfg.d/99-vmware.cfg <<EOF
# VMware datasource order
 datasource_list: [ VMware, OVF, NoCloud, None ]
EOF'
```

### 6.3 Template cleanup

Before turning the VM into a template:

```bash
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo cloud-init clean --logs
```

Then shut down and convert to Template.

## 7. Optional NoCloud seed.iso (user-data)

Use this if you want full control beyond vSphere customization.

### 7.1 user-data (example)

```yaml
#cloud-config
hostname: openwrt-build
manage_etc_hosts: true
timezone: Asia/Shanghai

users:
  - name: builder
    gecos: OpenWrt Builder
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    groups: sudo
    lock_passwd: false
    passwd: $6$REPLACE_WITH_SHA512_HASH
    ssh_authorized_keys:
      - ssh-ed25519 AAAA...your_key_here...

package_update: true
package_upgrade: false
packages:
  - open-vm-tools
  - sudo
  - ca-certificates
  - binutils
  - bison
  - build-essential
  - bzip2
  - clang
  - coreutils
  - diffutils
  - file
  - findutils
  - flex
  - g++
  - g++-multilib
  - gawk
  - gcc-multilib
  - gettext
  - git
  - grep
  - gzip
  - libncurses5-dev
  - libssl-dev
  - patch
  - perl
  - python3
  - python3-distutils
  - python3-pyelftools
  - python3-setuptools
  - rsync
  - subversion
  - swig
  - tar
  - unzip
  - util-linux
  - wget
  - which
  - zlib1g-dev

runcmd:
  - [ mkdir, -p, /work ]
```

### 7.2 meta-data (example)

```yaml
instance-id: openwrt-build-001
local-hostname: openwrt-build-001
```

### 7.3 Build seed.iso (macOS)

```bash
mkdir -p seed
cp user-data seed/user-data
cp meta-data seed/meta-data
hdiutil makehybrid -o seed.iso -hfs -joliet seed
```

Mount `seed.iso` to the VM before first boot. After first boot, remove it to avoid re-running cloud-init.

## 8. Clone from Template (vSphere Guest Customization)

1) Create a Guest Customization Spec in vSphere
- Hostname
- Network (DHCP or static)
- DNS
- Timezone
- Password (if needed)

2) Clone VM from Template and apply the spec
3) First boot applies the configuration

## 9. PowerCLI Automation (Example)

```powershell
Connect-VIServer vcenter.example.com

$template = Get-Template -Name "Debian12-OpenWrt-Template"
$specBase = Get-OSCustomizationSpec -Name "Debian-OpenWrt-Spec"
$spec = New-OSCustomizationSpec -Spec $specBase -Name "tmp-openwrt-build-001" -Type NonPersistent
$spec | Set-OSCustomizationSpec -NamingScheme Fixed -NamingPrefix "openwrt-build-001"

$vmhost = Get-VMHost -Name "esxi-01"
$ds = Get-Datastore -Name "datastore1"
$vm = New-VM -Name "openwrt-build-001" -Template $template -VMHost $vmhost -Datastore $ds -OSCustomizationSpec $spec

# Optional: mount seed.iso (upload it to datastore first)
$isoPath = "[datastore1] iso/seed.iso"
Get-CDDrive -VM $vm | Set-CDDrive -IsoPath $isoPath -Connected $true -StartConnected $true -Confirm:$false

Start-VM -VM $vm
```

## 10. OpenWrt Build on the VM

1) Switch to build user and clone or use existing source tree

```bash
su - builder
cd /work
```

2) Update feeds and configure

```bash
./scripts/feeds update -a
./scripts/feeds install -a
make menuconfig
```

3) Build

```bash
make -j$(nproc)
```

If build fails, re-run with verbose output:

```bash
make -j1 V=s
```

## 11. Notes on Official Release Repro

To match official release packages exactly:

- Use the exact release tag (example: `v24.10.5`)
- Fetch `config.buildinfo` and `feeds.buildinfo`
- Apply them before build
- Compare `packages.manifest` after build

