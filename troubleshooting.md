# Troubleshooting
{: .no_toc }

Issues concerning the primary operations of your node.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

### Can I login using SSH?

Yes! Open a terminal on your computer and enter `ssh umbrel@umbrel.local`. When prompted for your password, enter your dashboard password or `moneyprintergobrrr` if you don't have one yet.

### My node keeps crashing. What can I do to fix the issue?

If you're not using the official power supply, it's probably the power supply.
To detect undervoltage, [connect to your RPi via SSH](#can-i-login-using-ssh) and run this command: `vcgencmd get_throttled`.
If it doesn't output throttled=0x0, then it's either the power supply or your SSD is using too much power (this can only be the case if you're not using the recommended hardware).

If that doesn't help, [contact us on Discord](https://discord.gg/6U3kM2cjdB).

### Mynode doesn't boot. What can I do?

Do you have connected anything to the GPIO pins?
If yes, try to unplug it and reboot the RPi by unplugging the power supply and then plugging it back in.

### I can't access the dashboard at umbrel.local or my node keeps crashing. What can I do?

Check if your router detects your node.
If it doesn't, either you ethernet cable isn't plugged in correctly or the node doesn't boot.
If you think the ethernet cable isn't the issue, follow the answer of the previous question.

If it does detect the node, try to access it with the IP address directly.
If you can't access the dashboard via the IP address either, try to run an automatic issue finding tool [over SSH](#can-i-login-using-ssh):

```
~/umbrel/scripts/debug --upload
```

It'll automatically tell you the next step.

### I want to connect to my node using ...... over my local network, but it doesn't work. How can I fix this?

If you want to connect to your node over the local network, just replace your onion domain with umbrel.local for any of the connection strings.

### Setting a fixed address on the Raspberry Pi

If your router does not support setting a static ip address for a single device, you can also do this directly on the Raspberry Pi.

This can be done by configuring the DHCP-Client (on the Pi) to advertise a static IP address to the DHCP-Server (often the router) before it automatically assigns a different one to the Raspberry Pi.

1. Get ip address of default gateway (router).
   Run `netstat -r -n` and choose the IP address from the gateway column which is not `0.0.0.0`. In my occasion it's `192.168.178.1`.

2. Configure the static IP address for the Pi, the gateway path and a DNS server.
   The configuration for the DHCP client (Pi) is located in the `/etc/dhcpcd.conf` file:

   ```
   sudo nano /etc/dhcpcd.conf
   ```

   The following snippet is an example of a sample configuration. Change the value of `static routers` and `static domain_name_servers` to the IP of your router (default gateway) from step 1. Be aware of giving the Raspberry Pi an address which is **OUTSIDE** the range of addresses which are assigned by the DHCP server. You can get this range by looking under the router configurations page and checking for the range of the DHCP addresses. This means, that if the DHCP range goes from `192.168.178.1` to `192.168.178.99` you're good to go with the IP `192.168.178.100` for your Raspberry Pi.

   Add the following to the `/etc/dhcpcd.conf` file:

   ```
   # Configuration static IP address (CHANGE THE VALUES TO FIT FOR YOUR NETWORK)
   interface eth0
   static ip_address=192.168.178.100/24
   static routers=192.168.178.1
   static domain_name_servers=192.168.178.1
   ```

3. Restart networking system
   `sudo /etc/init.d/networking restart`

### Using WiFi instead of Ethernet

- Create a file `wpa_supplicant.conf` in the boot partition of the microSD card with the following content.
  Note that the network name (ssid) and password need to be in double-quotes (like `psk="password"`)

  ```conf
  ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
  update_config=1
  country=[COUNTRY_CODE]
  network={
    ssid="[WIFI_SSID]"
    psk="[WIFI_PASSWORD]"
  }
  ```

- Replace `[COUNTRY_CODE]` with the [ISO2 code](https://www.iso.org/obp/ui/#search){:target="\_blank"} of your country (eg. `US`)
- Replace `[WIFI_SSID]` and `[WIFI_PASSWORD]` with the credentials for your own WiFi.

### Manually accessing `bitcon-cli` and `lncli`

On Citadel, these binaries are always available in CITADEL_ROOT_DIR/bin/. On Citadel OS, you can [access them over SSH](#can-i-login-using-ssh) as

```
~/umbrel/bin/bitcoin-cli
```

and

```
~/umbrel/bin/lncli
```

### Reset your user data (if you lost your password)

Do this only if you **do not have any funds** on your LND wallet! If you have funds, then save your seed + backup file so you are able to restore it later if needed.

You are going to loose your seed, settings, data and applications!

```
sudo systemctl stop umbrel-startup && sudo rm -rf ~/umbrel/lnd/!(lnd.conf) && sudo rm ~/umbrel/db/user.json && sudo rm ~/umbrel/db/umbrel-seed/seed && sudo systemctl start umbrel-startup
```

### Manually updating Citadel

To manually update your node, run these commands [over SSH](#can-i-login-using-ssh):

```
cd ~/citadel && sudo ./scripts/update/update --repo runcitadel/compose-nonfree
```

If the update was stuck, run this before the above command:

```
sudo rm statuses/update-in-progress 
```

### Recovering from a channels.backup (Citadel OS)

Once you’ve restored from the 24 words, it might take a few minutes to a few hours for it to scan all of your previous Bitcoin (on-chain) transactions and balances.
Meanwhile, here's how you can restore the funds in your Lightning channels.

#### Step 1: Copy over the channel backup file from your computer to your node.

Open the “Terminal” app on Mac/Linux, or “PowerShell” on Windows and run this:

```
scp <path/to/your/channel/backup/file> umbrel@umbrel.local:/home/umbrel/umbrel/lnd/channel.backup
```

_(Replace `<path/to/your/channel/backup/file>` with the exact path to channel backup file on your computer)_

The password is `moneyprintergobrrr`, except on version 0.3.3 or later where the password is your personal user password instead.

####  Step 2: SSH into your node

This is [explained here](#can-i-login-using-ssh).

#### Step 3: Recover funds

```
cd ~/umbrel && ./bin/lncli restorechanbackup --multi_file /data/.lnd/channel.backup
```

After you run this, wait for 1 minute. You should now be able to see your channels being closed on http://umbrel.local/lightning.

### Lightning node renaming

Please keep the following security disclaimer in mind:

> Aliases can do more harm than good by leaking your private info (h/t @lukechilds for bringing this up when we were considering setting default aliases as <first name>’s Umbrel). 
> Imagine you name your alias “Lounès’s Umbrel”. I can then go to [1ml.com](https://1ml.com), instantly find your node, see your balance, open channels, etc. 
> There isn’t much of an upside of setting a custom alias for private use as aliases aren’t unique and you can’t directly open channels by just using them as you still need the public key (and the onion address if it’s a Tor node).
> They have more value for bigger nodes (usually businesses) like Bitrefill, Bitfinex, etc so you can instantly find them and open a channel.


By this you can rename your LN node [via SSH](#can-i-login-using-ssh), so you do not have random name.

```
sudo nano ~/umbrel/lnd/lnd.conf
```

Add `alias=My amazing node` just after `[Application Options]`

  ```conf
  [Application Options]
  alias=My amazing node
  listen=0.0.0.0:9735
  rpclisten=0.0.0.0:10009
  ```

Save the file using `Ctrl + X` and `y`

Restart your node:
`sudo systemctl restart umbrel-startup`

### I sent funds to my node's wallet and wasn't 100% synced, now I can't see them. 

Step 1 - In a terminal window, execute this to SSH:

```ssh -t umbrel@umbrel.local```

Password: < your own dashboard password > 
(remember that you will not see what you type, then press ENTER)

Step 2 - execute the following command to rescan wallet's UTXOs:

```sed -i "s/\[Application Options\]/\[Application Options\]\nreset-wallet-transactions=true/g;" ~/umbrel/lnd/lnd.conf && sudo reboot```

Step 3 - Wait some time so LND can rescan all wallet's UTXOs. If it solve the issue, run this to stop it from rescanning on a future restart:

```sed -i "s/reset-wallet-transactions=true//g;" ~/umbrel/lnd/lnd.conf && sudo reboot```

Wait, it can take quite a while.

---

This troubleshooting guide will be constantly updated with findings that have been or will be reported in the issues section. Feel free to contribute via a merge request.

---
