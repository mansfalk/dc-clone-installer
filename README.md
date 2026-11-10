# disc-clone

Voice rooms and screen sharing for a group of friends, on a server one of
you runs. The app is for Windows; the server runs on Windows and Linux.

Everything to download is on the
[latest release](https://github.com/mansfalk/dc-clone-installer/releases/latest).

- [Installing the app](#installing-the-app)
- [Hosting a server](#hosting-a-server)

## Installing the app

For Windows 10 and 11 (64-bit).

1. Download
   [`disc-clone-win-Setup.exe`](https://github.com/mansfalk/dc-clone-installer/releases/latest/download/disc-clone-win-Setup.exe)
   and run it. It installs for you alone, without asking for
   administrator rights, and starts the app.

   Windows may say "Windows protected your PC", because the installer
   carries no signature from a certificate authority. Press **More
   info**, then **Run anyway**.

2. In the app, press **+** in the column on the left ("Add a server") and
   fill in what whoever runs the server gave you:

   - **Address** and **port** (the port is 7879 unless they say otherwise);
   - **Password**;
   - **Your name there**: what the others will see you as;
   - under Advanced settings, the server's **fingerprint**, if you were
     given one. Left empty, the app takes the server's key the first time
     it connects and holds the server to it from then on.

3. Click a room to join it. The app can be connected to several servers
   at once.

The app updates itself: when a new version is out it is fetched in the
background, and the app says so and offers to restart into it.

Your name on a server belongs to this install of the app. After
reinstalling, or on another PC, ask whoever runs the server to free the
name before you can use it again.

## Hosting a server

One program, with nothing else to install. It needs a machine the others
can reach: your own PC, a machine at home that is always on (a Raspberry
Pi will do), or a rented server.

The steps below are the short version. [HOSTING.md](HOSTING.md) has the
whole guide: every setting, running it as a service that starts with the
machine, Docker, backing up, updating and what to do when something does
not work.

### 1. Get the program

From the [latest release](https://github.com/mansfalk/dc-clone-installer/releases/latest):

| Machine | File |
| --- | --- |
| Windows | `disc-clone-server-<version>-windows-x64.exe` |
| Linux, most PCs and rented servers (`uname -m` says `x86_64`) | `disc-clone-server-<version>-linux-x64` |
| Linux on ARM, such as a Raspberry Pi 4 or 5 with a 64-bit system (`aarch64`) | `disc-clone-server-<version>-linux-arm64` |

The Linux programs run on any distribution. The ARM one has not been
tried on a real ARM machine yet.

### 2. Start it

Put it in a folder of its own; its settings are kept next to it.

- **Windows**: double-click it. If Windows asks whether to let it through
  the firewall, allow it.
- **Linux**:

  ```sh
  mkdir disc-clone-server
  cd disc-clone-server
  curl -fL -o disc-clone-server <the link to the file, copied from the release page>
  chmod +x disc-clone-server
  ./disc-clone-server
  ```

The first start writes `server.toml` with a made-up password and prints
what people need to connect:

```text
What people type in the app:
  Address:     (set public_address in server.toml to see it here)
  Port:        7879
  Password:    k7hm2qx9vtrb
  Fingerprint: 3F0A 91C2 ...
```

To try it, add `127.0.0.1` with that password in the app on the same
machine.

### 3. Let others in

Out of the box only the machine itself can connect.

1. Open `server.toml`, next to the program, and set:

   ```toml
   name = "The friends"
   open_to = "everyone"
   public_address = "203.0.113.7"
   ```

   `public_address` is the address the others will type: your public
   address (sites like whatismyip.com show it), a domain name that points
   at it, or a VPN address such as one from [Tailscale](https://tailscale.com).
   Then stop the server (Ctrl+C, or close its window) and start it again.

2. Open port **7879 for both TCP and UDP**. TCP is for the app, UDP for
   voices and video; with only TCP open people can join but hear nobody.

   - At home: pass the port on to the machine in your router ("port
     forwarding"), for TCP and UDP.
   - On a rented server: open it in the provider's firewall, and in the
     machine's own if it has one turned on:

     ```sh
     sudo ufw allow 7879/tcp
     sudo ufw allow 7879/udp
     ```

   - With Tailscale: nothing to open. Everybody installs Tailscale and
     joins the same network, and nobody else can reach the server at all.
     This is the simplest and safest way for a group of friends.

3. Give the others the address, the port, the password and, if you like,
   the fingerprint.

### 4. Keep it running (Linux)

Started in a terminal, the server stops when you log out. To have it
start with the machine and keep running, install it as a service. The
release has the file for it, `disc-clone-server.service`:

```sh
curl -fL -o disc-clone-server.service <the link to disc-clone-server.service>
sudo useradd --system --home-dir /var/lib/disc-clone-server --shell /usr/sbin/nologin disc-clone
sudo install -m 755 disc-clone-server /usr/local/bin/disc-clone-server
sudo install -m 644 disc-clone-server.service /etc/systemd/system/disc-clone-server.service
sudo systemctl daemon-reload
sudo systemctl enable --now disc-clone-server
```

The service keeps its own settings in `/var/lib/disc-clone-server`, apart
from the folder you tried the program in, so it starts with a new
password. Read it, set the server up, and start it again:

```sh
sudo journalctl -u disc-clone-server
sudoedit /var/lib/disc-clone-server/server.toml
sudo systemctl restart disc-clone-server
```

### Updating a server

Download the new program, put it in place of the old one and start it
again; the settings stay. The notes on each release say what changed for
whoever runs a server, and whether the server has to be updated before
the apps.

### Good to know

- Keep `tls-key.pem` and `media-key.toml`, next to `server.toml`, to
  yourself, and back the folder up: with a new `tls-key.pem` everybody
  has to accept the server anew and take their names again.
- The password is kept as plain text in `server.toml` and in the app's
  settings. Do not reuse one that matters elsewhere.
- The server listens on IPv4 only.
