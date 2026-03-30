# Media Server

## Table of Contents

1. [Introduction](#introduction)
1. [Server Setup](#server-setup)
1. [Dockhand](#dockhand)
1. [Building the Stack](#building-the-stack)
1. [Homer](#homer)
1. [Jellyfin](#jellyfin)
1. [qBittorrent](#qbittorrent)
1. [Prowlarr](#prowlarr)
1. [Radarr](#radarr)
1. [Sonarr](#sonarr)
1. [Bazarr](#bazarr)
1. [Cleanuparr](#cleanuparr)
1. [Seerr](#seerr)
1. [Samba](#samba)

## Introduction

This media stack contains the following applications

- Dockhand - A utility that helps with management of your docker containers
- Homer - A simple homepage for managing your Media server. Bookmark it!
- Jellyfin - A free, open-source media server that organizes your movies and TV shows and streams them to all your devices
- Seer - A request management tool that lets you browse a "Netflix-style" interface to request new content be added to your library
- Gluetun - A VPN client in a container to securely route traffic from other apps through a VPN
- qBittorrent - A lightweight, open-source BitTorrent client for downloading and managing torrents
- Prowlarr - An indexer manager that connects your download apps to various torrent and Usenet sites, syncing all those "search locations" in one central place
- Radarr - A utility for locating Movies within your indexer and sending the requests to your download application
- Sonarr - Same as Radarr but for TV Shows
- Bazarr - Integrates with Sonarr and Radarr to download subtitles for Movies and Shows
- Cleanuparr - A tool for automating the cleanup of unwanted or blocked files in Sonarr, Radarr, and download clients like qBittorrent
- Samba - This allows network access to the media library and container configs using the SMB/CIFS protocol

### Additional Resources
- Setup Guide from Thomas Wilde - https://www.youtube.com/watch?v=LV3mcfqNgcQ
  - https://thomaswildetech.com/blog/2025/10/30/jellyfin---setting-up-the-entire-stack/
- Setup Guide from KL Tech - https://www.youtube.com/watch?v=QfpZcXXGpVA
- Setup Guide from TechHut - https://www.youtube.com/watch?v=twJDyoj0tDc
- Bazarr Setup Guide from AlienTech42 - https://www.youtube.com/watch?v=8vZ95HOdT-I
- Cleanuparr Setup Guide from AlienTech42 - https://www.youtube.com/watch?v=ckb9fytNkYo
- Guides for Quality Profiles - https://github.com/TRaSH-Guides/Guides

## Server Setup

Requirements

- At least 20 GBs disk space needed
- At least 8 GBs of RAM

LXC Container Requirements

- Skip this section if you are installing on a VM or bare metal
- For LXC containers, pass-through the following devices:
  - `/dev/net/tun`
  - `/dev/dri/renderD128`
- If `id 1000` returns a user, use that user instead
- You must create a user for the docker containers to run as. Do not use the built-in root account.
  - Use `sudo useradd -s /usr/bin/bash -G sudo -m -u 1000 <username>` to create one if needed
    - This creates a user with a UID and GID of "1000", creates a home folder, sets their shell to bash, and adds them to the sudo group
  - Set a password for the new user `passwd <username>`

Once you have a user established, connect with SSH to your server

Create the following folder structure on the drive where you will be keeping your media and configs

- There are commands below the overview to quickly create the folders
```
data
├── config
│   ├── bazarr
│   ├── cleanuparr
│   ├── flaresolverr
│   ├── gluetun
│   ├── homer
│   ├── jellyfin-cache
│   ├── jellyfin-config
│   ├── prowlarr
│   ├── qbittorrent
│   ├── radarr
│   ├── seerr
│   └── sonarr
├── downloads
└── media
    ├── movies
    └── shows
```
- Navigate to the root folder of your storage location
```
cd /path/to/storage
```
- Create the root of the data directory and take ownership
```
sudo mkdir data && sudo chown $(whoami):$(whoami) data
```
- Create the folders
```
mkdir -p data/config/{bazarr,cleanuparr,flaresolverr,gluetun,homer,jellyfin-cache,jellyfin-config,prowlarr,qbittorrent,radarr,seerr,sonarr}; \
mkdir -p data/downloads/; \
mkdir -p data/media/{movies,shows};
```

> [!WARNING]
> Ensure your user is the owner of the `data` directory and it's sub folders.
>
> You may need to use `sudo chown <username>:<username> -R data` to take ownership of the files.
>
> Use `ls -la` to verify ownership.
>
> If these folders are not created and owned by the correct user, many containers will fail to start.

## Dockhand
Install docker
```
curl -fsSL https://get.docker.com | sudo sh
```

Install Dockhand to simplify container management
```
sudo docker run -d \
  --name dockhand \
  --restart unless-stopped \
  -p 3000:3000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v dockhand_data:/app/data \
  fnsys/dockhand:latest
```

Visit your server IP address on port `3000` to open the web interface

### Dockhand Environments
Click `Go to Settings` to configure your local environment

- Click `+ Add environment`
- Name it `local`
- Under "Public IP" add the IP address of the server. e.g. `192.168.0.126`
- Click `+ Add`
- Lastly click `Test all`

### Updates
Keeping images up-to-date

<!-- Method may be changing after new 'pull images and restart stack' option is added -->
- On the Containers tab, click `Check for updates` to see if there are any image updates available
- If updates are available go to the Stacks tab, under Actions for your stack click the square `Stop` button and confirm
- Back on the Containers tab, click `Update all` and wait for the images to update
- Back to on the Stacks tab, under Actions click the `Play` button to start the stack and wait for it to finish
- After all the containers are back up, go to the Images tab then click `Prune unused` and confirm
  - This removes the old images to save disk space on the server
- Keeping images up-to-date is necessary for containers like Prowlarr and Flaresolverr to ensure they continue functioning properly

Updating Dockhand

- On the Containers tab, click `Check for updates` to see if there are any image updates available
- If a Dockhand update is available, navigate to Settings > About
- Click the yellow `Update available` text and then `Update Now`
- Wait for the update to finish and click `Reload`
- After the reload, go to the Images tab then click `Prune unused` and confirm

As a fallback to the built-in updater in the event it fails and Dockhand is not accessible, you can use these commands to update and remove the old image
```
sudo docker stop dockhand; \
sudo docker rm dockhand; \
sudo docker rmi fnsys/dockhand:latest; \
sudo docker pull fnsys/dockhand:latest; \
sudo docker run -d \
  --name dockhand \
  --restart unless-stopped \
  -p 3000:3000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v dockhand_data:/app/data \
  fnsys/dockhand:latest;
```

## Building the Stack
Pre-pull the docker images to speed up building the stack
```
sudo docker pull ghcr.io/jellyfin/jellyfin; \
sudo docker pull qmcgaw/gluetun; \
sudo docker pull ghcr.io/linuxserver/qbittorrent; \
sudo docker pull ghcr.io/linuxserver/prowlarr; \
sudo docker pull ghcr.io/flaresolverr/flaresolverr; \
sudo docker pull ghcr.io/linuxserver/radarr; \
sudo docker pull ghcr.io/linuxserver/sonarr; \
sudo docker pull ghcr.io/linuxserver/bazarr; \
sudo docker pull ghcr.io/cleanuparr/cleanuparr; \
sudo docker pull ghcr.io/seerr-team/seerr; \
sudo docker pull b4bz/homer; \
sudo docker pull ghcr.io/dockur/samba;
```

Go to Stacks and select `+ Create`

- Give your stack a name at the top
  - _*Stack name must be lowercase, start with a letter or number, and contain only letters, numbers, hyphens, and underscores_
  - e.g. `media-server`
- Paste the docker-compose file in the left side of the window
  - This is the contents of [mediaserver.yml](https://raw.githubusercontent.com/Rockz1152/MediaServer/refs/heads/main/mediaserver.yml)

Fill in your `.env` file

- To paste your `.env` file, you can either click `^ Load` button or change to edit mode by clicking the small document icon next to the words "Environment variables"
  - The file `.env.dist` also contains the variables that need to be filled in
  - Keep a backup of this information in the event you need to rebuild the server
```
# General
DATA_DIR=
TIMEZONE=

# Permissions
## Enter your docker User ID and Group ID, usually 1000
## Run "id <username>" to find your IDs
PUID=
PGID=

# Hardware Acceleration
## Specify the "render" Group ID for Jellyfin
## Run "getent group render | cut -d: -f3" to find your ID
RENDER_ID=

# Network Share Settings
## Share name, default is "Data"
SMB_SHARE=Data
## Username and Password
SMB_USER=
SMB_PASS=

# Gluetun VPN Client
## Requires OpenVPN
## https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers
VPN_SERVICE_PROVIDER=
OPENVPN_USER=
OPENVPN_PASSWORD=
SERVER_REGIONS=
#FIREWALL_VPN_INPUT_PORTS=  # if you have port forwarding enabled
```
- General
  - "DATA_DIR" will be the path to your drive that media will be saved to. e.g. `/mnt/usb/data`
  - Common Timezones:
    - America/New_York
    - America/Chicago
    - America/Denver
    - America/Phoenix
    - America/Los_Angeles
- Permissions
  - Use `id <username>` to find your `uid` and `gid`, usually this is `1000` but yours may differ
- Hardware Acceleration
  - This is the group ID for the "render" group required for Jellyfin to access the video card for Hardware Accelerated Encoding
  - Hardware Acceleration is not available on any Raspberry Pi
  - Use this command to the render Group ID `getent group render | cut -d: -f3`
  - Make sure to uncomment the lines for Hardware Acceleration in the docker-compose file under the Jellyfin container if you are going to use it
- Network Share Settings
  - Enter a Username and Password that will be used to access your network share
  - You can also change the name of the share, the default is "Data"
- You must fill-in your VPN login credentials into the Gluetun section or the container will continuously restart

Gluetun - VPN

- Visit https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers and select your VPN provider
- Depending on your provider, you may need to login to your VPN's web interface to retrieve Open VPN credentials and location data
  - For "Private Internet Access" you just use your current Username and Password
  - For "Windscribe" you'll need to login online and generate OpenVPN credentials

Once you're ready, click the `Create & Start` button to deploy

> [!NOTE]
> The first time you start the stack it may produce an error in Dockhand. You may have to stop and start the stack once or twice for it to finish creating directories.

- Run `sudo journalctl --unit docker -f` on the server to monitor for errors
- Startup time can vary from 1 to 3 minutes depending on hardware

## Homer
Port: `8000`

We will generate Homer's config using a bash script to fill-in the links to your Apps

- Open a shell session on your media server and navigate to `data/config/homer`
- Once you are in the Homer directory, copy and paste the code below.
- Open a browser to `http://Your.Server.IP:8000` to see your homepage

> [!NOTE]
> The Bash script below generates a file named `config.yml` in the current working directory.
> It uses the `hostname` command to find the system's IP Address in order to create the links in the config.
> Feel free to edit the file but changes must be in [YAML](https://yaml.org/) format.
>
> Remember to change to the `data/config/homer` directory before running the script

```bash
export HOST_IP=$(hostname -I | awk '{print $1}')
echo Server IP: $HOST_IP
cat > config.yml << EOF
---
title: "Media Server"
subtitle: "Homer"
documentTitle: "Media Server Dashboard"
header: false
footer: false
theme: neon
columns: 3
connectivityCheck: true

defaults:
  layout: 'list'
  colorTheme: 'dark'

colors:
  light:
    highlight-primary: "#3367d6"
    highlight-secondary: "#4285f4"
    background: "#f5f5f5"
    card-background: "#ffffff"
    text: "#363636"
    text-header: "#ffffff"
    text-title: "#303030"
    text-subtitle: "#424242"
    card-shadow: rgba(0, 0, 0, 0.1)
    link: "#3273dc"
    link-hover: "#363636"
  dark:
    highlight-primary: "#3367d6"
    highlight-secondary: "#4285f4"
    background: "#131313"
    card-background: "#2b2b2b"
    text: "#eaeaea"
    text-header: "#ffffff"
    text-title: "#fafafa"
    text-subtitle: "#f5f5f5"
    card-shadow: rgba(0, 0, 0, 0.4)
    link: "#3273dc"
    link-hover: "#ffdd57"

links:
  - name: "Media Server Dashboard"
  - name: "Github"
    icon: "fab fa-github"
    url: "https://github.com/Rockz1152/MediaServer"
    target: "_blank"

services:
  - name: "Media"
    icon: "fas fa-film"
    items:
      - name: "Jellyfin"
        logo: "https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/png/jellyfin.png"
        url: "http://$HOST_IP:8096"
        subtitle: "Media Server"
        target: "_blank"
      - name: "Seer"
        logo: "https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/png/seerr.png"
        url: "http://$HOST_IP:5055"
        subtitle: "Media Request Manager"
        target: "_blank"

  - name: "Infrastructure"
    icon: "fas fa-server"
    items:
      - name: "Dockhand"
        logo: "https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/png/dockhand.png"
        url: "http://$HOST_IP:3000"
        subtitle: "Container Management"
        target: "_blank"
      - name: "qBittorrent"
        logo: "https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/png/qbittorrent.png"
        url: "http://$HOST_IP:8080"
        subtitle: "Torrent Management"
        target: "_blank"

  - name: "Arr Apps"
    icon: "fa-regular fa-bookmark"
    items:
      - name: "Prowlarr"
        logo: "https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/png/prowlarr.png"
        url: "http://$HOST_IP:9696"
        subtitle: "Indexer"
        target: "_blank"
      - name: "Radarr"
        logo: "https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/png/radarr.png"
        url: "http://$HOST_IP:7878"
        subtitle: "Movie Manager"
        target: "_blank"
      - name: "Sonarr"
        logo: "https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/png/sonarr.png"
        url: "http://$HOST_IP:8989"
        subtitle: "TV Show Manager"
        target: "_blank"
      - name: "Bazarr"
        logo: "https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/png/bazarr.png"
        url: "http://$HOST_IP:6767"
        subtitle: "Subtitles Manager"
        target: "_blank"
      - name: "Cleanuparr"
        logo: "https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/png/cleanuparr.png"
        url: "http://$HOST_IP:11011"
        subtitle: "Download Regulator"
        target: "_blank"
EOF
```

Visit your server IP address on port `8000` to see your server dashboard

## Jellyfin
Port: `8096`

Welcome to Jellyfin!

- Give your server a name. e.g. `Jellyfin` and click `Next`
- Set an admin Username and Password and click `Next`

Setup your media libraries

- Setup Movies
  - Click `Add Media Library`
  - For "Content type" select `Movies`
  - Set a "Display name" for the library
  - Click the `+` next to "Folders"
    - Navigate to `/data/media/movies` and click `OK`
  - Set "Preferred download language:" to `English`
  - Set "Country:" to `United States`
  - Click `OK`
- Setup Shows
  - Click `Add Media Library`
  - For "Content type" select `Shows`
  - Set a "Display name" for the library
  - Click the `+` next to "Folders"
    - Navigate to `/data/media/shows` and click `OK`
  - Click `Ok` when you've selected the folder
  - Set "Preferred download language:" to `English`
  - Set "Country:" to `United States`
  - Click `OK`
- Click `Next`
  - Verify "Language" and "Country/Region" then click `Next`
- Configure Remote Access
  - If you don't plan on the server being remotely accessible uncheck `Allow remote connections to this server.` otherwise click `Next`
- Click `Finish`

Setup Hardware Acceleration

- Hardware acceleration lets a media server offload video encoding/decoding tasks to the hardware so media plays with less system load
  - Source: https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/intel/#configure-with-linux-virtualization
- Make sure you have compatible hardware
  - Hardware Acceleration is not available on Raspberry Pis
  - In the docker-compose file there are two lines you must uncomment under the Jellyfin section in order to pass through the hardware
- Make sure you have set the "render" Group ID in your environment file
  - You can run `getent group render | cut -d: -f3` on the host system to find the ID
- If you are enabling Hardware Acceleration after having already deployed the stack, you may need to "Down" or remove the containers first for the changes to apply
- In Jellyfin, navigate to Dashboard > Playback > Transcoding
- Select your Hardware Acceleration method
  - For AMD and Intel systems running Haswell (4th gen Core) and older, select `Video Acceleration API (VAAPI)`
  - For Intel Broadwell (5th gen Core) and newer, select `Intel Quicksync (QSV)`
- Select Hardware Decoding Formats
  - On the host system run the follow command `sudo docker exec -it jellyfin /usr/lib/jellyfin-ffmpeg/vainfo`
  - Under "Enable hardware decoding for", check any profiles that are returned by "vainfo: Supported profile and entrypoints"
  - For example, "VAProfileMPEG2Simple" is `MPEG2`, "VAProfileH264Main" is `H264`, "VAProfileVC1Main" is `VC1`
- Scroll to the bottom and click `Save`
- To test hardware acceleration, copy some media to your server and during playback force the bitrate to lower than original
- You can monitor your server with one of the tools below to check if the video card is active
  - Intel Systems
    - Tool: `intel_gpu_top`
    - Install: `sudo apt install intel-gpu-tools`
  - AMD Systems
    - Tool: `radeontop`
    - Install: `sudo apt install radeontop`

Setup Intro Skipper

- Intro Skipper adds a button to skip Introductions and Credits
  - Source: https://github.com/intro-skipper/intro-skipper/wiki/Installation#step-1-install-the-plugin
- Navigate to Dashboard > Plugins > Manage Repositories
- Click `+ New Repository`
  - Repository Name: `Intro Skipper`
  - Repository URL: `https://intro-skipper.org/manifest.json`
  - Click `Add`
- Navigate back to plugins and select `Available` in the top left
- Locate `Intro Skipper`
- Open the plugin page and select `Install` and `Install` again
- Navigate back to the Dashboard and select `Restart` to restart Jellyfin
- Wait a minute and refresh the page, Intro-Skipper should be working now
- By default some introductions may not be detected under the following conditions:
  - The introduction takes place after more than 25 percent of the show
  - The introduction takes place after 10 minutes
  - The introduction is shorter than 15 seconds
- The detection settings can be adjusted in the plugin interface
  - Dashboard > Intro Skipper under Plugins > Analysis
  - Adjust the value of the settings you want to change:
    - "Percent of media to analyze"
    - "Maximum runtime to analyze (in minutes)"
    - "Minimum introduction duration (in seconds)"
  - Click `Save` at the bottom if you want to change this
- Kick off an analysis of your media
  - Dashboard > Scheduled Tasks > Intro Skipper
  - Click the `Play` button next to the "Detect and Analyze Media Segments" task
  - This will take some time to complete depending on the size of your collection
- You can verify the status of media by going to the plugins settings page
  - Dashboard > Intro Skipper under Plugins > Timestamps
  - locating your media should show detected Intro and Credits segments
- Lastly, add a scheduled task to clean any removed media from Intro Skipper's cache
  - Dashboard > Scheduled Tasks > Intro Skipper > Clean Intro Skipper Cache
  - Click `+ Add Trigger`
  - Trigger type: `Weekly`
  - Day of week: `Sunday`
  - Time: `12:00 AM`
  - Click `Add`
- Let Intro Skipper finish analyzing your media while you continue with the rest of the server setup

How to create New Users

- On the left under "Server" click `Users`
- Click the `+` next to "Users"
- Give the user a simple name and password. e.g. `john`
- Under "Library Access", check `Enable access to all libraries`
- Click `Save`
- To make this user an admin, check `Allow this user to manage the server`
  - _*Only give this permission to users you trust!_

### User Specific Settings

> [!NOTE]
> These settings will need to be set on a per user basis

Disable pagination in the web view

- User Profile > Display > Scroll down to Libraries
- For "Library page size:" set the value to `0`
- Click `Save` at the bottom

Configure Default Audio

- User Profile > Playback > Audio Settings
- Set "Preferred audio language" to `English`
- Uncheck `Play default audio track regardless of language`
- Scroll to the bottom and click `Save`

Configure Subtitles

- User Profile > Subtitles
- Set "Preferred subtitle language" to `English`
- If you want Subtitles off by default set "Subtitle mode" to `None`
- Scroll to the bottom and click `Save`

## qBittorrent
Port: `8080`

After startup, open the log for qBittorrent and look for these lines. They will contain the login information.
```
The WebUI administrator username is: admin
The WebUI administrator password was not set. A temporary password is provided for this session: XXXXXXXXX
You should set your own password in program preferences.
```

- If you are having trouble locating the container logs, this command should show what you are looking for
```
sudo docker logs $(sudo docker ps -q --filter "name=qbitt")
```

After logging in, you should set your own password

- Tools > Options > WebUI
- Set a new Username and Password
  - If you don't want to use a Username and Password to use qBittorrent you can instead bypass authentication
  - Check the options `Bypass authentication for clients on localhost` and `Bypass authentication for clients in whitelisted IP subnets`
  - Enter these IP Subnets into the box `100.64.0.0/10,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16`
- Don't forget to scroll to the bottom of each page in qBittorrent's settings and click `Save` each time you make a change

Configure Downloads

- Tools > Options > Downloads
- Set Default Download Location
  - Set "Default Save Path" to `/data/downloads`
- Block Bad File Types
  - Check `Excluded file names`
  - This field will be populated later by Cleanuparr

<!-- Better list ow synced from Cleanuparr
*.exe
*.jar
*.lnk
*.zip
*.arj
*.rar
*.7z
*.tar
*.tar.gz
*.gz
-->

Configure Bandwidth

- Tools > Options > Speed > Global Rate Limits
- Limit the upload speed
  - e.g. `5000` KiB/s or 10 percent of your ISP upload rate
  - Leave Download set to `0` KiB/s for unlimited
- You can also optionally set alternative rates
  - This is intended to be a low speed mode which can switched on in the main download interface
  - Suggested values for Alternative Rate Limits: `10` KiB/s Upload and `1000` KiB/s Download

Enable Torrent Queuing

- Tools > Options > BitTorrent > Torrent Queuing
- Make sure `Torrent Queuing` is checked
- Increase `Maximum active downloads` to `6` or another preferred value
- Check `Do not count slow torrents in these limits`
  - In order to use this properly be sure to also increase `Maximum active torrents`
  - e.g set `Maximum active torrents` to `10`
- If you uncheck `Torrent Queuing`, all torrents become active at the same time

Limit Seeding

- Tools > Options > BitTorrent > Seeding Limits
- Check `When ratio reaches` and set it to `0.00`
- Make sure `then` is set to `Stop torrent`
  - _*Radarr and Sonarr will remove the torrent after it's been imported_

Check Torrents On Completion

- Tools > Options > Advanced
- Uncheck `Confirm torrent recheck`
- Check `Recheck torrents on completion`

Use https://ipleak.net/ "Torrent Address detection" to verify VPN routing

## Prowlarr
Port: `9696`

Create login information

- Set "Authentication Method" to `Forms (Login Page)`
- Set a value for Username and Password then click `Save`

Disable Telemetry

- Settings > General > Analytics
- Uncheck `Send Anonymous Usage Data`
- Click `Save Changes` at the top

Set Log Level

- Settings > General > Logging
- Set "Log Level" to `Info`
- Click `Save Changes` at the top

<!--
Configure Minimum Seeders

- Settings > Apps > Sync Profiles > Standard
- Increase "Minimum Seeders" to `3` or greater
-->

FlareSolverr

- Settings > Indexers > [+] > FlareSolverr
- Add a Tag. e.g. `flare`. This is used later to indicate if an indexer requires a Cloudflare challenge
- Leave "Host" to it's default value of `http://localhost:8191/`
- Click `Test` and then `Save`

Add Indexers

- Make sure Advanced Settings are set to show
  - Go to Settings in the left and look in the top bar for `Show Advanced`
  - Click it and it should change to "Hide Advanced"
- Indexers > Add New Indexer
- Find the indexer you want to add. Don't forget to include the `flare` tag if the site requires it.
  - This section is intentionally left vague, sorry.
  - If you are the author, check the Prowlarr-Indexer-List file
- You can use these filters to help find available public trackers
  - Protocol: `torrent`
  - Privacy: `Public`
- After configuring an indexer, click `Test` and then `Save`

Connect to qBittorrent

- _*This only allows torrents searched within Prowlarr to be sent for download, this does not automatically integrate downloads into Radarr and Sonarr_
- Settings > Download Clients
- Click [+] and select `qBittorrent`
- Enter your qBittorrent Username and Password
- Click `Test` and then `Save`
- You can now use the "Search" tab to search all indexers and download if necessary
  - After locating a torrent, check the box next to it and select `Grab Release(s)` at the bottom
  - Downloads performed through Prowlarr will remain in the downloads directory and must be manually removed from qBittorrent and moved to the media library
  - You must also manually import the media in Radarr or Sonarr

It's time to connect the Apps

## Radarr
Port: `7878`

Create login information

- Set "Authentication Method" to `Forms (Login Page)`
- Set a value for Username and Password then click `Save`

Disable Telemetry

- Settings > General > Analytics
- Uncheck `Send Anonymous Usage Data`
- Click `Save Changes` at the top

Set Log Level

- Settings > General > Logging
- Set "Log Level" to `Info`
- Click `Save Changes` at the top

Connect to Prowlarr

- Settings > General > Security
- Copy the `API Key`
- Go to Prowlarr and Navigate to Settings > Apps > Applications > [+] > Radarr
- Paste the API Key
- Click `Test` and `Save`

Setup Media Folders

- Navigate back to Radarr
- Settings > Media Management
- Click `Show Advanced` in the top bar
- Movie Naming
  - Enable `Rename Movies`
  - Colon Replacement: `Delete`
  - Standard Movie Format - Copy and paste the following:
```
{Movie.CleanTitle}{.Release.Year}{.Edition.Tags}{.MediaInfo VideoCodec}{.Quality.Full}{.Release Group}
```
- File Management
  - Enable `Unmonitor Deleted Movies`
- Root Folders
  - Click `Add Root Folder`
  - Navigate to `/data/media/movies/` and click `Ok`
- Click `Save Changes` at the top

Setup Download Client

- Settings > Download Clients > [+] > qBittorrent
- Fill in Username and Password for qBittorrent's WebUI
- Click `Test` and then `Save`

Set Quality Settings

- Navigate to Settings > Custom Formats > [+]
- Import the "Standard Size x264" format
  - Click `Import` in the bottom left and paste the following, followed by `Import` and `Save`
```
{
  "name": "Standard Size x264",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "Size",
      "implementation": "SizeSpecification",
      "negate": false,
      "required": true,
      "fields": {
        "min": 0.7,
        "max": 4
      }
    },
    {
      "name": "x264",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": true,
      "fields": {
        "value": "(x|h)\\.?264"
      }
    }
  ]
}
```
- Import the "Standard Size x265" format
  - Click `Import` in the bottom left and paste the following, followed by `Import` and `Save`
```
{
  "name": "Standard Size x265",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "Size",
      "implementation": "SizeSpecification",
      "negate": false,
      "required": true,
      "fields": {
        "min": 0.7,
        "max": 4
      }
    },
    {
      "name": "x265",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": true,
      "fields": {
        "value": "(((x|h)\\.?265)|(HEVC))"
      }
    }
  ]
}
```
- To set the score for these profiles, navigate to Settings > Profiles
- For each of the following profiles: "HD-720p/1080p, HD-720p, HD-1080p", do the following:
  - Minimum Custom Format Score: `100`
  - Set "Standard Size x264" score to `200`
  - Set "Standard Size x265" score to `100`
  - Uncheck any Quality that includes "Remux"
  - Click `Save`

## Sonarr
Port: `8989`

Create login information

- Set "Authentication Method" to `Forms (Login Page)`
- Set a value for Username and Password then click `Save`

Disable Telemetry

- Settings > General > Analytics
- Uncheck `Send Anonymous Usage Data`
- Click `Save Changes` at the top

Set Log Level

- Settings > General > Logging
- Set "Log Level" to `Info`
- Click `Save Changes` at the top

Connect to Prowlarr

- Settings > General > Security
- Copy the `API Key`
- Go to Prowlarr and Navigate to Settings > Apps > Applications > [+] > Sonarr
- Paste the API Key
- Click `Test` and `Save`

Setup Media Folders

- Navigate back to Sonarr
- Settings > Media Management
- Click `Show Advanced` in the top bar to show all option
- Episode Naming
  - Enable `Rename Episodes`
  - Colon Replacement: `Delete`
  - Series Folder Format - Set to `{Series TitleYear}`
  - Standard Episode Format - Copy and paste the following:
```
{Series.CleanTitleYear}.S{season:00}E{episode:00}.{Episode.CleanTitle}.{MediaInfo VideoCodec}.{Quality.Full}
```
- File Management
  - Enable `Unmonitor Deleted Episodes`
- Root Folders
  - Click `Add Root Folder`
  - Navigate to `/data/media/shows/` and click `Ok`
- Click `Save Changes` at the top

Setup Download Client

- Settings > Download Clients > [+] > qBittorrent
- Fill in Username and Password for qBittorrent's WebUI
- Click `Test` and then `Save`

Set Quality Settings

- Settings > Quality
- Adjust the presets to the chart below and then click `Save Changes` at the top

| Quality Contains | Min MB/m | Preferred MB/m | Max MB/m |
|------------------|----------|----------------|----------|
| SD/DVD/480p      | 4        | 15             | 20       |
| 720p             | 4        | 22             | 26       |
| 1080p            | 5        | 25             | 33       |

### Setup Custom Formats
Import Custom Format Presets

- Navigate to Settings > Custom Formats > [+]
- Follow these steps for each preset listed below
  - Click `Import` in the bottom left
  - Paste the JSON for the preset
  - Click `Import` and then click `Save`
- Preset: Season Pack x264
```
{
  "name": "Season Pack x264",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "Season Packs",
      "implementation": "ReleaseTypeSpecification",
      "negate": false,
      "required": false,
      "fields": {
        "value": 3
      }
    },
    {
      "name": "x264",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": true,
      "fields": {
        "value": "(x|h)\\.?264"
      }
    }
  ]
}
```
- Preset: Season Pack x265
```
{
  "name": "Season Pack x265",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "Season Packs",
      "implementation": "ReleaseTypeSpecification",
      "negate": false,
      "required": false,
      "fields": {
        "value": 3
      }
    },
    {
      "name": "x265",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": true,
      "fields": {
        "value": "(((x|h)\\.?265)|(HEVC))"
      }
    }
  ]
}
```
- Preset: x264
```
{
  "name": "x264",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "x264",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": true,
      "fields": {
        "value": "(x|h)\\.?264"
      }
    }
  ]
}
```
- Preset: x265
```
{
  "name": "x265",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "x265",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": true,
      "fields": {
        "value": "(((x|h)\\.?265)|(HEVC))"
      }
    }
  ]
}
```

Apply Custom Formats to Profiles

- Navigate to Settings > Profiles
- For each of the following profiles: "HD-720p/1080p, HD-720p, HD-1080p", do the following:
  - Set "Minimum Custom Format Score" to `100`
  - Set "Season Pack x264" score to `400`
  - Set "Season Pack x265" score to `300`
  - Set "x264" score to `200`
  - Set "x265" score to `100`
  - Click `Save`

## Bazarr
Port: `6767`

Create login information

- Under "Security" set "Authentication Method" to `Form`
- Set a value for Username and Password, scroll to the top and then click `Save`

Disable Telemetry

- Settings > General > Analytics
- Toggle off `Enable`
- Click `Save` at the top

Configure Languages

- Settings > Languages
- Subtitles Language
  - Under "Languages Filter" add `English`
- Click `Save` at the top
- Embedded Tracks Language
  - Set "Treat unknown language embedded subtitles track as" to `English`
- Languages Profile"
  - Click `Add New Profile`
  - Name: `English`
  - Click `Add Language`
    - Set "Subtitles Type" to `Forced (foreign part only)`
  - Click `Add Language` again
    - Set "Subtitles Type" to `Normal or hearing-impaired`
  - Cutoff: `en`
  - Click `Save`
- Scroll to the bottom and look for "Default Language Profiles For Newly Added Shows"
  - Enable `Series` and `Movies`
  - Set the profile to `English` for both
- Scroll to the top and click `Save`

Setup Providers

- Settings > Providers
- Under "Enabled Providers" click [+]
- Add the following providers, clicking `Enable` after selecting it from the list
  - `TVsubtitles`
  - `YIFY Subtitles`
  - `Gestdown (Addic7ed proxy)`
  - `Wizdom`
- (Optional but recommended) Select `OpenSubtitles.com`
  - This requires a free account at https://www.opensubtitles.com
  - Fill in your Username and Password for OpenSubtitles.com and click `Enable`
- At the top, click `Save`

Configure Subtitles

- Settings > Subtitles
- Subtitle File Options
  - Set "Hearing-impaired subtitles extension" to `.sdh`
- Upgrading Subtitles
  - Disable `Upgrade Manually Downloaded or Translated Subtitles`
- Sub-Zero Subtitle Content Modifications
  - Enable the following:
  - `Remove Tags`
  - `Remove Emoji`
  - `OCR Fixes`
  - `Common Fixes`
  - `Fix Uppercase`
- Audio Synchronization / Alignment
  - Enable `Always use Audio Track as Reference for Syncing`
  - Enable `Automatic Subtitles Audio Synchronization`
  - Enable both `Series Score Threshold For Audio Sync` and `Movies Score Threshold For Audio Sync`
  - For "Series Score Threshold For Audio Sync" set it to `90`
  - For "Movies Score Threshold For Audio Sync" set it to `90`
- Scroll to the top and click `Save`

Connect to Sonarr

- Settings > Sonarr
- Enable Sonarr
- Under host, set Address to `gluetun`
- Retrieve your Sonarr API key
  - In Sonarr go to Settings > General > Security
  - Copy the `API Key`
  - Go back to Bazarr and paste the Key under `API Key`
- Click `Test` which should return the version of the Sonarr server
- Scroll to the top and click `Save`
- Under "Options"
  - Set "Minimum Score For Episodes" to `50`
  - Enable `Download Only Monitored`
  - Enable `Exclude season zero (extras)`
- Under "Path Mappings" click `Add`
  - Sonar: `/data/media/shows/`
  - Bazarr `/data/media/shows/`
- Scroll to the top and click `Save`

Connect to Radarr

- Settings > Radarr
- Enable Radarr
- Under host, set Address to `gluetun`
- Retrieve your Radarr API key
  - In Radarr go to Settings > General > Security
  - Copy the `API Key`
  - Go back to Bazarr and paste the Key under `API Key`
- Click `Test` which should return the version of the Radarr server
- Scroll to the top and click `Save`
- Under "Options"
  - Set "Minimum Score For Movies" to `50`
  - Enable `Download Only Monitored`
- Under "Path Mappings" click `Add`
  - Radarr: `/data/media/movies/`
  - Bazarr `/data/media/movies/`
- Scroll to the top and click `Save`

Search for Missing Subtitles

- To see missing subtitles, on the left under "Wanted" click either `Episodes` or `Movies`
- At the top of each list click `Search All`
- Subtitles will be saved with the media

## Cleanuparr
Port: `11011`

Configure Prowlarr to support Cleanuparr

- Go to Prowlarr
- Navigate to Settings > Apps
- Enable advanced settings by clicking on Show Advanced
- Edit Radarr and Sonarr
- Enable `Sync Reject Blocklisted Torrent Hashes While Grabbing` and click `Save`
- Click `Test All Apps` and then `Sync App Indexers`

Configure Basic Settings

- Open Cleanuparr in a browser
- Setup a Username and Password
- For "Two-Factor Authentication" select `Skip for now`
- Click `Complete Setup`
- Log and got to Settings > General
- Toggle off `Display Support Banner`
- Click `Save Settings`

### Connect Apps
Sonarr

- On the left under "Media Apps" select `Sonarr` and then click `Add Instance`
- Fill in the connect information:
  - Name: `Sonarr`
  - URL: `http://localhost:8989`
- Retrieve your Sonarr API key
  - In Sonarr go to Settings > General > Security
  - Copy the `API Key`
  - Go back to Cleanuparr and paste the Key under `API Key`
- Click `Test` and then `Save`

Radarr

- On the left under "Media Apps" select `Radarr` and then click `Add Instance`
- Fill in the connect information:
  - Name: `Radarr`
  - URL: `http://localhost:7878`
- Retrieve your Radarr API key
  - In Radarr go to Settings > General > Security
  - Copy the `API Key`
  - Go back to Cleanuparr and paste the Key under `API Key`
- Click `Test` and then `Save`

qBittorrent

- On the left under "Media Apps" select `Download Clients` and then click `Add Client`
- Fill in the connect information:
  - Name: `qBittorrent`
  - Client Type: `qBittorrent`
  - URL: `http://localhost:8080`
  - Enter your qBittorrent Username and Password
- Click `Test` and then `Save`

### Configure Queue Cleaner
Settings > Queue Cleaner > Toggle on `Enabled`

- General
  - Set "Schedule Unit" to `Hours`
  - Set "Every" to `1`
  - Click `Save Settings`
- Failed Import
  - Max Strikes = 3
  - Set "Pattern Mode" to `Include`
    - Under "Included Patterns" paste the following `No files found are eligible`
    - Press `[Enter]` after pasting the pattern to add it to the list
  - Click `Save Settings`
- Downloading Metadata
  - Max Strikes = 10
  - Click `Save Settings`

Stalled Download Rules

- Click `+ Add Stall Rule`
- Rule: Early Stalled Downloads
  - Name = `Early Stalled Downloads`
  - Max Strikes = `24`
  - Privacy Type = `Both`
  - Min Completion % = `0`
  - Max Completion % = `85`
  - Enable `Reset Strikes on Progress`
  - Minimum Progress to Reset = `10 KB`
  - Click `Create`
  - Click `Save Settings`
- Click `+ Add Stall Rule`
- Rule: Later Stalled Downloads Rule
  - Name = `Later Stalled Downloads`
  - Max Strikes = `72`
  - Privacy Type = `Both`
  - Min Completion % = `85`
  - Max Completion % = `100`
  - Enable `Reset Strikes on Progress`
  - Minimum Progress to Reset = `10 KB`
  - Click `Create`
  - Click `Save Settings`

Slow Download Rules <!-- Remove these rules altogether? -->

- Click `+ Add Slow Rule`
- Rule: Early Slow Downloads
  - Name = `Early Slow Downloads`
  - Max Strikes = `24`
  - Min Speed = `1 KB/s`
  - Maximum Time (Hours) = `72`
  - Privacy Type = `Both`
  - Min Completion % = `0`
  - Max Completion % = `85`
  - Ignore Above Size = `25 GB`
  - Enable `Reset Strikes on Progress`
  - Click `Create`
  - Click `Save Settings`
- Click `+ Add Slow Rule`
- Rule: Later Slow Downloads
  - Name = `Later Slow Downloads`
  - Max Strikes = `72`
  - Min Speed = `1 KB/s`
  - Maximum Time (Hours) = `0`
  - Privacy Type = `Both`
  - Min Completion % = `85`
  - Min Completion % = `100`
  - Ignore above Size = `25 GB`
  - Enable `Reset Strikes on Progress`
  - Click `Create`
  - Click `Save Settings`

### Finish Configuration

Configure Malware Blocker

- Settings > Malware Blocker > Toggle on `Enabled`
- Set "Schedule Unit" to `Minutes`
- Set "Every" to `5`
- Expand `Arr Blocklists`
- Enable `Sonarr` and `Radarr` and paste the following URL in "Blocklist Path"
```
https://cleanuparr.pages.dev/static/blacklist
```
- Click `Save Settings`

Configure Blacklist Sync

- Settings > Blacklist Sync > Toggle on `Enabled`
- Paste the following for "Blacklist File Path"
```
https://cleanuparr.pages.dev/static/blacklist
```
- Click `Save Settings`

## Seerr
Port: `5055`

Welcome to Seerr

- Select `Configure Jellyfin`
- For "Jellyfin URL", just enter `jellyfin`
- Enter an email address followed by your login for Jellyfin
  - A fake email address is perfectly fine to use here
- Click `Sign in`
- Click `Sync Libraries`
  - Enable `Movies` and `Shows`
- Click `Start Scan`
- Scroll to the bottom and click `Continue`

### Configure Services

Radarr

- Click `+ Add Radarr Server`
- Check `Default Server`
- Server Name: `Radarr`
- Hostname or IP Address: Enter `gluetun`
- API Key
  - In Radarr go to Settings > General > Security
  - Copy the `API Key`
  - Go back to Seerr and paste the Key
- Scroll to the bottom and click `Test`
- Quality Profile: `HD-1080p`
- Root Folder: `/date/media/movies`
- Click `Add Server`

Sonarr

- Click `+ Add Sonarr Server`
- Check `Default Server`
- Server Name: `Sonarr`
- Hostname or IP Address: Enter `gluetun`
- API Key
  - In Sonarr go to Settings > General > Security
  - Copy the `API Key`
  - Go back to Seerr and paste the Key
- Scroll to the bottom and click `Test`
- Quality Profile: `HD-720p`
- Root Folder: `/date/media/shows`
- Check `Season Folders`
- Click `Add Server`
- Click `Finish Setup`

### Finish Configuration

Set Language & Region for Discovery Feed

- Go to Settings > General
- Set "Discover Region" to `United States`
- Set "Discover Language" to `English`
- Scroll to the bottom and click `Save Changes`

Setup User Permissions

- Go to Settings > Users
- Find "Default Permissions" and check the following items beneath it:
  - Advanced Requests
  - View Requests
  - View Recently Added
  - Request
  - Auto-approve
  - Manage Issues
- Click `Save Changes`
- These permissions will apply to new users when they are imported from Jellyfin
  - Any existing non-admin users will need their permissions manually adjusted
  - This can be done on the Users page located on the left navigation bar

## Samba
To connect to the network share, enter: `\\192.168.0.2\Data` in Windows Explorer and then enter the configured Username and Password you set in the environment file

> [!NOTE]
> Replace the example IP address above with that of your host.

For Macintosh or Linux use `smb://server/share`

<!-- Scratch notes
## Troubleshooting
episode downloads not being manaaged or monitored correctly in Sonarr
- Stop and start the stack in dockhand and then check invalid downloads in activity

Radarr & Sonarr fail to import a download
- Navigate to Wanted and select `Manual Import`
- Browse to /data/downloads and select your media
- Click `Move Automatically`

## future add-ons

Buildarr - automated setup for stack, abandoned
- https://github.com/buildarr/buildarr
- https://github.com/nantomarioni/buildarr

Byparr - Flaresolvr replacement if needed
- https://github.com/ThePhaseless/Byparr

Profilarr - High Quality Profiles but massive media sizes
- https://www.youtube.com/watch?v=TFG6A1d2C2c
- https://www.youtube.com/watch?v=u1FQNMsuzFc
- https://github.com/Dictionarry-Hub/database

Tdarr - Convert media and never worry about file sizes

RegEx for x264 and x265
(((x|h)\\.?(264|265))|(HEVC))

sudo docker stop $(sudo docker ps -a -q)
sudo docker image prune -f -a
-->
