# Raspberry Pi Media Automation Stack Guide

## Introduction

This guide provides detailed step-by-step instructions for setting up a comprehensive media automation stack on a Raspberry Pi 4. This stack typically includes a media server (like Plex or Jellyfin), download automation tools (Sonarr for TV shows, Radarr for movies), a request system (Overseerr or Jellyseerr), and a download client (like qBittorrent). The setup will utilize your existing ASUS router NAS (Network Attached Storage) share for storing media files.

We will use Docker and Docker Compose to manage the applications, making installation and updates significantly easier.

**Prerequisites:**

*   Raspberry Pi 4 (2GB RAM minimum, 4GB+ recommended)
*   Reliable MicroSD card (32GB+ recommended, Class 10/U1 minimum)
*   Power supply for the Raspberry Pi 4
*   Ethernet cable (recommended for stable network connection) or Wi-Fi access
*   A separate computer for flashing the SD card and initial SSH access
*   Your ASUS router with the USB SSD/HDD already set up and accessible via a Samba (SMB) share on your local network (e.g., `smb://192.168.1.1/share_name`). You'll need the router's IP address, the share name, and the username/password required to access the share.

---

## Phase 1: Prepare the Raspberry Pi

This phase covers installing the operating system and performing essential initial configuration.

### Step 1.1: Choose and Flash the Operating System

We recommend using **Raspberry Pi OS Lite (64-bit)**. The "Lite" version doesn't include a desktop environment, which saves resources, and the 64-bit version is generally better for performance with applications like Plex/Jellyfin.

1.  **Download Raspberry Pi Imager:** Go to the official Raspberry Pi website ([https://www.raspberrypi.com/software/](https://www.raspberrypi.com/software/)) and download the Raspberry Pi Imager application for your computer (Windows, macOS, or Linux).
2.  **Insert SD Card:** Insert your MicroSD card into your computer.
3.  **Launch Imager:** Open the Raspberry Pi Imager application.
4.  **Choose Device:** Click "CHOOSE DEVICE" and select "Raspberry Pi 4".
5.  **Choose OS:** Click "CHOOSE OS". Navigate to "Raspberry Pi OS (other)" and select "Raspberry Pi OS Lite (64-bit)".
6.  **Choose Storage:** Click "CHOOSE STORAGE" and select your MicroSD card. **Be absolutely sure you select the correct drive, as it will be completely erased.**
7.  **Configure Settings (IMPORTANT):** Before clicking "WRITE", click the **Gear icon** (Advanced options). Configure the following:
    *   **Set hostname:** Choose a name for your Pi on the network (e.g., `media-pi`). Check the box.
    *   **Enable SSH:** Check the box and select "Use password authentication".
    *   **Set username and password:** Set a username (default is `pi`, you can keep it or change it) and a **strong password**. Remember this password!
    *   **Configure wireless LAN:** If you plan to use Wi-Fi, check the box and enter your Wi-Fi network name (SSID) and password. Using Ethernet is strongly recommended for stability.
    *   **Set locale settings:** Set your Timezone and Keyboard Layout.
8.  **Write OS:** Click "SAVE" on the settings, then click "WRITE". Confirm that you want to erase the SD card. The imager will download the OS (if needed) and write it to the card. This will take several minutes.
9.  **Eject SD Card:** Once the write and verification process is complete, safely eject the MicroSD card from your computer.

### Step 1.2: Initial Boot and SSH Access

1.  **Insert SD Card into Pi:** Put the newly flashed MicroSD card into your Raspberry Pi 4.
2.  **Connect Peripherals:** Connect the Ethernet cable (if using) and the power supply. Do **not** connect a monitor or keyboard unless you skipped the SSH setup in the imager.
3.  **Power On:** The Pi will boot up. Give it a few minutes for the first boot.
4.  **Find Pi's IP Address:** You need the IP address your router assigned to the Pi. You can find this by:
    *   Logging into your ASUS router's web interface and looking at the list of connected clients. Look for the hostname you set (e.g., `media-pi`).
    *   Using a network scanning tool on your computer (like `nmap` on Linux/macOS or apps like Fing on mobile).
5.  **Connect via SSH:** Open a terminal (on Linux/macOS) or an SSH client like PuTTY (on Windows) on your computer. Connect to the Pi using the username you set (e.g., `pi`) and its IP address:
    ```bash
    ssh pi@<Raspberry Pi IP Address>
    ```
    (Replace `<Raspberry Pi IP Address>` with the actual IP).
6.  **Accept Host Key:** The first time you connect, you'll be asked to confirm the host key. Type `yes` and press Enter.
7.  **Enter Password:** Enter the password you set in the Raspberry Pi Imager settings.

You should now be logged into your Raspberry Pi's command line!

### Step 1.3: System Updates

It's crucial to update the system software to the latest versions.

1.  **Update Package List:**
    ```bash
    sudo apt update
    ```
2.  **Upgrade Packages:**
    ```bash
    sudo apt full-upgrade -y
    ```
    This might take some time. The `-y` flag automatically confirms prompts.
3.  **Reboot (Optional but Recommended):** After a large upgrade, it's often good practice to reboot:
    ```bash
    sudo reboot
    ```
    Wait a minute or two for the Pi to restart, then reconnect via SSH.

Your Raspberry Pi is now prepared with the basic OS and is up-to-date.



---

## Phase 2: Mount Router Storage (Samba Share)

To make the storage on your ASUS router accessible to the applications running on the Pi, we need to mount the Samba (SMB) share permanently.

### Step 2.1: Install Required Packages

We need the `cifs-utils` package to mount SMB shares.

```bash
sudo apt update
sudo apt install cifs-utils -y
```

### Step 2.2: Create Mount Point

We need a directory on the Pi where the router share will be mounted.

```bash
# Create a base directory for network shares (if it doesn't exist)
sudo mkdir -p /mnt/nas

# Create the specific mount point for your router share
sudo mkdir -p /mnt/nas/router_share
```

### Step 2.3: Create Credentials File (Secure Method)

Storing your router share username and password directly in the system mount file (`fstab`) is insecure. We'll create a separate credentials file.

1.  **Create the file:**
    ```bash
    sudo nano /etc/samba/credentials/router_credentials
    ```
2.  **Add credentials:** Enter the following lines, replacing `<Your Router Share Username>` and `<Your Router Share Password>` with the actual credentials needed to access your ASUS router's Samba share:
    ```
    username=<Your Router Share Username>
    password=<Your Router Share Password>
    ```
3.  **Save and Exit:** Press `Ctrl+X`, then `Y`, then `Enter`.
4.  **Set Permissions:** Restrict permissions so only the root user can read this file:
    ```bash
    sudo chmod 600 /etc/samba/credentials/router_credentials
    sudo chown root:root /etc/samba/credentials/router_credentials
    ```

### Step 2.4: Find User/Group IDs

Docker containers run under specific user IDs (UID) and group IDs (GID). To avoid permission issues when containers write to the mounted share, we should mount the share using the UID/GID of the user running the containers (usually the main Pi user, e.g., `pi`).

1.  **Find your user's UID and GID:** Run the `id` command (make sure you are logged in as the user you intend to run Docker containers as, typically `pi`):
    ```bash
    id
    ```
    Note down the `uid` number (e.g., `1000`) and the `gid` number (e.g., `1000`).

### Step 2.5: Configure Automatic Mounting (fstab)

We'll edit the `/etc/fstab` file to make the mount permanent across reboots.

1.  **Backup fstab (Important!):**
    ```bash
    sudo cp /etc/fstab /etc/fstab.bak
    ```
2.  **Edit fstab:**
    ```bash
    sudo nano /etc/fstab
    ```
3.  **Add Mount Line:** Go to the bottom of the file and add the following line. **Carefully replace the placeholders:**
    *   Replace `<Router IP Address>` with your ASUS router's IP (e.g., `192.168.1.1`).
    *   Replace `<Share Name>` with the name of the Samba share on your router (e.g., `sda1`, `AiDisk_a1`, or whatever it's called).
    *   Replace `<UID>` and `<GID>` with the numbers you found in Step 2.4 (e.g., `1000`).

    ```fstab
    //<Router IP Address>/<Share Name> /mnt/nas/router_share cifs credentials=/etc/samba/credentials/router_credentials,uid=<UID>,gid=<GID>,vers=3.0,iocharset=utf8,nofail 0 0
    ```

    *   **Explanation of options:**
        *   `//<Router IP Address>/<Share Name>`: The network path to the share.
        *   `/mnt/nas/router_share`: The local mount point directory.
        *   `cifs`: The filesystem type.
        *   `credentials=/etc/samba/credentials/router_credentials`: Path to the secure credentials file.
        *   `uid=<UID>,gid=<GID>`: Sets the owner and group of the mounted files locally to match your Pi user, crucial for Docker permissions.
        *   `vers=3.0`: Specifies SMB protocol version 3.0 (generally good for modern routers, adjust if needed, e.g., `vers=2.1`). Hugo->Here i switch to 2.0 after a error message
        *   `iocharset=utf8`: Ensures proper handling of character encoding.
        *   `nofail`: Prevents the Pi from halting boot if the network share isn't available.
        *   `0 0`: Standard fstab dump/pass options.

4.  **Save and Exit:** Press `Ctrl+X`, then `Y`, then `Enter`.

### Step 2.6: Test the Mount

1.  **Mount all entries in fstab:**
    ```bash
    sudo mount -a
    ```
    If you get no errors, it likely worked.
2.  **Verify:** Check if the share is mounted and if you can list its contents:
    ```bash
    df -h # Look for the //<Router IP Address>/<Share Name> entry
    ls -l /mnt/nas/router_share # You should see the files/folders on your router share
    ```
3.  **Create Test Directory (Optional):** Try creating a directory to ensure write permissions are working (needed for Docker containers):
    ```bash
    mkdir /mnt/nas/router_share/test_from_pi
    ls -l /mnt/nas/router_share # Check if test_from_pi was created
    rmdir /mnt/nas/router_share/test_from_pi # Clean up
    ```

If everything works, your router's storage is now permanently mounted on your Raspberry Pi at `/mnt/nas/router_share` and ready for Docker applications to use.



---

## Phase 3: Install Docker and Docker Compose

Docker allows us to run applications in isolated environments called containers. Docker Compose helps manage multi-container applications defined in a single configuration file.

### Step 3.1: Install Docker

We will use the official convenience script to install the latest version of Docker.

1.  **Download and Run the Script:**
    ```bash
    curl -fsSL https://get.docker.com -o get-docker.sh
    sudo sh get-docker.sh
    ```
    This downloads the script and executes it with root privileges.
2.  **Clean Up Script:**
    ```bash
    rm get-docker.sh
    ```

### Step 3.2: Add User to Docker Group

To run Docker commands without needing `sudo` every time, add your user (e.g., `pi`) to the `docker` group.

```bash
sudo usermod -aG docker $USER
```

**IMPORTANT:** You need to log out and log back in (or reboot) for this group change to take effect.

```bash
# Log out of the current SSH session
exit

# Reconnect via SSH
ssh pi@<Raspberry Pi IP Address>
```

### Step 3.3: Install Docker Compose

Docker Compose is used to define and run multi-container Docker applications. It can usually be installed via the package manager on recent systems.

```bash
sudo apt update
dockersudo apt install docker-compose-plugin -y
# Note: On older systems or if the above fails, you might need to install it via pip or direct download.
# Check the official Docker documentation for the latest recommended installation method if needed.
```

### Step 3.4: Verify Installation

Check that Docker and Docker Compose are installed correctly.

1.  **Check Docker Version:**
    ```bash
    docker --version
    ```
2.  **Check Docker Compose Version:**
    ```bash
    docker compose version
    ```
3.  **Run Hello World Container (Optional):**
    ```bash
    docker run hello-world
    ```
    This downloads a small test image and runs it. If successful, it confirms Docker is working.

Docker and Docker Compose are now installed and ready for deploying the media stack applications.



---

## Phase 4: Deploy Applications with Docker Compose

Now we will create a Docker Compose file to define and launch all our media stack applications as containers.

### Step 4.1: Create Directory Structure

Organizing configuration files is important. We will create a central directory for our Docker stack.

```bash
# Create a directory in your user's home directory
mkdir ~/docker_stack
cd ~/docker_stack

# Create subdirectories for each application's configuration data
mkdir -p config/qbittorrent
mkdir -p config/sonarr
mkdir -p config/radarr
mkdir -p config/jellyfin
mkdir -p config/overseerr
```

### Step 4.2: Create the Docker Compose File

Create a file named `docker-compose.yml` within the `~/docker_stack` directory.

```bash
nano docker-compose.yml
```

Paste the following configuration into the file. **Read the comments carefully and adjust paths/values as needed.**

```yaml
version: "3.8"

services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1000 # Replace with your user's UID from Step 2.4
      - PGID=1000 # Replace with your user's GID from Step 2.4
      - TZ=Etc/UTC # Replace with your Timezone, e.g., Europe/London
      - WEBUI_PORT=8080
    volumes:
      - ./config/qbittorrent:/config
      # Mount the router share for downloads
      # IMPORTANT: Adjust the left side (/mnt/nas/router_share) if your mount point is different
      - /mnt/nas/router_share/downloads:/downloads # Completed downloads go here
      - /mnt/nas/router_share/incomplete:/incomplete # Incomplete downloads (optional)
    ports:
      - "8080:8080" # Web UI port
      - "6881:6881" # Incoming torrent connection port (TCP)
      - "6881:6881/udp" # Incoming torrent connection port (UDP)
    restart: unless-stopped

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=1000 # Replace with your user's UID
      - PGID=1000 # Replace with your user's GID
      - TZ=Etc/UTC # Replace with your Timezone
    volumes:
      - ./config/sonarr:/config
      # Mount the router share for TV shows and downloads
      # IMPORTANT: Paths on the left must match your Pi. Paths on the right are *inside* the container.
      - /mnt/nas/router_share/tv:/tv # Your TV show library location on the NAS
      - /mnt/nas/router_share/downloads:/downloads # Where qBittorrent saves completed files
    ports:
      - "8989:8989" # Web UI port
    restart: unless-stopped

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=1000 # Replace with your user's UID
      - PGID=1000 # Replace with your user's GID
      - TZ=Etc/UTC # Replace with your Timezone
    volumes:
      - ./config/radarr:/config
      # Mount the router share for Movies and downloads
      # IMPORTANT: Paths on the left must match your Pi. Paths on the right are *inside* the container.
      - /mnt/nas/router_share/movies:/movies # Your movie library location on the NAS
      - /mnt/nas/router_share/downloads:/downloads # Where qBittorrent saves completed files
    ports:
      - "7878:7878" # Web UI port
    restart: unless-stopped

  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    environment:
      - PUID=1000 # Replace with your user's UID
      - PGID=1000 # Replace with your user's GID
      - TZ=Etc/UTC # Replace with your Timezone
      # Optional: Enable hardware acceleration if needed (requires extra setup/permissions)
      # - JELLYFIN_PublishedServerUrl=http://<Raspberry Pi IP Address>:8096
    volumes:
      - ./config/jellyfin:/config
      # Mount the router share for media libraries
      # IMPORTANT: Paths on the left must match your Pi. Paths on the right are *inside* the container.
      - /mnt/nas/router_share/tv:/data/tvshows # Path inside Jellyfin for TV shows
      - /mnt/nas/router_share/movies:/data/movies # Path inside Jellyfin for movies
    ports:
      - "8096:8096" # Web UI port (HTTP)
      # - "8920:8920" # Optional: HTTPS port if you configure it
      # - "1900:1900/udp" # Optional: DLNA service discovery
      # - "7359:7359/udp" # Optional: Client discovery
    # Optional: Add device mapping for hardware acceleration (e.g., /dev/dri)
    # devices:
    #   - /dev/dri:/dev/dri
    restart: unless-stopped

  overseerr:
    image: lscr.io/linuxserver/overseerr:latest
    container_name: overseerr
    environment:
      - PUID=1000 # Replace with your user's UID
      - PGID=1000 # Replace with your user's GID
      - TZ=Etc/UTC # Replace with your Timezone
    volumes:
      - ./config/overseerr:/config
    ports:
      - "5055:5055" # Web UI port
    depends_on:
      - sonarr
      - radarr
    restart: unless-stopped

# Optional: Add other services like Prowlarr (indexer manager) or Jackett if needed.

```

**Key Adjustments:**

*   **PUID/PGID:** Replace `1000` with the actual UID/GID you found in Step 2.4.
*   **TZ:** Replace `Etc/UTC` with your correct timezone (e.g., `America/New_York`, `Europe/Paris`). A list can be found online.
*   **Volumes (`/mnt/nas/router_share/...`):** Ensure the path on the *left* side of the colon (`:`) exactly matches the location where your router share is mounted on the Pi (`/mnt/nas/router_share` from Phase 2) and the subdirectories you intend to use (e.g., `/downloads`, `/tv`, `/movies`). You might need to create these subdirectories on the NAS share first (e.g., using Nautilus connected to the share, or via `mkdir /mnt/nas/router_share/downloads` etc. on the Pi).
*   **Ports:** The ports listed (e.g., `8080`, `8989`, `7878`, `8096`, `5055`) are the default ports for these applications. If they conflict with other services on your network or Pi, you can change the port on the *left* side (e.g., `- "8081:8080"` would make qBittorrent accessible on port 8081).

### Step 4.3: Save the File

Press `Ctrl+X`, then `Y`, then `Enter` to save the `docker-compose.yml` file.

### Step 4.4: Start the Application Stack

Now, launch all the containers defined in your compose file.

```bash
# Navigate to the directory containing the docker-compose.yml file
cd ~/docker_stack

# Start the containers in detached mode (-d)
docker compose up -d
```

Docker will now download the images for each application (this might take a while the first time) and start the containers in the background.

### Step 4.5: Check Container Status

You can check if the containers are running:

```bash
docker compose ps
```

You should see output listing all the containers (qbittorrent, sonarr, radarr, jellyfin, overseerr) with a status like `Up` or `Running`.

If any containers are not running or are restarting, you can check their logs:

```bash
docker compose logs <service_name> # e.g., docker compose logs sonarr
```

Your media stack applications are now running in Docker containers!



---

## Phase 5: Configure Applications

Now that the containers are running, you need to configure each application via its web interface.

**Accessing Web UIs:**

Open a web browser on your computer (connected to the same network as the Pi) and navigate to the following addresses (replace `<Raspberry Pi IP Address>` with the actual IP):

*   **qBittorrent:** `http://<Raspberry Pi IP Address>:8080` (Default login: user `admin`, password `adminadmin` - **CHANGE THIS IMMEDIATELY**) 
*   **Sonarr:** `http://<Raspberry Pi IP Address>:8989`
*   **Radarr:** `http://<Raspberry Pi IP Address>:7878`
*   **Jellyfin:** `http://<Raspberry Pi IP Address>:8096`
*   **Overseerr:** `http://<Raspberry Pi IP Address>:5055`

### Step 5.1: Configure qBittorrent

1.  **Login:** Access `http://<Raspberry Pi IP Address>:8080`. Log in with `admin`/`adminadmin`. hugo->first time open docker logs e lá diz a password temporaria.
2.  **Change Password (CRITICAL):** Go to `Tools -> Options -> Web UI`. Under `Authentication`, change the username and set a strong password. Click `Save`.
3.  **Set Download Paths:** Go to `Tools -> Options -> Downloads`.
    *   **Default Save Path:** Set this to the path *inside the container* where completed downloads should go. Based on our `docker-compose.yml`, this is `/downloads`. 
    *   **(Optional) Incomplete Files Path:** If you mapped an `incomplete` volume, you can set `Keep incomplete torrents in:` to `/incomplete`.
    *   **Important:** Ensure `Append .!qB extension to incomplete files` is checked.
    *   **(Optional) Categories:** You can set up categories (e.g., `tv`, `movies`) and assign specific save paths for each (e.g., `/downloads/tv`, `/downloads/movies`). This helps Sonarr/Radarr pick up the files correctly. If using categories, ensure the paths exist within the `/downloads` volume mapped in Docker.
4.  **Enable Remote Control (for Sonarr/Radarr):** Go to `Tools -> Options -> Web UI`. Ensure `Enable Web User Interface (Remote control)` is checked (it should be by default).
5.  **Port Forwarding (Optional but Recommended):** For better torrent health, consider forwarding the incoming torrent port (e.g., 6881 TCP/UDP) in your ASUS router settings to your Raspberry Pi's IP address. This is outside the scope of this guide; consult your router's manual.
6.  Click `Save`.

### Step 5.2: Configure Sonarr (TV Shows)

1.  **Login:** Access `http://<Raspberry Pi IP Address>:8989`.
2.  **Media Management Settings:**
    *   Go to `Settings -> Media Management`.
    *   **Enable Advanced Settings:** Toggle `Show Advanced` (usually top right or bottom).
    *   **Root Folders:** Click `+ Add Root Folder`. Enter the path *inside the Sonarr container* where your TV shows are stored. Based on our `docker-compose.yml`, this is `/tv`. Click `Add`.
    *   **Naming:** Configure how you want your TV show files and folders named (optional).
    *   **Importing:** Ensure `Use Hardlinks instead of Copy` is **OFF** if your `/downloads` and `/tv` paths are on the *same* mounted NAS share but appear as different volumes *inside* the container (which they do in our example). If they were on physically different filesystems, hardlinks wouldn't work anyway. Atomic moves (copy+delete) are usually fine for NAS shares.
3.  **Profiles:** Go to `Settings -> Profiles`. Review the default quality profiles and release profiles. Adjust them based on the quality and types of releases you prefer.
4.  **Indexers:**
    *   Go to `Settings -> Indexers`. Click `+` to add indexers (sources where Sonarr searches for releases, e.g., public trackers or private Usenet/torrent indexers).
    *   You'll need API keys or login details for most indexers. Configure each one according to its requirements.
    *   **Prowlarr/Jackett:** If you use many indexers, consider using Prowlarr (recommended) or Jackett as an indexer manager. You would add Prowlarr/Jackett to your Docker stack and then add it as a single indexer source in Sonarr/Radarr.
5.  **Download Clients:**
    *   Go to `Settings -> Download Clients`. Click `+` to add qBittorrent.
    *   Select `qBittorrent`.
    *   **Name:** Give it a name (e.g., `qBit`).
    *   **Host:** `qbittorrent` (Docker containers in the same compose network can usually reach each other by their service name).
    *   **Port:** `8080` (or the port you configured for qBittorrent's Web UI).
    *   **Username/Password:** Enter the username and **new password** you set for qBittorrent's Web UI.
    *   **(Optional) Category:** If you set up categories in qBittorrent (e.g., `tv`), enter that category name here. This tells qBittorrent where to save Sonarr downloads.
    *   Click `Test`. It should report "Testing qBittorrent succeeded".
    *   Click `Save`.
6.  **Connect (Sonarr <-> Radarr/Overseerr):** Go to `Settings -> Connect`. You can add connections here later if needed (e.g., to notify Overseerr when a download completes).
7.  **Add Your First Show:** Go to `Series -> Add New`. Search for a TV show, select it, choose the correct Root Folder (`/tv`), select a Quality Profile, and click `Add + Search`.

### Step 5.3: Configure Radarr (Movies)

Radarr's configuration is very similar to Sonarr's.

1.  **Login:** Access `http://<Raspberry Pi IP Address>:7878`.
2.  **Media Management Settings:**
    *   Go to `Settings -> Media Management`.
    *   **Enable Advanced Settings.**
    *   **Root Folders:** Click `+ Add Root Folder`. Enter the path *inside the Radarr container* for your movies. Based on our `docker-compose.yml`, this is `/movies`. Click `Add`.
    *   **Naming:** Configure movie naming (optional).
    *   **Importing:** Check hardlink settings as described for Sonarr.
3.  **Profiles:** Go to `Settings -> Profiles`. Review quality profiles.
4.  **Indexers:** Go to `Settings -> Indexers`. Add your indexers just like you did in Sonarr.
5.  **Download Clients:**
    *   Go to `Settings -> Download Clients`. Click `+` and select `qBittorrent`.
    *   Configure it exactly like you did in Sonarr (Host: `qbittorrent`, Port: `8080`, Username/Password).
    *   **(Optional) Category:** Enter the category for movies if you set one up in qBittorrent (e.g., `movies`).
    *   Test and Save.
6.  **Connect:** Go to `Settings -> Connect` (for later use).
7.  **Add Your First Movie:** Go to `Movies -> Add New`. Search for a movie, select it, choose the Root Folder (`/movies`), Quality Profile, and click `Add + Search`.

### Step 5.4: Configure Jellyfin (Media Server)

1.  **Login:** Access `http://<Raspberry Pi IP Address>:8096`.
2.  **Welcome Wizard:** The first time you access Jellyfin, you'll go through a setup wizard.
    *   **Language:** Select your preferred language.
    *   **Create User:** Create your primary admin user account (username and password).
    *   **Setup Media Libraries:** Click `Add Media Library`.
        *   **Content Type:** Select `Shows`.
        *   **Display Name:** Enter a name (e.g., `TV Shows`).
        *   **Folders:** Click `+ Add Folder`. Enter the path *inside the Jellyfin container* where your TV shows are stored. Based on our `docker-compose.yml`, this is `/data/tvshows`. Click `OK`.
        *   Configure other library settings (metadata downloaders, etc.) as desired. Click `OK`.
        *   Click `Add Media Library` again.
        *   **Content Type:** Select `Movies`.
        *   **Display Name:** Enter a name (e.g., `Movies`).
        *   **Folders:** Click `+ Add Folder`. Enter the path *inside the Jellyfin container* for movies. Based on our `docker-compose.yml`, this is `/data/movies`. Click `OK`.
        *   Configure other settings. Click `OK`.
    *   **Metadata Language:** Select your preferred language for metadata.
    *   **Remote Access:** Configure remote access settings (usually leave defaults unless you know you need changes).
    *   Click `Finish`.
3.  **Explore:** Log in with the user you created. Jellyfin will start scanning the libraries you added. Explore the `Dashboard` (under the admin user menu) for more advanced settings like transcoding, users, plugins, etc.

### Step 5.5: Configure Overseerr (Request System)

1.  **Login:** Access `http://<Raspberry Pi IP Address>:5055`.
2.  **Welcome Wizard:** Overseerr also has a setup wizard.
    *   **Sign in with Plex/Jellyfin:** Choose `Jellyfin`.
    *   **Jellyfin Settings:**
        *   **Hostname or IP Address:** `jellyfin` (Docker service name).
        *   **Port:** `8096`.
        *   Click `Authenticate with Jellyfin`. A popup will ask for your Jellyfin username and password. Enter them and grant access.
        *   Select your Jellyfin server from the list.
        *   Click `Save Changes`.
    *   **Connect to Sonarr:**
        *   Click `Add Sonarr Server`.
        *   **Server Name:** Give it a name (e.g., `Sonarr`).
        *   **Hostname or IP Address:** `sonarr` (Docker service name).
        *   **Port:** `8989`.
        *   **API Key:** Go to Sonarr's Web UI -> `Settings -> General`. Copy the API Key and paste it here.
        *   Select the correct Quality Profiles and Root Folder (`/tv`) that you configured in Sonarr.
        *   Enable `SSL` only if you configured Sonarr for SSL (we didn't in this basic setup).
        *   Click `Test`. It should succeed.
        *   Click `Add Server`.
    *   **Connect to Radarr:**
        *   Click `Add Radarr Server`.
        *   **Server Name:** Give it a name (e.g., `Radarr`).
        *   **Hostname or IP Address:** `radarr` (Docker service name).
        *   **Port:** `7878`.
        *   **API Key:** Go to Radarr's Web UI -> `Settings -> General`. Copy the API Key and paste it here.
        *   Select the correct Quality Profiles and Root Folder (`/movies`) that you configured in Radarr.
        *   Enable `SSL` only if needed.
        *   Click `Test`. It should succeed.
        *   Click `Add Server`.
    *   Click `Finish Setup`.
3.  **Create User:** Create your main Overseerr user account (this is separate from Jellyfin).
4.  **Explore:** Log in and explore Overseerr. You should be able to browse media and request items. Requests should automatically be sent to Sonarr/Radarr.

All the core applications should now be configured to work together!



---

## Phase 6: Validate the Workflow

Now, let's test the entire process to ensure everything is working together correctly.

1.  **Request Media via Overseerr:**
    *   Open Overseerr (`http://<Raspberry Pi IP Address>:5055`).
    *   Search for a readily available movie or a specific episode of a TV show.
    *   Click `Request`.

2.  **Check Sonarr/Radarr:**
    *   Open Sonarr (for TV) or Radarr (for movies).
    *   Look in the `Activity` queue. You should see the requested item appear and transition through searching/grabbing states.

3.  **Check qBittorrent:**
    *   Open qBittorrent (`http://<Raspberry Pi IP Address>:8080`).
    *   You should see the torrent for the requested media added and downloading (or seeding if it finishes quickly).
    *   Wait for the download to complete 100%.

4.  **Monitor File Processing:**
    *   Keep an eye on Sonarr/Radarr's `Activity` queue again.
    *   Once the download completes in qBittorrent, Sonarr/Radarr should detect it (this might take a minute or two based on scan intervals).
    *   It should then process the file: rename it according to your settings and move it from the `/downloads` directory to your library directory (`/tv` or `/movies`).
    *   You can verify this by checking the contents of `/mnt/nas/router_share/tv` or `/mnt/nas/router_share/movies` on your Pi, or by browsing the NAS share directly.

5.  **Check Jellyfin Library:**
    *   Open Jellyfin (`http://<Raspberry Pi IP Address>:8096`).
    *   Navigate to the appropriate library (TV Shows or Movies).
    *   Jellyfin should automatically detect the new file and add it to the library. This might take a few minutes depending on your library scan settings (you can manually trigger a scan in `Dashboard -> Libraries` if you're impatient).
    *   Verify the metadata (artwork, description) is downloaded correctly.

6.  **Test Playback:**
    *   Using a Jellyfin client app (on your Android TV, phone, computer web browser, etc.), find the newly added movie or episode.
    *   Start playback.
    *   Check for smooth playback without excessive buffering. If you encounter issues, check the Jellyfin dashboard for transcoding status and network conditions.

If you can successfully request, download, organize, and play media through this workflow, your setup is complete!

---

## Maintenance and Further Steps

*   **Updates:** Regularly update the Raspberry Pi OS (`sudo apt update && sudo apt full-upgrade -y`) and the Docker containers (`cd ~/docker_stack && docker compose pull && docker compose up -d`). Check the documentation for each container image before major updates.
*   **Backups:** Regularly back up the configuration volumes (`~/docker_stack/config`) for your containers. This saves your settings if you need to rebuild the stack.
*   **Security:** Ensure you use strong passwords everywhere. Consider setting up HTTPS/SSL for remote access if needed (requires more advanced configuration, potentially with a reverse proxy like Nginx Proxy Manager).
*   **Monitoring:** Tools like `htop` (install with `sudo apt install htop`) can help monitor your Pi's resource usage.
*   **Hardware Acceleration:** If you experience buffering due to transcoding in Jellyfin, research enabling hardware acceleration for the Raspberry Pi 4 (might require specific Jellyfin image versions, kernel modules, and Docker permissions).

This guide provides a solid foundation for your automated media server. Enjoy!

