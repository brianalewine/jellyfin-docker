# Jellyfin Server Setup with Docker, Caddy, and Tailscale

This guide will walk you through the process of setting up a Jellyfin server using Docker, Caddy, and Tailscale. It will allow you to access your server from anywhere in the world, while keeping your media private and secure.

This setup was made possible with the help of the following resources:
- [Jellyfin Remote Access](https://www.ethanmad.com/post/jellyfin_remote_access/)
- [Docker, Tailscale, and Caddy with HTTPS: A Love Story](https://www.reddit.com/r/Tailscale/comments/104y6nq/docker_tailscale_and_caddy_with_https_a_love_story/)

# Table of Contents
1. [Jellyfin Server Setup with Docker, Caddy, and Tailscale](#jellyfin-server-setup-with-docker-caddy-and-tailscale)
2. [Directory Structure](#directory-structure)
3. [Understanding the Folders](#understanding-the-folders)
4. [Setup Instructions](#setup-instructions)
5. [Setup for `jellyfin-local`](#setup-for-jellyfin-local)
6. [Setup for `jellyfin-tailscale`](#setup-for-jellyfin-tailscale)
7. [Running the Server](#running-the-server)
   - [Jellyfin-Local](#jellyfin-local)
   - [Jellyfin-Tailscale](#jellyfin-tailscale)
8. [Troubleshooting](#troubleshooting)

## Directory Structure

The layout of the current folder is as follows:

```bash
.
├── README.md
├── jellyfin-local
│   └── docker-compose.yaml
├── jellyfin-server
│   ├── cache
│   └── config
└── jellyfin-tailscale
    ├── caddy
    │   ├── Caddyfile
    │   ├── config
    │   └── data
    ├── docker-compose.yaml
    └── tailscale
        └── varlib
```

To verify that your folder is structured correctly, use the `tree` command:
```bash
$ tree
```
## Understanding the Folders

The Jellyfin setup consists of three main folders:

1. **`jellyfin-server`**: This folder contains all the information your Jellyfin server needs to run, excluding the media itself. Any changes you make to the server will be saved here.

2. **`jellyfin-local`**: This folder contains the Docker Compose file that will run your server. It's configured to restrict server access to your machine or local network, depending on your settings. This setup essentially transforms your server machine into a local media player, making it an ideal choice for situations where Tailscale connectivity is unavailable or when you want a portable laptop server.

3. **`jellyfin-tailscale`**: This folder contains the Docker Compose file that will run your server and only allow connections from devices on your Tailnet. It uses a reverse proxy through Caddy, enabling you to connect to your server over HTTPS. This folder also contains the directories that the Caddy and Tailscale containers will use to store their data while running on Docker.

> **Important:** Both `jellyfin-tailscale` and `jellyfin-local` interact with the `jellyfin-server` when running in Docker. To avoid conflicts and potential data loss, these two Docker containers should not run simultaneously. If you want to switch between the two setups, stop the running container before starting the other one.

## Setup Instructions

Follow these steps to set up your environment:

1. **Docker Setup**
   - If you haven't done so already, install [Docker Desktop](https://www.docker.com/products/docker-desktop/) on your system. 
   - If you are on Linux, you can follow the [instructions here](https://docs.docker.com/desktop/install/linux-install/).

2. **Tailscale Setup**
   - [Register a Tailscale account](https://login.tailscale.com/start). A free account can support up to 100 devices and 3 other users. More details can be found in the [Tailscale pricing blog](https://tailscale.com/blog/pricing-v3).
   - [Download and install Tailscale](https://tailscale.com/download) on your server and clients.
   - Enable [MagicDNS](https://tailscale.com/kb/1081/magicdns) and [HTTPS](https://tailscale.com/kb/1153/enabling-https) in the [DNS page](https://login.tailscale.com/admin/dns) of the Tailscale admin console.
   - Once you have authenticated your server with Tailscale, you can [share access with friends](https://tailscale.com/kb/1084/sharing).

## Setup for `jellyfin-local`

This section will guide you through the process of setting up `jellyfin-local` by modifying the `docker-compose.yaml` file located in the `jellyfin-local` directory.

### Steps

1. **Open the Docker Compose File**  
   Navigate to the `jellyfin-local` directory and open the `docker-compose.yaml` file.

2. **Navigate to Jellyfin Volumes**  
   Find the section `services: jellyfin: volumes:`.

3. **Understand the File Paths**  
   This section contains file paths, split by a colon (`:`). The part before the colon is the file's location on your machine, and the part after is its location in the Docker container. The `:ro` means that the files are read only, preventing the Docker container from modifying your media files.

4. **Replace Local File Paths**  
   Replace the local file paths (the parts before the colon) with your own file paths. 

   ```yaml
   - ~/Jellyfin/jellyfin-server/config:/config
   - ~/Jellyfin/jellyfin-server/cache:/cache
   - ~/Documents/Jellyfin/Movies:/Movies:ro
   - ~/Documents/Jellyfin/Shows:/Shows:ro
   ```

## Setup for `jellyfin-tailscale`

This guide will walk you through the process of setting up `jellyfin-tailscale` by modifying the `docker-compose.yaml` file located in the `jellyfin-tailscale` directory.

### Steps

1. **Open the Docker Compose File**  
   Navigate to the `jellyfin-tailscale` directory and open the `docker-compose.yaml` file.

2. **Configure Jellyfin Volumes**  
   Under `services: jellyfin: volumes:`, replace the local file paths (before `:`) with your own. The `:ro` makes them read-only inside the Docker container.
   ```yaml
   - ~/Jellyfin/jellyfin-server/config:/config
   - ~/Jellyfin/jellyfin-server/cache:/cache
   - ~/Documents/Jellyfin/Movies:/Movies:ro
   - ~/Documents/Jellyfin/Shows:/Shows:ro
   ```

3. **Configure Caddy Volumes**  
   Under `services: caddy: volumes:`. Replace the local file paths with your own.
   ```yaml
   - ~/Jellyfin/jellyfin-tailscale/caddy/Caddyfile:/etc/caddy/Caddyfile
   - ~/Jellyfin/jellyfin-tailscale/caddy/data:/data
   - ~/Jellyfin/jellyfin-tailscale/caddy/config:/config
   ```

4. **Configure Tailscale Volumes**  
   Under `services: tailscale: volumes:`, replace the local file paths with your own.
   ```yaml
   - ~/Jellyfin/jellyfin-tailscale/tailscale/varlib:/var/lib
   ```

5. **Set the hostname**  
   Under `services: tailscale: hostname:`, set the hostname. This will appear as the machine name in the [Tailscale admin console](https://login.tailscale.com/admin/machines). Recommended name: `jellyfin`.

6. **Set the Tailscale authentication key**  
   Under `services: tailscale: environment:`, replace the `TS_AUTHKEY` with your key from the [Tailscale admin console](https://login.tailscale.com/admin/settings/keys). Make sure the key is set to `Reusable`. More information can be found [here](https://tailscale.com/kb/1085/auth-keys).

7. **Setup Caddyfile**:
   - Navigate to the `jellyfin-tailscale/caddy` directory and open the `Caddyfile`.
   - Replace `<machine-name>` with the hostname you set in the `docker-compose.yaml` file (e.g., `jellyfin`).
   - Replace `<tailnet-name>` with your tailnet name, found in the [Tailscale admin console](https://login.tailscale.com/admin/dns) under the `DNS` tab.

## Running the Server

### Jellyfin-Local

1. Navigate to the `jellyfin-local` directory in your terminal.
2. Execute the command `docker-compose up -d`.
3. If everything is working, you should be able to access your server at `http://localhost:8096`.
4. Stop the server in Docker Desktop before running `jellyfin-tailscale`.

### Jellyfin-Tailscale

1. Navigate to the `jellyfin-tailscale` directory in your terminal and execute the command `docker-compose up -d`.
2. Check that the jellyfin server is connected to your tailnet by going to the [Tailscale admin console](https://login.tailscale.com/admin/machines).
3. In the `jellyfin-tailscale` directory in your terminal, execute the command `docker exec tailscaled tailscale --socket /tmp/tailscaled.sock cert <machine-name>.<tailnet-name>.ts.net`, replacing `<machine-name>` and `<tailnet-name>` with your own values. This [generates a certificate](https://tailscale.com/kb/1010/machine-certs) for your server.
4. With the Docker container still running, comment out or remove the line with your authentication key in the `docker-compose.yaml` file.
5. Remake the `jellyfin-tailscale` Docker container by executing the command `docker-compose up -d`.
6. If everything is working, you should be able to access your server at `https://<machine-name>.<tailnet-name>.ts.net`.

## Troubleshooting

- **Check File Paths**: Ensure all the file paths in the `docker-compose.yaml` files are correct.
- **Check Docker Logs**: If you're having trouble connecting to your server, check the logs of the Docker containers in Docker Desktop.
- **Verify Tailscale Authentication Key**: Make sure the Tailscale authentication key is correct and set to `Reusable`.
- **Check Network Connection**: Verify that your network connection is stable and that the server is accessible from your network.
- **Inspect Docker Services**: Use `docker-compose ps` to check the status of your Docker services. All services should be in the `Up` state.
- **Check Caddyfile Configuration**: Ensure that the Caddyfile configuration is correct, and that the hostname and tailnet name match those set in the Tailscale admin console.
- **Restart Docker Services**: If all else fails, try restarting your Docker services with `docker-compose down` followed by `docker-compose up -d`.
- **USB Passthrough**: Docker desktop doesn't allow for USB passthrough, so if you want to access media from a USB device, you will have to install [colima](https://github.com/abiosoft/colima)
