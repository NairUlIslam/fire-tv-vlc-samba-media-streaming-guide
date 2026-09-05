# VLC on Fire TV from a Linux Samba share

This guide covers the Linux side first so the later VLC steps have fixed values to use. Set the share name, server name, and credentials here, then use those exact values on the Fire TV.

Two naming layers matter. Avahi can advertise an mDNS `.local` name on the LAN, but Samba also exposes its own short SMB server name. In the later VLC section, enter `<SAMBA_HOST_SHORT>` as the SMB server name. Do not treat the mDNS name and the Samba/NetBIOS name as interchangeable.

## Linux server setup

Choose one short Samba/NetBIOS name and keep it unique on your LAN. A copied or generic name can make browsing unreliable or send a client to the wrong host. This guide uses `<SAMBA_HOST_SHORT>` as the placeholder for that short server name. Keep the exported share name as `SharedMovies` so the later VLC step can browse it by the same label.

### 1. Create the shared folder

With the server name and share label fixed, create the folder and give it group-writable directory permissions:

```bash
mkdir -p "$HOME/SharedMovies"
chmod 0775 "$HOME/SharedMovies"
realpath "$HOME/SharedMovies"
```

Use the `realpath` output as `<ABSOLUTE_SHARE_PATH>` in `smb.conf`. If you choose a different location, keep the share name `SharedMovies` and update the Samba `path` to match the real folder exactly.

### 2. Install Samba and, if needed, Avahi

Install Samba first. Avahi is optional and only handles mDNS advertisement on the local network.

```bash
sudo apt update
sudo apt install -y samba smbclient avahi-daemon
```

If you use Avahi, restrict it to the LAN-facing interface so Docker and other virtual interfaces do not advertise misleading addresses:

```ini
[server]
allow-interfaces=<LAN_INTERFACE>
```

That Avahi setting is for mDNS only. The later VLC step should still use `<SAMBA_HOST_SHORT>` for SMB.

### 3. Configure the Samba server and share

Edit `/etc/samba/smb.conf` and make sure the global section defines the short Samba/NetBIOS name that VLC will use later:

```ini
[global]
   workgroup = WORKGROUP
   server string = SharedMovies server
   netbios name = <SAMBA_HOST_SHORT>
   interfaces = lo <LAN_INTERFACE>
   bind interfaces only = yes
   map to guest = never

[SharedMovies]
   comment = Shared movies folder
   path = <ABSOLUTE_SHARE_PATH>
   browseable = yes
   read only = no
   guest ok = no
   valid users = <SAMBA_USERNAME>
   create mask = 0664
   directory mask = 0775
```

The interface binding keeps Samba and NetBIOS discovery on the LAN instead of advertising through VPN, Tailscale, Docker, or other virtual adapters. Those adapters can leave stale browser registrations when routes change. Keep `lo` in the list for local administration. If clients connect through a software hotspot, add its interface to the list as well.

Some installations also keep a small amount of tuning under `[global]`, such as `use sendfile = yes`, `socket options = TCP_NODELAY IPTOS_LOWDELAY`, or `deadtime = 0`. Treat those as optional. First confirm that the basic share works without them.

### 4. Add the Samba account

Set `<SAMBA_USERNAME>` to the Linux account allowed by `valid users`. If that account is your current login, add it to Samba with:

```bash
sudo smbpasswd -a "$USER"
```

Choose a Samba password separately and record it as `<SAMBA_PASSWORD>`. You will enter `<SAMBA_USERNAME>` and `<SAMBA_PASSWORD>` in VLC later.

### Optional anonymous read/write mode for a trusted LAN

Keep authenticated access as the default. Switch to anonymous read/write access only on a trusted LAN, and only if VLC cannot retain credentials and subtitle downloads need write access inside the share.

With this mode, any LAN client that can reach the share can read files, add files, modify files, and delete files. That is convenient for a private network, but it removes per-user access control for this share.

Replace `map to guest = never` in the global section and apply the share-level changes below:

```ini
[global]
   map to guest = bad user

[SharedMovies]
   guest ok = yes
   force user = <GUEST_FORCE_USER>
   force group = <GUEST_FORCE_GROUP>
```

Remove this line from the authenticated example:

```ini
valid users = <SAMBA_USERNAME>
```

`force user` and `force group` keep new files owned by a single local account and group even when the client connects anonymously. If you enable this mode, keep the LAN restricted to devices you trust. In the corresponding VLC entry, leave the username and password fields blank.

### 5. Validate the config and restart services

Check the file and restart the services now so the next Fire TV connection test uses the final server settings:

```bash
sudo testparm
sudo systemctl restart smbd nmbd
sudo systemctl restart avahi-daemon
```

If you did not enable Avahi, only the Samba restart is required. You can then test the LAN address and short server name from the Linux host:

```bash
nmblookup <SAMBA_HOST_SHORT>
smbclient -I <LAN_IPV4> -L //<SAMBA_HOST_SHORT> -N
smbclient -I <LAN_IPV4> //<SAMBA_HOST_SHORT>/SharedMovies -N -c 'ls'
```

The `-I` argument makes this self-test use the LAN address. Some Linux systems map their own short hostname to a loopback address in `/etc/hosts`, which is different from the name resolution used by another device on the LAN. For an authenticated share, omit `-N` and use `-U <SAMBA_USERNAME>` instead. Once these checks pass, the server side is ready.

## Part 2: Fire TV VLC manual SMB entry

With the server name, `SharedMovies` share, and Samba credentials already set on the Linux side, move to the Fire TV Stick and add the connection manually in VLC.

Open **VLC** and go to **Browse**. Select the **+** icon, choose **SMB**, and enter the server by hand. The SMB Server address must be the short Samba/NetBIOS name `<SAMBA_HOST_SHORT>` without `.local`. Keep the protocol on `SMB`, enter `SharedMovies` as the Folder path, and leave the Port field blank/default so VLC uses `445`.

| VLC field | Enter this value |
| --- | --- |
| Protocol | `SMB` |
| SMB Server address | `<SAMBA_HOST_SHORT>` |
| Folder path | `SharedMovies` |
| Username | `<SAMBA_USERNAME>` |
| Password | `<SAMBA_PASSWORD>` |
| Port | Leave blank/default (`445`) |

By contrast, if you selected the optional guest-writable server mode in setup because VLC would not retain credentials, use the same values below and leave **Username** and **Password** blank. This matches the high-risk trusted-LAN mode described there.

| VLC field | Enter this value |
| --- | --- |
| Protocol | `SMB` |
| SMB Server address | `<SAMBA_HOST_SHORT>` |
| Folder path | `SharedMovies` |
| Username | Leave blank |
| Password | Leave blank |
| Port | Leave blank/default (`445`) |

Whichever entry matches the server mode you chose in setup, save it and then open `SharedMovies` to confirm that the share is reachable. If your VLC build keeps saved network locations in **Favorites**, the server should appear there after you save it. If the save works but the connection, server name, credentials, or share discovery still fails, continue with the Troubleshooting section before changing playback settings.

If the share opens and playback is the only problem, adjust VLC only after the connection is already working. Under **Settings** > **Extra settings** > **Video**, try **Hardware acceleration** on **Disabled** or **Automatic** if video freezes while audio continues. Under **Settings** > **Advanced**, increase **Network caching** if playback stutters on an established stream. These playback settings are for freeze or buffering issues after connection, not for login failures, name-resolution problems, or a missing share.

## Troubleshooting and security notes

After the Fire TV client setup, match the symptom before changing anything else.

### If VLC cannot connect or login

If the share does not open or VLC rejects the login, work through these checks in order.

1. Confirm the exported share is still named `SharedMovies`, and confirm the Samba `path` points to that same folder.
2. Run `sudo testparm` to catch config mistakes, then reload or restart Samba before testing again.
3. In VLC, enter `<SAMBA_HOST_SHORT>` as the server value. Do not add `.local` here. The `.local` name belongs to mDNS, not to the SMB host value validated in VLC.
4. If VLC asks for a folder, use `SharedMovies`. If it shows available shares after login, open `SharedMovies`.
5. Leave the port blank so VLC uses the default SMB port `445`, unless you intentionally changed Samba to listen on a different port.
6. Re-enter `<SAMBA_USERNAME>` and `<SAMBA_PASSWORD>` exactly as created with `smbpasswd`.
7. If VLC keeps asking for credentials as you move between folders, first check whether the saved login for that SMB entry is being reused. If needed, remove the saved entry and add it again with the same server name and Samba credentials.
8. If VLC still cannot persist credentials and you need subtitle downloads or other write operations from the client, use the optional anonymous writable trusted-LAN configuration described in the server setup. That option exposes read, add, modify, and delete access to every client on the LAN.
9. Verify that the Fire TV device and the Samba host can actually reach each other on the LAN. The same SSID does not prove peer reachability.
10. If devices appear to be on the same Wi-Fi but still cannot talk, check AP or client isolation, guest-network rules, repeater or mesh segmentation, and any other policy that blocks local peer traffic.
11. Check UFW only if it is active. If it is disabled, skip firewall changes.
12. If you are troubleshooting name discovery, keep both Avahi and Samba scoped to the active LAN interface `<LAN_INTERFACE>`. VPN, Tailscale, Docker, and stale hotspot interfaces should not advertise the share.
13. If your router keeps changing the host address and that complicates recovery, set a DHCP reservation as a fallback for stable address assignment. Keep `<SAMBA_HOST_SHORT>` as the normal VLC server value rather than treating the reservation as a replacement for the short Samba name.

### If the share appears and disappears

First check which interface carries the LAN address:

```bash
ip -brief address
testparm -s | grep -E 'interfaces|bind interfaces only'
```

The Samba output should list `lo` and the active LAN interface, with `bind interfaces only = Yes`. If `log.nmbd` contains repeated `Network is unreachable` messages for VPN, Docker, or old network ranges, correct the interface list and restart `smbd` and `nmbd`.

A software hotspot created on the same Wi-Fi adapter can briefly reset that adapter. Starting or stopping the hotspot will interrupt active SMB sessions even when the Samba configuration is correct. Wait for the LAN interface to settle, restart Samba if discovery does not return, and reconnect from VLC.

### If VLC connects but playback freezes or stutters

Once the share opens and files are visible, connection setup is done. Playback-only fixes belong here, not in the login path.

- In VLC, try hardware acceleration on `Disabled` or `Automatic`, then raise network caching if playback still stutters.
- If the Linux host uses Wi-Fi, check `/etc/NetworkManager/conf.d/default-wifi-powersave-on.conf`. If that file manages Wi-Fi power saving, set `wifi.powersave = 2` and restart NetworkManager. Use this only for post-connection playback problems.

### Security notes

- Keep modern SMB defaults. Do not enable legacy protocols unless you have a demonstrated compatibility problem with a specific client.
- Keep guest access off and use a dedicated Samba password for `<SAMBA_USERNAME>`.
- Prefer least-privilege share permissions such as `775`, not world-writable modes.

If a check fails, return to the server setup for share, service, or credential fixes, to the VLC entry section for host and login fields, or to the playback section once browsing works but streaming does not.
