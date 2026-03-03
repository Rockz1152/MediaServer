# Media Server

## Table of Contents

1. [Introduction](#introduction)
1. [Server Setup](#server-setup)
1. [Dockhand](#dockhand)
1. [Building the Stack](#building-the-stack)
1. [Jellyfin](#jellyfin)
1. [qBittorrent](#qbittorrent)
1. [Prowlarr](#prowlarr)
1. [Radarr](#radarr)
1. [Sonarr](#sonarr)
1. [Bazarr](#bazarr)
1. [Seerr](#seerr)
1. [Samba](#samba)

## Introduction

This media stack contains the following applications

- Dockhand - A utility that helps with management of your docker containers
- Jellyfin - A free, open-source media server that organizes your movies and TV shows and streams them to all your devices
- Seer - A request management tool that lets you browse a "Netflix-style" interface to request new content be added to your library
- Gluetun - A VPN client in a container to securely route traffic from other apps through a VPN
- qBittorrent - A lightweight, open-source BitTorrent client for downloading and managing torrents
- Prowlarr - An indexer manager that connects your download apps to various torrent and Usenet sites, syncing all those "search locations" in one central place
- Radarr - A utility for locating Movies within your indexer and sending the requests to your download application
- Sonarr - Same as Radarr but for TV Shows
- Bazarr - Integrates with Sonarr and Radarr to download subtitles for Movies and Shows
- Samba - This allows network access to the media library and container configs using the SMB/CIFS protocol

### Additional Resources
- Setup Guide from Thomas Wilde - https://www.youtube.com/watch?v=LV3mcfqNgcQ
  - https://thomaswildetech.com/blog/2025/10/30/jellyfin---setting-up-the-entire-stack/
- Setup Guide from KL Tech - https://www.youtube.com/watch?v=QfpZcXXGpVA
- Setup Guide from TechHut - https://www.youtube.com/watch?v=twJDyoj0tDc
- Bazarr Setup Guide from AlienTech42 - https://www.youtube.com/watch?v=8vZ95HOdT-I
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

Create the following folder structure on the drive or network locations where you will be keeping your media and configs

- There are commands below the overview to quickly create the folders
```
data
├── config
│   ├── jellyfin-cache
│   └── jellyfin-config
├── downloads
└── media
    ├── movies
    └── shows
```

- First, navigate to the root folder of your storage location
```
cd /path/to/storage
```
- Create the folders
```
mkdir -p data/config/{jellyfin-cache,jellyfin-config}; \
mkdir -p data/downloads/; \
mkdir -p data/media/{movies,shows}
```

> [!NOTE]
> Ensure your user is the owner of the data directory and it's sub folders.
>
> You may need to use `sudo chown <username>:<username> -R data` to take ownership of the files.
>
> Use `ls -la` to verify ownership.
>
> If the folders "jellyfin-cache" and "jellyfin-config" are not owned by your user, Jellyfin will fail to start.

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

- On the Containers tab, click `Check for updates` to see if there are any image updates available
- If updates are available go to the Stacks tab, under Actions for your stack click the square `Stop` button and confirm
- Back on the Containers tab, click `Update all` and wait for the images to update
- Back to on the Stacks tab, under Actions click the `Play` button to start the stack and wait for it to finish
- After all the containers are back up, go to the Images tab then click `Prune unused` and confirm
  - This removes the old images to save disk space on the server
- Keeping images up-to-date is necessary for containers like Prowlarr and Flaresolverr to ensure they continue functioning properly

Updating Dockhand

- These commands will update Dockhand and remove the old image
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

<!-- sudo docker image prune -f -a; -->
<!-- Dockhand self update work-in-progress, seems to stall out currently
- On the Containers tab, click `Check for updates` to see if there are any image updates available
- If a Dockhand update is available, navigate to Settings > About
- Click the yellow `Update available` text and then `Update Now`
- Wait for the update to finish
-->
<!-- Workaround to broken Dockhand v1.0.19+
sudo docker stop dockhand; \
sudo docker rm dockhand; \
sudo docker rmi fnsys/dockhand:latest; \
sudo docker pull fnsys/dockhand:v1.0.18; \
sudo docker run -d \
  --name dockhand \
  --restart unless-stopped \
  -p 3000:3000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v dockhand_data:/app/data \
  fnsys/dockhand:v1.0.18;
-->

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
sudo docker pull ghcr.io/seerr-team/seerr; \
sudo docker pull ghcr.io/dockur/samba;
```

Go to Stacks and select `+ Create`

- Give your stack a name at the top
  - _*Stack name must be lowercase, start with a letter or number, and contain only letters, numbers, hyphens, and underscores_
  - e.g. `media-server`
- Paste your compose file in the left side of the window
  - This is the contents of `mediaserver.yml`

Gluetun - VPN

- Visit https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers and select your VPN provider
- Depending on your provider, you may need to login to your VPN's web interface to retrieve Open VPN credentials and location data
  - For "Private Internet Access" you just use your current Username and Password
  - For "Windscribe" you'll need to login online and generate OpenVPN credentials
- Save this info in order to prepare your environment file in Dockhand next

Fill in your `.env` file

- To paste your `.env` file, you can either click `^ Load` button or change to edit mode by clicking the small document icon next to the words "Environment variables"
  - The file `.env.dist` contains a template of all the variables that need to be filled in
- "DATA_DIR" will be the path to your drive that media will be saved to. e.g. `/mnt/usb/data`
- Common Timezones:
  - America/New_York
  - America/Chicago
  - America/Denver
  - America/Phoenix
  - America/Los_Angeles
- Use `id <username>` to find your `uid` and `gid`, usually this is `1000` but yours may differ
- Under "Network Share Settings", you can change the name of the network share and specy the Username and Password to access it
- You must fill-in your VPN info into the Gluetun section or the container will restart over and over

Once you're ready, click the `Create & Start` button to deploy

> [!NOTE]
> The first time you start the stack it may produce an error in Dockhand. You may have to stop and start the stack once or twice for it to finish creating directories before it will start and stop without error.

- Run `sudo journalctl --unit docker -f` on the server to monitor for errors
- Startup time can vary from 1 to 3 minutes depending on hardware

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

Set Default Audio

- User Profile > Playback > Audio Settings
- Set "Preferred audio language" to `English`
- Uncheck `Play default audio track regardless of language`
- Scroll to the bottom and click `Save`

Disable pagination in the web view

- User Profile > Display
- Scroll down to "Library page size:" and set the value to `0`
- Click `Save` at the bottom

## qBittorrent
Port: `8080`

After startup, open the log for qBittorrent and look for these lines. They will contain the login information.
```
The WebUI administrator username is: admin
The WebUI administrator password was not set. A temporary password is provided for this session: XXXXXXXXX
You should set your own password in program preferences.
```

After logging in, you should set your own password

- Tools > Options > WebUI
- Set a new Username and Password
- Don't forget to scroll to the bottom of each page in qBittorrent's settings and click `Save` every time you make a change

Configure Downloads

- Tools > Options > Downloads
- Set Default Download Location
  - Set "Default Save Path" to `/data/downloads`
- Block Bad File Types
  - Check `Excluded file names` and paste the following in the box underneath
```
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
```

Configure Bandwidth

- Tools > Options > Speed > Global Rate Limits
- Limit the upload speed
  - e.g. `5000` KiB/s or 10 percent of your ISP upload rate
- You can also optionally set alternative rates
  - This is intended to be a low speed mode which can switched on in the main download interface

Enable Torrent Queuing

- Tools > Options > BitTorrent > Torrent Queuing
- Make sure `Torrent Queuing` is checked
- Increase `Maximum active downloads` to `5` or another preferred value
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

<!--
Configure Minimum Seeders

- Settings > Apps > Sync Profiles > Standard
- Increase "Minimum Seeders" to `3` or greater
-->

FlareSolverr

- Settings > Indexers > [+] > FlareSolverr
- Add a Tag. e.g. `flare` This is used later to indicate if an indexer requires a Cloudflare challenge
- Leave "Host" to it's default value of `http://localhost:8191/`
- Click `Save`

Add Indexers

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

Connect to Prowlarr

- Settings > General > Security
- Copy the `API Key`
- Go back to Prowlarr and Navigate to Settings > Apps > Applications > [+] > Radarr
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
- Importing
  - Enable `Import Extra Files`
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

Connect to Prowlarr

- Settings > General > Security
- Copy the `API Key`
- Go back to Prowlarr and Navigate to Settings > Apps > Applications > [+] > Sonarr
- Paste the API Key
- Click `Test` and `Save`

Setup Media Folders

- Navigate back to Sonarr
- Settings > Media Management
- Click `Show Advanced` in the top bar to show all option
- Episode Naming
  - Enable `Rename Episodes`
  - Colon Replacement: `Delete`
  - Standard Episode Format - Copy and paste the following:
```
{Series.CleanTitleYear}.S{season:00}E{episode:00}.{Episode.CleanTitle}.{MediaInfo VideoCodec}.{Quality.Full}
```
  - Series Folder Format
    - Set to `{Series TitleYear}`
- Importing
  - Enable `Import Extra Files`
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
<!--
RegEx for x264 and x265
(((x|h)\\.?(264|265))|(HEVC))
-->

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
  - Enable `Automatic Subtitles Audio Synchronization`
  - Enable both `Series Score Threshold For Audio Sync` and `Movies Score Threshold For Audio Sync`
  - For "Series Score Threshold For Audio Sync" set it to `90`
  - For "Movies Score Threshold For Audio Sync" set it to `85`
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
- Under "Options" enable `Download Only Monitored`
- Under "Path Mappings" click `Add`
  - Radarr: `/data/media/movies/`
  - Bazarr `/data/media/movies/`
- Scroll to the top and click `Save`

Search for Missing Subtitles

- To see missing subtitles, on the left under "Wanted" click either `Episodes` or `Movies`
- At the top of each list click `Search All`
- Subtitles will be saved with the media

## Seerr
Port: `5055`

Welcome to Seerr

- Select `Configure Jellyfin`
- For "Jellyfin URL", just enter `jellyfin`
- Enter an email address followed by your login for Jellyfin
- Click `Sign in`
- Click `Sync Libraries`
  - Enable `Movies` and `Shows`
- Click `Start Scan`
- Scroll to the bottom and click `Continue`

Set Language & Region for Discovery Feed

- Go to Settings
- Discover Region
  - Click the dropdown and select `United States`
- Discover Language
  - Click the dropdown and select `English`
- Scroll to the bottom and click `Save Changes`

### Configure Services

Radarr

- Click `+ Add Radarr Server`
- Check `Default Server`
- Server Name: `Radarr`
- Hostname or IP Address: Enter your docker server IP Address
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
- Hostname or IP Address: Enter your docker server IP Address
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

## Samba
To connect to the network share, enter: `\\192.168.0.2\Data` in Windows Explorer and then enter the configured Username and Password you set in the environment file

> [!NOTE]
> Replace the example IP address above with that of your host.

For Macintosh or Linux use `smb://server/share`

<!-- Scratch notes
## Troubleshooting
episode downloads not being manaaged or monitored correctly in Sonarr
- Stop and start the stack in dockhand and then check invalid downloads in activity

## future add-ons
Profilarr - High Quality Profiles but massive media sizes
- https://www.youtube.com/watch?v=u1FQNMsuzFc
- https://www.youtube.com/watch?v=TFG6A1d2C2c
- https://github.com/Dictionarry-Hub/database
Homer - A dashboard to easily get to all services?
Tdarr - Convert media and never worry about file sizes

sudo docker stop $(sudo docker ps -a -q)
-->
