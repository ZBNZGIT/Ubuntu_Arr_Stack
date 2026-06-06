# 📺 Arr Stack Media Server Setup (Ubuntu Server)

A hybrid media server ecosystem optimized for **Ubuntu Server**. This guide walks you through configuring permanent storage network mounts, deploying an automation, indexer, and downloading stack via Docker Compose, and performing a bare-metal native installation of Jellyfin for optimal hardware acceleration and streaming performance.

---

## 🏗️ Architecture Overview

The services interact in a continuous automated loop:
1. **Request:** User requests content via **Seerr**.
2. **Search:** **Radarr/Sonarr** tracks the media and asks **Prowlarr** to search indexers (using **FlareSolverr** to bypass blocks).
3. **Download:** **qBittorrent** grabs the torrent and saves it to storage.
4. **Process:** **Radarr/Sonarr** imports and organizes the file, while **Bazarr** grabs subtitles.
5. **Stream:** **Jellyfin** (running natively on the host) serves the media beautifully to your devices.

---

## 📦 Services Included

| Service | Category | Description | Installation Type |
| :--- | :--- | :--- | :--- |
| 🎬 **Jellyfin** | Streaming | Open-source media server to stream your TV shows and movies. | **Native Host Service** |
| 🍿 **Seerr** | Discovery | Sleek media request manager for users to discover and request content. | Docker Container |
| 📥 **qBittorrent** | Downloader | High-performance BitTorrent client with a web UI. | Docker Container |
| 🔍 **Prowlarr** | Indexer | Centralized indexer manager for torrent trackers and Usenet indexers. | Docker Container |
| 🕵️‍♂️ **FlareSolverr**| Proxy | Proxy server to bypass Cloudflare protection challenges on indexers. | Docker Container |
| 🛠️ **Radarr** | Automation | Movie collection manager and automation tool. | Docker Container |
| 📺 **Sonarr** | Automation | TV show collection manager and automation tool. | Docker Container |
| 📝 **Bazarr** | Companion | Automated subtitle downloader companion for Radarr and Sonarr. | Docker Container |
| 🐕 **Houndarr** | Utility | Syncing tool to keep instances perfectly in line. | Docker Container |

---

## 🌐 Web Interface Access Ports

Once deployed, you can access your services locally using your Ubuntu server's IP address or `localhost`:

| Service | Default Port | Quick Link |
| :--- | :---: | :--- |
| 🎬 **Jellyfin** | `8096` | http://localhost:8096 |
| 🍿 **Seerr** | `5055` | http://localhost:5055 |
| 🛠️ **Radarr** | `7878` | http://localhost:7878 |
| 📺 **Sonarr** | `8989` | http://localhost:8989 |
| 🔍 **Prowlarr** | `9696` | http://localhost:9696 |
| 📥 **qBittorrent**| `8080` | http://localhost:8080 |
| 📝 **Bazarr** | `6767` | http://localhost:6767 |
| 🕵️‍♂️ **FlareSolverr**| `8191` | http://localhost:8191 |

---

## 📂 Storage Paths & Volumes

Before deploying, ensure your host machine has mounted your TrueNAS datasets to the paths listed below. The stack relies on these exact paths to store app configurations, manage incomplete downloads, and store your media library.

### Application Configuration Data
Persistent data for settings, databases, and logs are mapped under `/config/`:
* `/config/jellyseerr` — Seerr request logs and user database
* `/config/qbittorrent` — Torrent client tracking and preferences
* `/config/prowlarr` — Indexer mappings and proxy history
* `/config/radarr` — Movie database and tracking lists
* `/config/sonarr` — TV show database and tracking lists
* `/config/bazarr` — Subtitle configuration and search profiles
* `/config/houndarr` — Sync utility config data

*(Note: Jellyfin native server data is stored globally on the host at `/var/lib/jellyfin` and `/etc/jellyfin`)*

### Media Library Data
Your shared media pools are mapped directly to the automation managers and streaming server:
* `/mnt/truenas/movies` — Main storage directory for all processed movies
* `/mnt/truenas/tvshows` — Main storage directory for all processed TV show seasons
* `/mnt/truenas/media-downloads` — Temp folder where qBittorrent stores current and completed downloads before they are imported by Radarr/Sonarr

---

## 🚀 Quick Start Guide

Follow this unified block of instructions on your Ubuntu Server instance to configure storage mounts, install the core Docker runtime environment, set up your workspace paths, pull the automation stack, and execute the native host script for Jellyfin.

```bash
# =========================================================================
# 1. PREREQUISITES & TRUENAS MOUNT CONFIGURATION (CIFS/SAMBA)
# =========================================================================
# Install the required CIFS utility package so Ubuntu can read network shares
sudo apt update && sudo apt install cifs-utils -y

# Create a secure credentials file for your TrueNAS user details
# Inside this file, add your TrueNAS Samba user details exactly like this:
# username=your_truenas_username
# password=your_truenas_password
sudo nano /etc/samba/cred

# Lock down the credentials file permissions so only root can read it
sudo chmod 600 /etc/samba/cred

# Create the local destination directories where your shares will mount
sudo mkdir -p /mnt/truenas/movies
sudo mkdir -p /mnt/truenas/tvshows
sudo mkdir -p /mnt/truenas/media-downloads

# Open your filesystem table configuration file to add permanent mounts
# Append the following three lines to the bottom of the /etc/fstab file:
# //10.0.0.84/movies              /mnt/truenas/movies               cifs credentials=/etc/samba/cred,file_mode=0777,dir_mode=0777 0 0
# //10.0.0.84/tvshows             /mnt/truenas/tvshows              cifs credentials=/etc/samba/cred,file_mode=0777,dir_mode=0777 0 0
# //10.0.0.84/media-downloads      /mnt/truenas/media-downloads      cifs credentials=/etc/samba/cred,file_mode=0777,dir_mode=0777 0 0
sudo nano /etc/fstab

# Mount all newly added network drives instantly
sudo mount -a

# =========================================================================
# 2. INSTALL DOCKER & DOCKER COMPOSE
# =========================================================================
# Download and execute the official automated Docker installation script
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Verify the container environment and runtime are properly set up
docker compose version

# =========================================================================
# 3. DIRECTORY SETUP & FILE DOWNLOAD
# =========================================================================
# Create a local workspace directory for your compose deployment and step into it
mkdir -p ~/arr-stack && cd ~/arr-stack

# Fetch the raw configuration file directly from the GitHub repository via curl
curl -sSL https://raw.githubusercontent.com/ZBNZGIT/arr-stack/main/docker-compose.yml -o docker-compose.yml

# =========================================================================
# 4. DEPLOY AND INSTANTIATE AUTOMATION STACK
# =========================================================================
# Launch all service containers sequentially in detached mode (background layer)
docker compose up -d

# Verify container lifecycles, runtime mapping statuses, and health configurations
docker compose ps

# =========================================================================
# 5. NATIVE JELLYFIN INSTALLATION (BARE-METAL HOST METHOD)
# =========================================================================
# Fetch the official deployment script directly from the Jellyfin repository
curl -o install-jellyfin.sh https://repo.jellyfin.org/install-debuntu.sh

# Elevate execution permissions and run the installer environment script
chmod +x install-jellyfin.sh
sudo ./install-jellyfin.sh
