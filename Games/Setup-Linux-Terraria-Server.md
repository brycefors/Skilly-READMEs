# Setup Linux Terraria Server

This guide outlines the steps to set up a dedicated Terraria server on Linux (Debian/Ubuntu). It uses `tmux` to run the server in the background and `systemd` to ensure it starts automatically on boot.

## 1. Install Dependencies
Update the system and install necessary utilities.

```bash
sudo apt update && sudo apt install -y wget unzip tmux
```

## 2. Download and Install
Create a directory for the server, download the latest version, and extract it.

```bash
# Create a dedicated system user and home directory
sudo useradd -r -m -d /terraria -s /bin/bash terraria
cd /terraria

# Note: The version number (e.g., 1453) changes with updates. Check terraria.org for the latest link.
sudo wget https://terraria.org/api/download/pc-dedicated-server/terraria-server-1453.zip -O terraria-server.zip

# Unzip
sudo unzip terraria-server.zip

# Navigate to the Linux binary folder
# Note: The version folder (e.g., '1453') changes with updates.
cd */Linux

# Make binary executable
sudo chmod +x TerrariaServer.bin.x86_64

# Set ownership to the dedicated user
sudo chown -R terraria:terraria /terraria
```

## 3. Configure Server
Create the server configuration file.

`sudo nano /terraria/serverconfig.txt`

Paste the following configuration (adjust paths and settings as needed):

```ini
# Path to the world file
world=~/.local/share/Terraria/Worlds/Worldy_McWorld.wld
# Auto-create world size if missing: 1=Small, 2=Medium, 3=Large
autocreate=3
# Name of the world to generate
worldname=Worldy McWorld
# Difficulty: 0=Classic, 1=Expert, 2=Master, 3=Journey
difficulty=2
# Max players allowed
maxplayers=8
# Server port (default 7777)
port=7777
# Message of the Day shown on join
motd=You guys are great!
# Universal Plug and Play (0=Off, 1=On) - Best left off for dedicated servers
upnp=0
```

## 4. Create Systemd Service
Create a service file to manage the server process.

`sudo nano /etc/systemd/system/terraria.service`

**Important:** Check the actual path of your extracted server (e.g., `/terraria/1449/Linux`) and update the `WorkingDirectory` below accordingly.

```ini
[Unit]
Description=Terraria Dedicated Server
After=network.target

[Service]
Type=forking
User=terraria
# UPDATE THIS PATH to match your extracted version folder
WorkingDirectory=/terraria/1453/Linux
ExecStart=/usr/bin/tmux new-session -s terraria-server -d ./TerrariaServer.bin.x86_64 -config /terraria/serverconfig.txt
ExecStop=/usr/bin/tmux kill-session -t terraria-server
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
```

## 5. Enable and Start
Reload systemd to recognize the new service, then enable and start it.

```bash
sudo systemctl daemon-reload
sudo systemctl enable terraria
sudo systemctl start terraria
```

Check the status to ensure it is running:
```bash
sudo systemctl status terraria
```

## 6. Accessing the Console
To interact with the server console (to run commands like `save`, `say`, or `exit`), attach to the tmux session.

```bash
# List active sessions
sudo -u terraria tmux list-sessions

# Attach to the session
sudo -u terraria tmux attach -t terraria-server
```

**To Detach:**
Press `Ctrl` + `B`, release both, then press `D`. This leaves the server running in the background.

## 7. External Access (Port Forwarding)
To allow players to connect from outside your local network, you need to configure your router to forward traffic to this server.

1.  **Find your Server's Local IP:**
    Run `hostname -I` to get your local IP address (e.g., `192.168.1.50`).

2.  **Configure Router Port Forwarding:**
    *   Log in to your router's admin panel.
    *   Navigate to the **Port Forwarding** section.
    *   Create a rule forwarding **TCP Port 7777** to your server's **Local IP**.

3.  **Get your Public IP:**
    *   Run `curl -4 ifconfig.me` to see your public IP address.
    *   Share this IP with your friends.

**Note:** If you have a firewall enabled on this server (like `ufw`), ensure you allow port 7777: `sudo ufw allow 7777/tcp`.
