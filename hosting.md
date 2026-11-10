# Running a disc-clone server

A disc-clone server is where people meet: the app connects to it, lists
its rooms and lets you join them. Every server has its own name, picture
and password, and the app can be connected to several servers at once.

Behind a server are two programs on one machine:

- **the disc-clone server**, which the app connects to. It checks the
  password, keeps track of who is in which room, and hands out passes to
  the rooms;
- **the media server** ([LiveKit](https://livekit.io)), which carries the
  voices and screen shares between the people in a room.

There are three ways to run them:

- **[On its own](#running-it-on-its-own)**: one program, which has the
  media server packed inside and starts it itself. Everything goes
  through one port, and the media server's key is made for you. This
  works on Windows and Linux.
- **[As a service on Linux](#on-linux-as-a-service)**: the same program,
  started when the machine starts and kept running. This is the way for
  a machine that is on all the time, such as a rented server or a
  Raspberry Pi.
- **[With Docker](#running-it-with-docker)**: the same program in a
  container, on Linux.

All three use the same settings file, [`server.toml`](#the-settings-servertoml).

## Running it on its own

1. **Get the program** from the project's releases, the same page the
   app's installer is on:

   - on Windows, `disc-clone-server-<version>-windows-x64.exe`;
   - on Linux, `disc-clone-server-<version>-linux-x64` for most PCs and
     rented servers, or `disc-clone-server-<version>-linux-arm64` for
     ARM machines, such as a Raspberry Pi 4 or 5 with a 64-bit system.
     `uname -m` tells which a machine is: `x86_64` or `aarch64`. Either
     program runs on any Linux, whatever the distribution. The ARM one
     has not been tried on a real ARM machine yet.

   To build it yourself instead, see [Building it yourself](#building-it-yourself).

2. **Put it in a folder of its own**, such as `disc-clone-server`. Its
   settings will be kept next to it. On Linux, copy the link to the file
   from the release page and download it there, under the name
   `disc-clone-server`, then allow it to run:

   ```sh
   mkdir disc-clone-server
   cd disc-clone-server
   curl -fL -o disc-clone-server <the link to the file>
   chmod +x disc-clone-server
   ```

3. **Start it** from that folder:

   ```sh
   ./disc-clone-server
   ```

   On Windows, double-click it. Windows may ask whether to let it through
   the firewall; allow it if others should connect.

4. The first start writes `server.toml` in that folder, with a made-up
   password, `media-key.toml`, the media server's key, and the
   certificate the connection is encrypted with (`tls-cert.pem` and
   `tls-key.pem`). It also unpacks the media server, once, into a folder
   of its own (`%LOCALAPPDATA%\disc-clone-server` on Windows,
   `~/.local/share/disc-clone-server` on Linux). Then the server prints
   what people need to connect:

   ```text
   disc-clone server (no name yet) is running.

     Settings:    /home/you/disc-clone-server/server.toml
     Logo:        none
     Certificate: the server's own (tls-cert.pem)
     Open to:     this machine only

   What people type in the app:
     Address:     (set public_address in server.toml to see it here)
     Port:        7879
     Password:    k7hm2qx9vtrb
     Fingerprint: 3F0A 91C2 ... (optional; the app takes it on first contact)
   ```

5. In the app, press **+** in the column on the left and enter `127.0.0.1`
   and the password. You are on your own server.

Out of the box only programs on this machine can connect to the server.
The media server's port for voices and video is on the machine's own
network address even then, because voices and video do not work over a
loopback-only port; nothing gets through it without a pass, which only
the server gives out. To let others in, see
[Letting others in](#letting-others-in).

The server stops with Ctrl+C, or by closing its window on Windows; the
media server stops with it. If the media server stops by itself, it is
started again after a few seconds. To keep a server running on Linux
after you log out, and have it start with the machine, install it
[as a service](#on-linux-as-a-service).

Options, for when the defaults do not fit:

- `--config <file>`: use another settings file; `media-key.toml` is kept
  next to it.
- `--media-server <file>`: use another LiveKit program than the packed
  one. A `livekit-server` (`.exe`) placed next to the program is used
  instead of the packed one as well.
- `--licenses`: print the license of the media server packed in.

## The settings: `server.toml`

Everything about how the server appears and who gets in is in one file,
`server.toml`. The first start writes it with an explanation next to every
setting:

```toml
name = "The friends"
logo = "logo.png"
password = "k7hm2qx9vtrb"
public_address = "203.0.113.7"
port = 7879
tcp_fallback_port = 0
open_to = "everyone"
```

| Setting | What it does | When a change applies |
| --- | --- | --- |
| `name` | The name shown in everybody's app, unless they gave the server a name of their own. At most 40 characters. | at once |
| `logo` | The picture shown in everybody's app (see below). | at once |
| `password` | What people type to be let in. Whoever is already connected stays connected. | at once |
| `public_address` | The address people use to reach this machine (see [Which address to use](#which-address-to-use)). Voices and video are sent there. | at the next start |
| `port` | The port the app connects to. Voices and video use the same number over UDP. | at the next start |
| `tcp_fallback_port` | A second port, for people whose network blocks UDP (see below). `0` for none. | at the next start |
| `open_to` | `"this-machine"` lets only programs on this machine in; `"everyone"` lets in anybody who can reach it. | at the next start |
| `tls_certificate`, `tls_key` | A certificate of your own and its key, as PEM files in the same folder (see [Encryption](#encryption)). Empty to use the one the server makes. | at the next start |

"At once" means within a couple of seconds of saving the file; apps that
are connected update by themselves. If the file has a mistake in it, the
server says so in its log and goes on with the settings it had.

Keep `media-key.toml` to yourself: whoever has it can get into rooms
without the password. If it is deleted, a new key is made at the next
start, and people simply join their room again.

### Encryption

Everything between the app and the server goes over TLS on the server's
one port: the password, the passes to the rooms, and the media server's
signalling, which the server passes through. Voices and video are
encrypted on their own.

No authority vouches for a server among friends, so the app goes by the
server's key instead, the way SSH does. The server makes a key and a
certificate on its first start and prints the key's fingerprint next to
the password. Give people the fingerprint with the invitation and the app
checks it on the first connection (it goes under "Advanced settings" in
the server form); left out, the app takes whatever key the server shows
the first time and holds the server to it from then on. Either way the
app refuses the server if its key changes, saying so with the new
fingerprint, and the person puts the new one in under Edit server, or
clears it to take the new key.

Keep `tls-key.pem` to yourself. If the key is lost and made anew,
everybody sees the refusal once and has to accept the new fingerprint,
and the names on record no longer match (see [Names](#names)).
Whoever has a certificate for a domain of their own names its PEM files
in `tls_certificate` and `tls_key`; the fingerprint is of the key, so a
certificate renewed with the same key keeps it.

### Names

A name belongs to the first app install that connects under it: the app
proves it is that install, and the server turns away anyone else who
asks for the name, so nobody can be impersonated or pushed out of a room
by somebody picking their name. The proof is made from a secret the app
made up once and keeps, and from this server's key, so what one server
is shown is of no use on any other, and no server learns the secret. An
install has one name at a time; connecting under a new one gives the
old one up, and anybody may take it. The server keeps a record in
`names.toml`, next to `server.toml`, one line per name with a
fingerprint of that proof and the day the name was first used. So that
nobody fills it by making up installs, one address may bring 20 installs
the server has not seen before in an hour; the next is told to try again
later. Installs it knows always get in.

Somebody who reinstalls the app, or loses its settings, has a new secret
and cannot use their old name until it is freed: delete its line in
`names.toml`. The change applies at the next start. The same goes for
everybody at once if the server's key is made anew (see
[Encryption](#encryption)): every proof changes with it, so delete
`names.toml` then. Apps from before names were reserved have no secret
and are turned away with a word to update.

Servers up to 0.5.0 were shown the secret itself, and could make from
it the proof for any other server, so from 0.6.3 the app makes a new
secret, once. The first time it connects to a server after that, it
sends along what the server knew it by before, and the server, once
updated, moves the name over to the new proof; update the server before
the apps, as an updated app's proof means nothing to a server that still
goes by what came before. The name's line then says what it was moved
from, `moved_from`, and on which day, `moved_on`.

Whoever ran a server up to 0.5.0 could send the same along and take a
name that has not been moved yet: nothing tells the two apart, so the
first to come gets the name. When a second app comes with the same, the
server turns it away, telling its user to ask you, and warns in its log
with the name and the fingerprint of that app. Ask the person who goes
by the name whether they were just told that their name was moved. If
so, it went to somebody else: write the fingerprint from the warning as
the `owner` on the name's line. The change applies at the next start.
If not, the second app was the one that was not who it said, and there
is nothing to do.

### The logo

Put a PNG, JPEG, GIF or WebP picture of at most 1 MB in the same folder as
`server.toml` and write its file name in `logo`. A square picture looks
best; the app shows it as a circle, and as a rounded square for the server
you are looking at. 256×256 pixels is plenty.

The app keeps a copy, so the picture still shows while the server is
down. Without a logo the app shows the first letters of the name.

### Logs people send

When something went wrong for somebody, they can send you the app's log
files from its Settings ("Send the log files to <your server>") instead
of looking for them and mailing them. The app first shows them which
files would go and what they hold, and sends them only when they say
so. The files land in
`logs-from-apps` next to `server.toml` (`deploy/data` with Docker), in
a folder named after the person, each file under the day it was sent,
as in `logs-from-apps/anna/2026-10-17-app-1234.log`. A file sent again
the same day replaces the earlier copy. Nothing is deleted for you;
delete what you have read.

Anybody who is let in can send, so there are limits: only the app's own
log files are taken, a file may be 10 MB at most, a person may have
50 MB here in all, and everybody together 500 MB; past that the app
tells them to ask you to make room.
A name that would not do as a folder name everywhere, as one with a
slash or a dot in it, is made plain and gets a fingerprint of the name
behind it.

The files hold what the app wrote down about its last runs, whichever
server they were on, yours or another: who was in the rooms the person
joined, how calls and shares went, the names of their sound devices,
and what went wrong. Apps up to 0.6.2 also wrote down the titles of
windows they shared; later ones leave them out. Treat the files as the
person's private notes.

## Letting others in

1. **In `server.toml`**: set `open_to = "everyone"` and `public_address`
   to the address people will use, then start the server again.

2. **On the router**, if the machine is at home: pass the port on to the
   machine, **for both TCP and UDP**. With the usual settings that is port
   7879; most routers let you choose "TCP and UDP" (or "Both") in a single
   rule. On a rented server (a VPS) there is usually no router, but there
   may be a firewall that needs the port opened for both; see
   [Opening the firewall](#opening-the-firewall).

   The server prints what has to be passed on when it starts.

Voices and video go over UDP, which is fastest. Some networks, at schools
and workplaces for instance, block UDP; people on those can join but hear
and see nobody. For them, set `tcp_fallback_port` to a second port, such
as 7880, and pass that one on for TCP as well.

### Which address to use

- **Your public address**, as shown by sites like whatismyip.com, if you
  pass the port on at the router. It can change from time to time unless
  your internet provider gives you a fixed one. Left empty, the server
  finds it out by itself at every start.
- **A domain name** that points at that address, which can be kept up to
  date with a dynamic DNS service.
- **A rented server's public address**, as its provider's control panel
  shows it. At some providers, Amazon, Google and Oracle among them, the
  machine itself only knows a private address (such as `10.0.0.5`) and
  the public one is passed on to it. That is fine: write the public one
  in `public_address`. The media server listens on the machine's own
  address and tells the apps to send to the public one.
- **A VPN address**, such as one from [Tailscale](https://tailscale.com).
  Everybody installs Tailscale and joins the same network; nothing needs
  to be passed on at the router, and nobody else can reach the server at
  all. This is the simplest and safest way for a group of friends. Here
  `public_address` has to be set: the server cannot find out a VPN
  address by itself.

## Several servers on one machine

Each server needs a folder of its own and a port of its own. Make a second
folder, put the program in it (or start the same program with
`--config <second folder>/server.toml`), start it once to get its
`server.toml`, and give it another `port`, such as 7889. Each server gets
its own name, password and media server key. If both use
`tcp_fallback_port`, those have to differ too.

On the router, pass on each server's port, for TCP and UDP. In the app,
add each one with the same address and its own port.

## On Linux, as a service

A service is a program the system starts by itself when the machine
starts, starts again if it stops, and keeps running when nobody is
logged in. On Linux that is done by systemd, which nearly every
distribution has (Ubuntu, Debian, Fedora, Raspberry Pi OS and the
rest). The server then runs as a user of its own, `disc-clone`, that may
touch nothing but the server's folder, `/var/lib/disc-clone-server`.

Try the program [on its own](#running-it-on-its-own) first if you like;
nothing it writes there is used by the service, which starts afresh in
its own folder.

### Installing it

1. **Download** the program (see [Running it on its own](#running-it-on-its-own)
   for which one) and the service file, `disc-clone-server.service`,
   from the same release page, into one folder:

   ```sh
   curl -fL -o disc-clone-server <the link to the program>
   curl -fL -o disc-clone-server.service <the link to disc-clone-server.service>
   ```

2. **Make the server's user**, put the program where programs go, and
   hand the service file to systemd:

   ```sh
   sudo useradd --system --home-dir /var/lib/disc-clone-server --shell /usr/sbin/nologin disc-clone
   sudo install -m 755 disc-clone-server /usr/local/bin/disc-clone-server
   sudo install -m 644 disc-clone-server.service /etc/systemd/system/disc-clone-server.service
   sudo systemctl daemon-reload
   ```

3. **Start it**, now and from now on whenever the machine starts:

   ```sh
   sudo systemctl enable --now disc-clone-server
   ```

4. **Read what it says.** What the server prints goes to the system's
   log, the journal. The first start writes `server.toml` with a made-up
   password, and prints it with the rest of what people need, as
   [above](#running-it-on-its-own):

   ```sh
   sudo journalctl -u disc-clone-server
   ```

   The password is in `server.toml` as well, whenever you need it again:

   ```sh
   sudo grep '^password' /var/lib/disc-clone-server/server.toml
   ```

5. **Set it up** in `/var/lib/disc-clone-server/server.toml` (see
   [The settings](#the-settings-servertoml) and
   [Letting others in](#letting-others-in)). The file belongs to the
   server's user, so edit it with `sudoedit`, which leaves it that way:

   ```sh
   sudoedit /var/lib/disc-clone-server/server.toml
   ```

   A logo goes into the same folder, also as the server's:

   ```sh
   sudo install -o disc-clone -g disc-clone -m 600 logo.png /var/lib/disc-clone-server/
   ```

   Name, logo and password apply at once. For the rest, start it again:

   ```sh
   sudo systemctl restart disc-clone-server
   ```

6. **Open the firewall**, if the machine has one turned on (below).

Day to day:

- **Is it running?** `systemctl status disc-clone-server`
- **What is it saying?** `sudo journalctl -u disc-clone-server -f` follows
  the log as it is written; Ctrl+C stops following, not the server.
- **Stopping and starting**: `sudo systemctl stop disc-clone-server`,
  `sudo systemctl start disc-clone-server`.
- **Log files people send** land in `/var/lib/disc-clone-server/logs-from-apps`;
  `sudo ls -R` and `sudo less` show them.

### Opening the firewall

The server's port has to be open **for both TCP and UDP**: TCP for the
app, UDP for voices and video. With the usual settings that is port
7879. If you set `tcp_fallback_port`, open that one for TCP as well.

- **Ubuntu and Debian** mostly use `ufw`. `sudo ufw status` says whether
  it is on; if it says "inactive", there is nothing to open. Otherwise:

  ```sh
  sudo ufw allow 7879/tcp
  sudo ufw allow 7879/udp
  ```

  If you turn `ufw` on yourself on a rented server, allow SSH first
  (`sudo ufw allow OpenSSH`), or it shuts you out of the machine.

- **Fedora, Rocky, AlmaLinux and other Red Hat-like systems** use
  `firewalld`:

  ```sh
  sudo firewall-cmd --permanent --add-port=7879/tcp --add-port=7879/udp
  sudo firewall-cmd --reload
  ```

- **Rented servers** often have a second firewall in the provider's
  control panel, in front of the machine: a "security group" at Amazon,
  "firewall rules" at Google, a "security list" at Oracle, "Firewalls" at
  Hetzner, and so on. Add a rule there for the port, for TCP and for UDP,
  from anywhere. Oracle's own Ubuntu images also come with a firewall of
  their own inside the machine; open it there too:

  ```sh
  sudo iptables -I INPUT -p tcp --dport 7879 -j ACCEPT
  sudo iptables -I INPUT -p udp --dport 7879 -j ACCEPT
  sudo netfilter-persistent save
  ```

### Updating

Read what changed for whoever runs a server in the
[release notes](releases.md) first; sometimes the server has to be
updated before the apps. Then download the new program, put it in place
of the old one, and start it again:

```sh
curl -fL -o disc-clone-server <the link to the new program>
sudo install -m 755 disc-clone-server /usr/local/bin/disc-clone-server
sudo systemctl restart disc-clone-server
```

Everything in `/var/lib/disc-clone-server` stays as it is. The restart
takes a few seconds. Meanwhile the app says the server cannot be reached
and tries again by itself every few seconds; people in a room lose the
sound for that moment and may have to join the room again.

A new version may bring a new media server, which is unpacked next to
the old one, into `/var/lib/disc-clone-server/disc-clone-server/livekit-<version>`.
The old one's folder can then be deleted. If the release notes say the
service file changed, install the new one as in step 2 of
[Installing it](#installing-it), `sudo systemctl daemon-reload` included,
before the restart.

### What to back up

Everything a server has is in one folder: `/var/lib/disc-clone-server`
for the service, the program's folder when it runs on its own, and
`deploy/data` with Docker. What is in it, and what losing it costs:

| File | What it is | If it is lost |
| --- | --- | --- |
| `server.toml` | The settings, the password among them. | A new one is written with a new password, which everybody has to be given. |
| `tls-key.pem`, `tls-cert.pem` | The server's key and certificate. | A new key is made. Every app refuses the server once and has to take the new fingerprint, and nobody can use their name until `names.toml` is cleared (see [Encryption](#encryption)). |
| `names.toml` | Which install each name belongs to. | Names are no longer reserved: whoever connects first under a name gets it (see [Names](#names)). |
| `media-key.toml` | The media server's key. | Little: a new key is made, and people join their room again. |
| the logo | The picture named in `server.toml`. | The app shows the first letters of the name. |
| `logs-from-apps` | Log files people sent. | Only those files. |

The `disc-clone-server` folder inside, with the unpacked media server, is
made again by itself and need not be kept.

To make a backup of the service's folder into a file only you can read,
`disc-clone-server-backup.tar.gz` in the folder you are in:

```sh
(umask 077; sudo tar -czf - -C /var/lib --exclude=disc-clone-server/disc-clone-server disc-clone-server > disc-clone-server-backup.tar.gz)
```

Keep it somewhere safe: it holds the password and the keys. To put it
back, on the same machine or a new one where the service is installed:

```sh
sudo systemctl stop disc-clone-server
sudo tar -xzf disc-clone-server-backup.tar.gz -C /var/lib
sudo systemctl start disc-clone-server
```

### Removing it

This stops the server, removes the program and the service, and, with
the fourth command, everything the server kept. Make a backup first if
there is anything you want to keep.

```sh
sudo systemctl disable --now disc-clone-server
sudo rm /etc/systemd/system/disc-clone-server.service /usr/local/bin/disc-clone-server
sudo systemctl daemon-reload
sudo rm -r /var/lib/disc-clone-server
sudo userdel disc-clone
```

Close the port in the firewall again: `sudo ufw delete allow 7879/tcp`
and `sudo ufw delete allow 7879/udp`, or
`sudo firewall-cmd --permanent --remove-port=7879/tcp --remove-port=7879/udp`
followed by `sudo firewall-cmd --reload`, and remove the rule at the
provider.

## Running it with Docker

The same program runs in a container, with the media server inside it.
This is for Linux with [Docker Engine](https://docs.docker.com/engine/install/).
The container uses the machine's network as it is, so `server.toml` works
exactly as without Docker: its port is the port, for TCP and UDP. (Docker
Desktop on Windows and macOS only allows that with "host networking"
turned on in its settings; there, running the program itself is simpler.)

1. Get the project and start the server from its `deploy` folder:

   ```sh
   git clone <this repository> disc-clone
   cd disc-clone/deploy
   docker compose up -d --build
   ```

   The first build takes a few minutes, and downloads the media server to
   pack it in; later builds take seconds.

2. The first start writes `deploy/data/server.toml`, with a made-up
   password, and `deploy/data/media-key.toml`. See what the server says:

   ```sh
   docker compose logs server
   ```

3. In the app, add `127.0.0.1` with that password.

Everything else is as described above: edit `deploy/data/server.toml` and
put the logo in `deploy/data`. Name, logo and password changes apply at
once; for the rest, start it again with `docker compose restart`.

The container runs as the user with id 1000, normally the first user on
the machine, so that the files in `deploy/data` stay yours to edit. If
`id -u` says otherwise, write your ids into a file `deploy/.env`:

```sh
printf 'UID=%s\nGID=%s\n' "$(id -u)" "$(id -g)" > .env
```

- **Logs**: `docker compose logs -f server`.
- **Stopping**: `docker compose down`.
- **Updating**: `git pull`, then `docker compose up -d --build`;
  everything in `deploy/data` stays as it is.

## Building it yourself

The server can be built from the project's source on Windows and Linux,
for instance for a machine the released programs are not made for, such
as a Raspberry Pi with a 32-bit system. It needs [Rust](https://rustup.rs).
In the project folder:

```sh
cargo build --release -p server --bin disc-clone-server
```

The program is then `target/release/disc-clone-server` (`.exe` on
Windows). The first build downloads the media server, LiveKit, to pack
it in; LiveKit's own checksum is checked. A program built this way on
Linux runs on that machine, and on others with the same or a newer
version of the distribution. The released Linux programs, which run on
any, are made with `./release-linux.sh`; how to set that up is written
at its top.

## Good to know

- **The password is stored as plain text**, in `server.toml` here (on
  Linux and macOS a new server writes it so that only you can read it)
  and in the app's settings on every machine that connects. It travels
  encrypted (see [Encryption](#encryption)), but don't reuse a password
  that matters elsewhere.
- **"UDP receive buffer is too small"** (Linux) and **"CPU monitoring
  unsupported"** (Windows): the media server says these at every start.
  Neither needs doing anything about for a server among friends. On
  Linux the first one goes away, and big rooms cope better with bursts,
  if the system allows larger buffers, as LiveKit suggests:

  ```sh
  echo 'net.core.rmem_max = 5000000' | sudo tee /etc/sysctl.d/90-disc-clone-server.conf
  sudo sysctl --system
  ```

- **The server only takes IPv4 connections.** A machine that has only an
  IPv6 address, as some of the cheapest rented servers do, cannot host
  it; most providers add an IPv4 address for a small charge.

## When something does not work

- **The app says the server cannot be reached**: check that the server
  runs, that `open_to` is `"everyone"`, and that the port is passed on at
  the router. "Answers, but not encrypted" means a server from before
  encryption; update it.
- **The app says the server's key is not the one it had before**: the
  server was set up again, or its `tls-key.pem` was replaced. Compare the
  fingerprint the server prints with what the app shows, and put it in
  under Edit server, or clear the field there to take the new key.
- **The app says the server did not let you in**: the password is wrong,
  the name is not allowed (empty, longer than 40 characters, more than
  one line, with a character in it that cannot be seen, or with spaces
  other than single, ordinary ones), the name belongs to another
  install (see [Names](#names)), or the app is too old for the server.
  The reason is shown in the app.
- **Joining a room hangs, or you hear and see nobody**: voices and video
  do not get through. Check that the port is passed on for **UDP** as
  well as TCP, and that `public_address` is the address people use. On a
  network that blocks UDP, set `tcp_fallback_port`.
- **With Docker, the server stops at once with "Permission denied"** for a
  file in `/data`: the container's user may not write to `deploy/data`.
  Make the folder yours (`sudo chown -R "$(id -u):$(id -g)" data`), and
  check the ids as described under [Running it with Docker](#running-it-with-docker).
- **The service does not start, or keeps starting again**:
  `systemctl status disc-clone-server` and
  `sudo journalctl -u disc-clone-server -n 50` say why. "Permission
  denied" for a file in `/var/lib/disc-clone-server` means it was put
  there as another user: `sudo chown -R disc-clone:disc-clone /var/lib/disc-clone-server`.
- **The server stops at once, saying `livekit-server` was not found**: the
  program was built without the media server packed in (on a system LiveKit
  has no release for). Put LiveKit's `livekit-server` next to it, or name
  it with `--media-server`.
