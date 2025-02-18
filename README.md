# pbs-exporter

Fork from https://github.com/rare-magma/pbs-exporter

The idea is to build and offer a Container Image and provide instructions to deploy in Kubernetes.

## Dependencies

- [curl](https://curl.se/)
- [jq](https://stedolan.github.io/jq/)
- Optional: [make](https://www.gnu.org/software/make/) - for automatic installation support

## Relevant documentation

- [Proxmox Backup Server API](https://pbs.proxmox.com/docs/api-viewer/index.html)
- [Proxmox Backup Server API Tokens](https://pbs.proxmox.com/docs/user-management.html#api-tokens)
- [Prometheus Pushgateway](https://github.com/prometheus/pushgateway/blob/master/README.md)
- [Systemd Timers](https://www.freedesktop.org/software/systemd/man/systemd.timer.html)

## Installation

### With Kubernetes

TODO

### With Docker

#### docker-compose

1. Configure `pbs_exporter.conf` (see the configuration section below).
1. Run it.

   ```bash
   docker compose up --detach
   ```

#### docker build & run

1. Build the docker image.

   ```bash
   docker build . --tag pbs-exporter
   ```

1. Configure `pbs_exporter.conf` (see the configuration section below).
1. Run it.

   `docker run --rm --init --tty --interactive --volume $(pwd):/app localhost/pbs-exporter`

<details>
<summary>As normal user</summary>

### With the Makefile

For convenience, you can install this exporter with the following command or follow the manual process described in the next paragraph.

```bash
make install-user
$EDITOR $HOME/.config/pbs_exporter.conf
```

### Manually

1. Copy `pbs_exporter.sh` to `$HOME/.local/bin/` and make it executable.

2. Copy `pbs_exporter.conf` to `$HOME/.config/`, configure it (see the configuration section below) and make it read only.

3. Edit pbs-exporter.service and change the following lines:

```bash
ExecStart=/usr/local/bin/pbs_exporter.sh
EnvironmentFile=/etc/pbs_exporter.conf
```

to

```bash
ExecStart=/home/%u/.local/bin/pbs_exporter.sh
EnvironmentFile=/home/%u/.config/pbs_exporter.conf
```

4. Copy the systemd unit and timer to `$HOME/.config/systemd/user/`:

```bash
cp pbs-exporter.* $HOME/.config/systemd/user/
```

5. and run the following command to activate the timer:

```bash
systemctl --user enable --now pbs-exporter.timer
```

It's possible to trigger the execution by running manually:

```bash
systemctl --user start pbs-exporter.service
```

</details>
<details>
<summary>As root</summary>

### With the Makefile

For convenience, you can install this exporter with the following command or follow the manual process described in the next paragraph.

```bash
sudo make install
sudoedit /etc/pbs_exporter.conf
```

### Manually

1. Copy `pbs_exporter.sh` to `/usr/local/bin/` and make it executable.

2. Copy `pbs_exporter.conf` to `/etc/`, configure it (see the configuration section below) and make it read only.

3. Copy the systemd unit and timer to `/etc/systemd/system/`:

```bash
sudo cp pbs-exporter.* /etc/systemd/system/
```

4. and run the following command to activate the timer:

```bash
sudo systemctl enable --now pbs-exporter.timer
```

It's possible to trigger the execution by running manually:

```bash
sudo systemctl start pbs-exporter.service
```

</details>
<br/>

### Config file

The config file has a few options:

```bash
PBS_API_TOKEN_NAME='user@pam!prometheus'
PBS_API_TOKEN='123e4567-e89b-12d3-a456-426614174000'
PBS_URL='https://pbs.example.com'
PUSHGATEWAY_URL='https://pushgateway.example.com'
```

- `PBS_API_TOKEN_NAME` should be the value in the "Token name" column in the Proxmox Backup Server user interface's `Configuration - Access Control - Api Token` page.
- `PBS_API_TOKEN` should be the value shown when the API Token was created.
  - This token should have at least the `Datastore.Audit` access role assigned to it and the path set to `/datastore`.
- `PBS_URL` should be the same URL as used to access the Proxmox Backup Server user interface
- `PUSHGATEWAY_URL` should be a valid URL for the [push gateway](https://github.com/prometheus/pushgateway).

### Troubleshooting

<details>
<summary>As normal user</summary>

Run the script manually with bash set to trace:

```bash
bash -x $HOME/.local/bin/pbs_exporter.sh
```

Check the systemd service logs and timer info with:

```bash
journalctl --user --unit pbs-exporter.service
systemctl --user list-timers
```

</details>
<details>
<summary>As root</summary>

Run the script manually with bash set to trace:

```bash
sudo bash -x /usr/local/bin/pbs_exporter.sh
```

Check the systemd service logs and timer info with:

```bash
journalctl --unit pbs-exporter.service
systemctl list-timers
```

</details>
<br>

## Exported metrics per PBS store

The following metrics are available for all stores currently not in maintenance mode:

- pbs_available: The available bytes of the underlying storage.
- pbs_size: The size of the underlying storage in bytes.
- pbs_used: The used bytes of the underlying storage.
- pbs_snapshot_count: The total number of backups.
- pbs_snapshot_vm_count: The total number of backups per VM.

## Exported metrics example

```bash
# HELP pbs_available The available bytes of the underlying storage.
# TYPE pbs_available gauge
# HELP pbs_size The size of the underlying storage in bytes.
# TYPE pbs_size gauge
# HELP pbs_used The used bytes of the underlying storage.
# TYPE pbs_used gauge
# HELP pbs_snapshot_count The total number of backups.
# TYPE pbs_snapshot_count gauge
# HELP pbs_snapshot_vm_count The total number of backups per VM.
# TYPE pbs_snapshot_vm_count gauge
pbs_available {host="pbs.example.com", store="store2"} 567317757952
pbs_size {host="pbs.example.com", store="store2"} 691587252224
pbs_used {host="pbs.example.com", store="store2"} 124269494272
pbs_snapshot_count {host="pbs.example.com", store="store2"} 295
pbs_snapshot_vm_count {host="pbs.example.com", store="store2", vm_id="101"} 11
pbs_snapshot_vm_count {host="pbs.example.com", store="store2", vm_id="102"} 12
pbs_snapshot_vm_count {host="pbs.example.com", store="store2", vm_id="103"} 10
```

## Uninstallation

<details>
<summary>As normal user</summary>

### With the Makefile

For convenience, you can uninstall this exporter with the following command or follow the process described in the next paragraph.

```bash
make uninstall-user
```

### Manually

Run the following command to deactivate the timer:

```bash
systemctl --user disable --now pbs-exporter.timer
```

Delete the following files:

```bash
$HOME/.local/bin/pbs_exporter.sh
$HOME/.config/pbs_exporter.conf
$HOME/.config/systemd/user/pbs-exporter.timer
$HOME/.config/systemd/user/pbs-exporter.service
```

</details>
<details>
<summary>As root</summary>

### With the Makefile

For convenience, you can uninstall this exporter with the following command or follow the process described in the next paragraph.

```bash
sudo make uninstall
```

### Manually

Run the following command to deactivate the timer:

```bash
sudo systemctl disable --now pbs-exporter.timer
```

Delete the following files:

```bash
/usr/local/bin/pbs_exporter.sh
/etc/pbs_exporter.conf
/etc/systemd/system/pbs-exporter.timer
/etc/systemd/system/pbs-exporter.service
```

</details>
<br>

## Credits

- [reddec/compose-scheduler](https://github.com/reddec/compose-scheduler)

This project takes inspiration from the following:

- [mad-ady/prometheus-borg-exporter](https://github.com/mad-ady/prometheus-borg-exporter)
- [OVYA/prometheus-borg-exporter](https://github.com/OVYA/prometheus-borg-exporter)
