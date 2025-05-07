# Sonarr & Radarr Indexer Setup Guide

## Introduction

Indexers are the sources Sonarr and Radarr use to find releases (downloads) for the TV shows and movies you want. Without properly configured indexers, Sonarr and Radarr cannot automate your downloads. This guide explains two main approaches to setting them up:

1.  **Direct Indexer Setup:** Adding indexers one by one directly into Sonarr and Radarr.
2.  **Using Jackett:** Using a separate application called Jackett to manage multiple indexers and present them as a single source to Sonarr and Radarr.

We assume you have Sonarr and Radarr running (ideally in Docker, as per the previous guide) and accessible via their web interfaces.

---

## Approach 1: Direct Indexer Setup

This involves finding the specific details for each indexer you want to use and adding them individually in both Sonarr and Radarr (or just one, if the indexer only supports TV or movies).

### Step 1.1: Understanding Indexer Types

*   **Public Indexers:** These are generally public torrent trackers. They are easy to find but may have limited content, lower retention, or potentially less safe releases. Some may still require registration.
*   **Private Indexers:** These are usually private torrent trackers or Usenet indexers. They often require an invitation to join and may have rules regarding sharing ratios (for torrents) or subscription fees (for Usenet indexers). They typically offer better content selection, speed, retention, and security.

### Step 1.2: Finding Indexer Details

This is the most crucial part and depends entirely on the specific indexer you want to add.

1.  **Identify Indexers:** Search online forums (like Reddit communities dedicated to Sonarr, Radarr, Usenet, or torrents) for recommendations of reputable public or private indexers relevant to your needs.
2.  **Register/Join:** For most indexers (even many public ones), you will need to register an account on their website.
3.  **Locate API Key and URL:** Once registered and logged in, navigate the indexer's website. Look for sections like "Profile", "Settings", "API", or "Integration". You are typically looking for:
    *   **API Key:** A unique string of characters that authenticates Sonarr/Radarr with your account on the indexer.
    *   **Site URL / API URL:** The specific web address Sonarr/Radarr needs to communicate with the indexer. Sometimes this is just the main site URL, other times it's a specific API endpoint (e.g., `https://indexer.example.com/api`).
    *   **(For Torrents) Passkey:** Some private torrent trackers use a "passkey" embedded in the torrent download URL instead of, or in addition to, an API key for searching. Sonarr/Radarr usually handle this automatically if you provide the correct tracker URL.
    *   **(For Usenet) Site ID / Categories:** Usenet indexers often require specific category IDs (e.g., `2040` for HD Movies, `5070` for HD TV) to be entered in Sonarr/Radarr to ensure searches are efficient. These are usually listed on the indexer's site or in Sonarr/Radarr's presets.

**Example (Conceptual - Details Vary Greatly):**

*   You join a private Usenet indexer called "AwesomeNZB".
*   You log in, go to your profile page.
*   You find your **API Key:** `a1b2c3d4e5f67890abcdef1234567890`.
*   The site documentation or Sonarr/Radarr preset tells you the **API URL** is `https://api.awesomenzb.com`.
*   It also lists relevant **Category IDs**: `5030` (SD TV), `5040` (HD TV), `2030` (SD Movies), `2040` (HD Movies).

### Step 1.3: Adding the Indexer in Sonarr/Radarr

The process is nearly identical in both applications.

1.  **Open Sonarr or Radarr Web UI.**
2.  Go to `Settings -> Indexers`.
3.  Click the `+` button to add a new indexer.
4.  **Find the Preset:** Search for the name of your indexer (e.g., "AwesomeNZB"). Sonarr/Radarr have built-in presets for many popular indexers.
    *   If a preset exists, click it. It will pre-fill some settings like the URL and category IDs.
    *   If no preset exists, you might need to choose a generic type like `Torznab` (common for torrent indexers supporting a standard API) or `Newznab` (common for Usenet indexers).
5.  **Configure Settings:**
    *   **Name:** Give the indexer a recognizable name.
    *   **Enable:** Ensure it's checked.
    *   **URL:** Enter the Site URL or API URL you found. (Often pre-filled by the preset).
    *   **API Path:** Sometimes required for generic presets (e.g., `/api`).
    *   **API Key / Passkey:** Enter the API Key you found.
    *   **Categories:** Select the relevant categories (e.g., `5030`, `5040` for Sonarr; `2030`, `2040` for Radarr). Presets often pre-fill these.
    *   **Other Settings:** Configure any other specific settings required by the indexer (check tooltips `(?)` or the indexer's documentation).
6.  **Test:** Click the `Test` button. Sonarr/Radarr will attempt to connect to the indexer using the details provided.
    *   **Success:** You should see green checkmarks indicating the connection and authentication worked.
    *   **Failure:** You'll get an error message. Double-check the URL, API Key, and any other settings. Common issues include typos, incorrect URLs, expired API keys, or network connectivity problems.
7.  **Save:** If the test succeeds, click `Save`.

Repeat this process for every indexer you want to add, in both Sonarr and Radarr.



---

## Approach 2: Using Jackett

Jackett acts as a proxy server for your indexers. You add your indexers (especially torrent trackers) to Jackett, and then you add Jackett itself as a single indexer source (using the "Torznab" feed format) in Sonarr and Radarr. This is particularly useful for indexers that don\u0027t have built-in presets in Sonarr/Radarr or if you want to manage many torrent indexers in one place.

*(Note: Prowlarr is a newer alternative to Jackett with better integration into the *arr ecosystem, but Jackett is still widely used and effective.)*

### Step 2.1: Add Jackett to Docker Compose

1.  **Edit your `docker-compose.yml` file:**
    ```bash
    cd ~/docker_stack
    nano docker-compose.yml
    ```
2.  **Add the Jackett service:** Add the following service definition alongside your other services (like sonarr, radarr, etc.). Adjust `PUID`, `PGID`, and `TZ` as you did for the other containers.

    ```yaml
      jackett:
        image: lscr.io/linuxserver/jackett:latest
        container_name: jackett
        environment:
          - PUID=1000 # Replace with your user\u0027s UID
          - PGID=1000 # Replace with your user\u0027s GID
          - TZ=Etc/UTC # Replace with your Timezone
          - AUTO_UPDATE=true # Optional: Allows Jackett to update itself
        volumes:
          - ./config/jackett:/config
          - /mnt/nas/router_share/downloads/blackhole:/downloads # Optional: For blackhole downloads if needed
        ports:
          - "9117:9117" # Web UI port
        restart: unless-stopped
    ```
3.  **Save and Exit:** Press `Ctrl+X`, then `Y`, then `Enter`.
4.  **Start Jackett:** Bring up the new container (this won\u0027t affect your other running containers):
    ```bash
    docker compose up -d jackett
    # Or restart the whole stack: docker compose up -d
    ```

### Step 2.2: Configure Jackett

1.  **Access Jackett Web UI:** Open `http://<Raspberry Pi IP Address>:9117` in your browser.
2.  **Find Your Jackett API Key:** Look near the top right of the Jackett interface. You\u0027ll see an "API Key". Copy this key; you\u0027ll need it later.
3.  **Add Indexers within Jackett:**
    *   Click the `+ Add indexer` button.
    *   Browse the list or search for the indexers you want to add (public or private torrent trackers).
    *   Click the wrench icon (`Configure`) next to the indexer you want to add.
    *   Fill in any required details, such as username, password, 2FA codes, cookies, etc., as prompted. The requirements vary greatly between indexers.
    *   Click `Okay`. Jackett will test the connection.
    *   If successful, the indexer will be added to your list on the main Jackett page.
    *   Repeat for all the torrent indexers you want to use.

### Step 2.3: Add Jackett to Sonarr/Radarr

Now, instead of adding each indexer individually to Sonarr/Radarr, you add Jackett once.

1.  **Open Sonarr or Radarr Web UI.**
2.  Go to `Settings -> Indexers`.
3.  Click the `+` button.
4.  **Choose Preset:** Search for and select `Torznab`.
5.  **Configure Torznab Settings:**
    *   **Name:** Give it a name (e.g., `Jackett All`, or you can add specific feeds later).
    *   **Enable:** Ensure checked.
    *   **URL:** This needs to point to Jackett\u0027s Torznab feed. The base URL is `http://jackett:9117/api/v2.0/indexers/all/results/torznab/`. 
        *   Use `http://jackett:9117` because Sonarr/Radarr can reach Jackett via its service name within the Docker network.
        *   `/api/v2.0/indexers/all/results/torznab/` provides a feed combining *all* your configured Jackett indexers. You can also get URLs for specific indexers from the Jackett Web UI if you prefer to add them separately in Sonarr/Radarr.
    *   **API Path:** Leave blank (it\u0027s part of the URL).
    *   **API Key:** Paste the Jackett API Key you copied from the Jackett Web UI.
    *   **Categories:** Select the appropriate categories (e.g., TV and Movie categories).
    *   **Anime Categories:** Select if needed.
    *   **(Optional) Additional Parameters:** Sometimes needed for specific setups.
6.  **Test:** Click the `Test` button. It should succeed if Sonarr/Radarr can reach Jackett and the API key is correct.
7.  **Save:** Click `Save`.

Now, when Sonarr or Radarr searches for media, they will query Jackett. Jackett will then query all the individual indexers you configured within it and return the combined results.

---

## Choosing Between Approaches

*   **Direct Setup:** Simpler if you only use a few indexers that have native presets in Sonarr/Radarr (especially Usenet indexers).
*   **Jackett:** Better if you use many torrent trackers, especially those without native Sonarr/Radarr support, as you manage them in one place. Requires running an extra container.
*   **Prowlarr:** A newer alternative to Jackett, often preferred for its tighter integration with the *arr stack, but follows a similar principle.

You can also use a hybrid approach (e.g., add Usenet indexers directly and use Jackett for torrent trackers).

