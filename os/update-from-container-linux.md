# Updating from CoreOS Container Linux

If you already have CoreOS Container Linux clusters and can't or don't want to freshly install Flatcar Container Linux, you can update to Flatcar Container Linux directly from CoreOS Container Linux by performing the following steps.

**NOTE:** General differences when [migrating from CoreOS Container Linux](migrate-from-container-linux.md) also apply.

At [the end of the section](#all-steps-in-one-script) you can find the [update-to-flatcar.sh](/update-to-flatcar.sh) script that does all steps for you.

## Getting the public update key

First, you need to get Flatcar's public update key:

```shell
curl -L -o /tmp/key https://raw.githubusercontent.com/flatcar-linux/coreos-overlay/flatcar-master/coreos-base/coreos-au-key/files/official-v2.pub.pem
```

Since the `/usr` partition is read-only, to allow the updater to use the new key, you need to bind-mount it:

```shell
sudo mount --bind /tmp/key /usr/share/update_engine/update-payload-key.pub.pem
```

## Modifying the configuration files

Migrate your `/etc/coreos/update.conf` settings but keep a symlink:

```shell
sudo mv /etc/coreos /etc/flatcar
sudo ln -s flatcar /etc/coreos
```

Now, you need to point update_engine to Flatcar's update server by setting the `SERVER` configuration option in `/etc/flatcar/update.conf`:

```shell
SERVER=https://public.update.flatcar-linux.net/v1/update/
```

To make sure you get an update even if you're running the same CoreOS Container Linux version as the latest Flatcar Container Linux, you need to force an update by clearing the current version number from the `release` file.
This file also lives in the `/usr` partition so you need to do a bind-mount again:

```shell
cp /usr/share/coreos/release /tmp
sudo mount --bind /tmp/release /usr/share/coreos/release
```

Then, you need to edit `/usr/share/coreos/release` and replace the value of `COREOS_RELEASE_VERSION` with `0.0.0`:

```shell
COREOS_RELEASE_VERSION=0.0.0
```

Migrate any cloud-config `user_data` if it exists in `/var/lib/coreos-install/user_data` (e.g., in bare metal installations):

```shell
test -d /var/lib/coreos-install && sudo ln -sn /var/lib/coreos-install /var/lib/flatcar-install
```

## Restart service and reboot

After that, restart the update service so it rescans the edited configuration. Initiate an immediate update.
This takes some time. Afterwards remove the `SERVER` parameter from the `update.conf` file because it is already
specified in the `/usr/share/flatcar/update.conf` file on the new partititon.
Usually 5 minutes after the update finished, the system will reboot into Flatcar Container Linux, but you can also reboot manually:

```shell
sudo systemctl restart update-engine
sudo update_engine_client -update
sudo sed -i "/SERVER=.*/d" /etc/flatcar/update.conf
sudo systemctl reboot
```

## All steps in one script

The [update-to-flatcar.sh](/update-to-flatcar.sh) script does all required steps mentioned above for you:

```shell
# To be run on the node via SSH
core@host ~ $ wget https://docs.flatcar-linux.org/update-to-flatcar.sh
core@host ~ $ less update-to-flatcar.sh # Double check the content of the script
core@host ~ $ chmod +x update-to-flatcar.sh
core@host ~ $ ./update-to-flatcar.sh
[…]
Done, please reboot now
core@host ~ $ sudo systemctl reboot
```

**Before you reboot, check that you migrated the variable names as written in [Migrating from CoreOS Container Linux](migrate-from-container-linux.md).**

## Going back to CoreOS Container Linux

You can also go the other way.

### Manual rollback

If you just updated to Flatcar (and haven't done any additional updates), CoreOS Container Linux will still be on your disk, you just need to roll back to the other partition.

To do that, just use this command composition:

```shell
sudo cgpt prioritize "$(sudo cgpt find -t flatcar-usr | grep --invert-match "$(rootdev -s /usr)")"
```

Now you can reboot and you'll be back to CoreOS Container Linux.
Remember to undo your changes in your `/etc/coreos/update.conf` after rolling back if you want to keep getting CoreOS Container Linux updates.

For more information about manual rollbacks, check [Performing a manual rollback](/os/manual-rollbacks/#performing-a-manual-rollback).

### Force an update to CoreOS Container Linux

This procedure is similar to updating from CoreOS Container Linux to Flatcar Container Linux.
You need to get CoreOS Container Linux's public key, point update_engine to CoreOS Container Linux's update server, and force an update.

Get CoreOS Container Linux's public key:

```shell
curl -L -o /tmp/key https://raw.githubusercontent.com/coreos/coreos-overlay/master/coreos-base/coreos-au-key/files/official-v2.pub.pem
```

Bind-mount it:

```shell
sudo mount --bind /tmp/key /usr/share/update_engine/update-payload-key.pub.pem
```

Create an `/etc/flatcar` directory and copy the current update configuration:

```shell
sudo mkdir -p /etc/flatcar
sudo cp /etc/coreos/update.conf /etc/flatcar/
```

Change the `SERVER` field in `/etc/flatcar/update.conf`:

```shell
SERVER=https://public.update.core-os.net/v1/update/
```

Bind-mount the release file:

```shell
cp /usr/share/flatcar/release /tmp
sudo mount --bind /tmp/release /usr/share/flatcar/release
```

Edit `FLATCAR_RELEASE_VERSION` to force an update:

```shell
FLATCAR_RELEASE_VERSION=0.0.0
```

After that, restart the update service so it rescans the edited configuration and initiates an update.
The system will reboot into CoreOS Container Linux:

```shell
sudo systemctl restart update-engine
sudo update_engine_client -update
```
