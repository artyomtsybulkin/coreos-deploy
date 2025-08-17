# coreos-deploy

Fedora CoreOS deployment procedures

How to geenrate password and public key.

- user `core` has no password, only ssh key
- user `root` has no permissions for ssh login by default
- default `root` password: `pa$$w0rd`

```bash
mkpasswd --method=yescrypt pa$$w0rd
ssh-keygen -t ed25519 -N "" -f vm-podman -C "Comment" -q
```

Validate and generate Ignition config based on Butane config.

```bash
dnf install epel-release -y && dnf update -y
dnf install butane ignition mkpasswd ignition-validate

butane --pretty --strict podman.bu > podman.ign
ignition-validate podman.ign
```

There is how to host Ignition config locally.

```bash
git clone https://github.com/artyomtsybulkin/coreos-deploy.git
cd coreos-deploy/
python -m http.server -b 0.0.0.0 80
```

Start system using instalation iso image and begin installation using `coreos-installer`.  
Or clean disks in case of reinstalling by input whole snippet.

```bash
sudo -i
wipefs -a /dev/sda
wipefs -a /dev/sdb
sgdisk --zap-all /dev/sda
sgdisk --zap-all /dev/sdb
coreos-installer install /dev/sda \
    --insecure-ignition \
    --ignition-url http://172.23.32.1/podman.ign
poweroff
```

I prefer `poweroff` then eject iso image and start VM again after.  
Post-installation script, execute with `sudo` commands 1-3 and last as `core` user:

```bash
sudo chmod +x /var/mnt/post-install.sh
sudo chmod +x /var/mnt/podman.sh
sudo /bin/bash /var/mnt/post-install.sh
/bin/bash /var/mnt/podman.sh
```

Configure `firewalld`:

```bash
sudo firewall-cmd --permanent --add-rich-rule='
    rule family="ipv4" source address="192.168.1.100" port protocol="tcp" port="22" accept'
sudo firewall-cmd --permanent --add-rich-rule='
    rule family="ipv4" port protocol="tcp" port="22" reject'
```

Sompose security options definition.

| Field             | Description                                                       |
| ----------------- | ----------------------------------------------------------------- |
| `mem_limit`       | Limit container memory usage                                      |
| `cpus`            | Limit CPU usage                                                   |
| `pids_limit`      | Prevent fork bombs                                                |
| `cap_drop: ALL`   | Drop all Linux capabilities                                       |
| `cap_add`         | Add only specific capability (`NET_BIND_SERVICE` for ports <1024) |
| `user`            | Run container as non-root (must exist in image)                   |
| `read_only: true` | Make container's root FS read-only                                |
| `tmpfs`           | Create a temporary in-memory FS for `/tmp` or other paths         |
| `security_opt`    | Disable gaining new privileges inside container                   |
| `logging.driver`  | Log to `journald`, integrates with `journalctl`                   |
