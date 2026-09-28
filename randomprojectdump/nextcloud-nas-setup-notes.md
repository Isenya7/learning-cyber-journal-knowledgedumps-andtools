# Nextcloud + NAS setup notes

## Part 1: installing nextcloud

Ubuntu server 24.04 install, on the "featured server snaps" screen selected nextcloud, let it install during setup.

After reboot, logged in via ssh.

```
snap list
```
confirmed nextcloud installed alongside snapd and core20

```
snap info nextcloud
```
checked version/channel, got 34.0.4snap3, on latest/stable

latest/stable = newest version actually deemed stable enough (not always the literal newest release). other channels like 26/stable, 27/stable pin to old versions, candidate/beta/edge are pre release, not what you want for a real setup.

## Part 2: first time setup

Opened `http://server-ip` in browser, got the admin account creation screen.

Storage & database screen showed up too, left everything default (data folder path, database user/pass/name/host), these are the snap's own bundled mariadb config, already wired up correctly, no reason to touch it.

Recommended apps screen threw an auth error (password kept getting rejected). known bug in nextcloud 32+ era snap installs. fix: just skip/close it, none of those apps are required, can install any later from the actual apps page where the bug doesn't happen.

Landed on the dashboard after that, fully working with sample demo files.

## Part 3: nextcloud office (the annoying part)

Wanted spreadsheet/doc editing in browser. went to apps, searched office.

First attempt: enabled "nextcloud office" + "nextcloud office (collabora)". this needs a separate document editing server running somewhere else, it's not built in. tried entering a documentserver address, kept failing.

Second attempt: tried "community document server" instead (self contained, no separate server needed in theory). auto filled its own url, still got "error when trying to connect (bad healthcheck status)".

debugged with:
```
curl -v http://localhost/apps/documentserver_community/healthcheck
```
got a straight 404, meaning that app's routes weren't even registered properly.

turns out this is a known incompatibility. nextcloud snap runs everything sandboxed (snap confinement), and community document server needs to spin up its own background processes to do file conversion. the sandbox blocks that. not a config mistake, just doesn't work on the snap. dead end, moved on.

## Part 4: docker + collabora (the fix that worked)

Installed docker:
```
sudo apt update
sudo apt install -y docker.io docker-compose-v2
```

Added self to docker group so no sudo needed every time:
```
sudo usermod -aG docker $USER
```
(had to log out and back in for this to actually apply)

Ran collabora in a container:
```
docker run -t -d -p 9980:9980 \
  -e "domain=192.168.xx.xxx" \
  -e "username=admin" \
  -e "password=whatever" \
  -e "extra_params=--o:ssl.enable=false --o:ssl.termination=false" \
  --restart always \
  --cap-add MKNOD \
  --name collabora \
  collabora/code
```
ssl disabled here since nextcloud itself is running plain http right now, keeps both sides consistent.

Disabled the broken community document server + onlyoffice app, enabled "nextcloud office (collabora)" instead.

Went to settings > nextcloud office, entered:
```
http://192.168.xx.xxx:9980/
```

got a green "collabora online server is reachable" message. tested by making a new spreadsheet in files app, actually opened and worked.

## Part 5: remote access without exposing anything

Context: wanted to reach nextcloud from anywhere, not just home wifi, without port forwarding and getting scanned by randoms all day.

Tailscale = auto wireguard vpn, no open ports needed. got advertised this by some very good word predictor.

Install command:
```
curl -fsSL https://tailscale.com/install.sh | sh
```
adds tailscale apt repo, installs the client

At the end:
```
Installation complete! Log in to start using Tailscale by running:
sudo tailscale up
```

Given `https://login.tailscale.com/a/stuff` to authenticate. Chose gmail to authenticate, got Success.

```
tailscale ip -4
100.xx.xxx.xx
```
Ran that command to check tailscale private address

Now:
```
sudo nextcloud.occ config:system:set trusted_domains 2 --value=100.xx.xxx.xx
```

- `nextcloud.occ` is Nextcloud's cli tool (occ)
- `config:system:set` tells occ i want to change a system wide setting, Nextcloud stores config as key-value pairs (and sometimes arrays) in a file called config.php; this command edits that file safely instead of hand-editing it
- `trusted_domains` is the changed setting
- `2` is position on list, `--value=100.xx.xxx.xx` is the actual address being added at that slot
- `sudo` = run as admin, needed since this edits nextcloud's core config file
- tl;dr: told nextcloud to trust the tailscale ip as a 3rd allowed address (0 = localhost, 1 = home ip)

```
System config value trusted_domains => 2 set to string 100
```

Installed tailscale app on phone, logged into same account. showed up as a device alongside lahomelab.

Turned off wifi, kept cellular on, went to `http://100.xx.xxx.xx` on phone.

it worked lol

## Part 6: syncing existing files across devices

Context: had CTF writeup folders already organized locally on linux (cachy), wanted them living on the server as the real home base, synced both ways to linux and windows.

Local structure being synced:
```
~/Categories/
  Forensics/
  Pwn/
  Cryptography/
  Reverse-Engineering/
  Web/
  General-Skills/
```

Installed nextcloud client on cachy:
```
sudo pacman -S nextcloud-client
```

Tried pointing it at server via tailscale ip immediately, got a timeout. turns out tailscale has to actually be running on the client machine too, not just the server, since tailscale ips only resolve between devices on the same tailnet.

Installed tailscale on cachy same way as the server:
```
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```
logged in with same gmail, joined the tailnet.

went back to nextcloud client, server address now worked.

initial setup wizard forces the first sync folder to be `/home/user/Nextcloud`, no way to manually type a different path at that stage. worked around it by letting that finish, then using the client's separate "add folder sync connection" option to point at `~/Categories` specifically.

on the remote side, briefly ended up with Categories nested under `Nextcloud/` by accident, moved it back to root in the web ui, client picked up the change automatically without touching local files.

on windows: browser was hitting the home wifi ip (`192.168.xx.xxx`) directly, timed out since windows machine wasn't on the same home network at that moment. installed tailscale for windows, logged into same gmail account, then accessed nextcloud via the tailscale ip instead. worked immediately once on the tailnet.

End result: cachy and windows both sync back and forth through the server, my stuff is well in the "cloud" ig (technically a self-hosted personal cloud, not the AWS/Google kind, since it's just one drive with no redundancy yet).

## Part 7: quick terminal access to nextcloud's actual files

Context: snap keeps nextcloud's real files buried in `/var/snap/nextcloud/common/nextcloud/data/USERNAME/files/`, annoying to type every time.

fix, made a symlink (shortcut) straight to it:
```
ln -s /var/snap/nextcloud/common/nextcloud/data/USERNAME/files ~/nextcloud-data
```

`ln -s` = make a symbolic link, basically a shortcut. doesn't copy anything, just points to the real folder.

now can just do:
```
ls ~/nextcloud-data
cd ~/nextcloud-data
```
acts exactly like the real folder.

side quest that went nowhere: tried adding a minimal gui (xorg + openbox) to the server just to open a browser directly on it. works in theory (`startx` launches it, has to be done at the physical console not ssh) but decided against it, ctrl+c'd out mid install, ran `apt --fix-broken install` and `apt autoremove` to clean up. not worth it, nextcloud's own web ui from another device does the same job.

## Architecture note: what's actually containerized

- **Nextcloud** — runs as a snap, not docker
- **Collabora (office editor)** — runs in a docker container
- **Tailscale** — installed directly on the OS, not containerized

Docker is available for adding more services later same pattern as the collabora run command, just watch for port collisions (nextcloud has 80/443, collabora has 9980).

## Still todo later

- Move nextcloud data off the laptop's main drive onto real dedicated storage
- Set up actual backup/redundancy (single drive right now = no redundancy, one disk failure = data gone)
- Lock down the WOPI allow-list warning in collabora settings
