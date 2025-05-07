# Media Center Project

Welcome to the Media Center project! This repository is dedicated to building a centralized platform for managing and accessing your movies and series. The goal is to provide a seamless experience for users to organize, search, and enjoy their media collection.

## Features

- **Media Organization**: Easily categorize and manage your content.
- **Search and Filter**: Quickly find specific files using advanced search and filtering options.
- **Cross-Platform Support**: Access your media from multiple devices.
- **Customizable Interface**: Tailor the platform to suit your preferences.

## Getting Started

To get started, clone this repository and follow the setup instructions in the [Installation Guide](#).
After setting up, you can run:
1. **Run qbittorrent**: open the web interface at `http://localhost:8080` and log in with the default credentials (admin/adminadmin) to manage your torrents.
2. **Run radarr**: open the web interface at `http://localhost:7878` to manage your movies.
3. **Run sonarr**: open the web interface at `http://localhost:8989` to manage your series.
4. **Run jacket**: open the web interface at `http://localhost:10000` to manage your indexers.
5. **Run jellyfin**: open the web interface at `http://localhost:8096` to manage your media library.
6. **Run jellyseerr**: open the web interface at `http://localhost:5055` to manage your media requests.

## Installation guide

1. Clone the repository:
   ```bash
   git clone
2. Navigate to the project directory:
   ```bash
   cd media-center
   ```
3. Install the required dependencies:
   ```bash
   sudo apt install docker docker-compose
   ```
4. Start the application from the root directory using docker-compose.yml:
    ```bash
    docker-compose up -d
    ```
## Next Steps
1. install a jellyseerr agent on device to manage your requests media.
2. Fill in the jackett indexers with more indexers to get more content.
3. Finish the sonarr setup to get your series.
4. Try another solution like plex or emby to manage your media.
5. Explore solutions to install on android xaiomi devices to work as catalog, like plex, emby, jellyfin, etc.

## License

This project is licensed under the [MIT License](LICENSE).

---
Feel free to explore and enhance the Media Center project. Happy coding!