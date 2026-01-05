🚗 Spotify Pi Car Player (3.5" Touchscreen)
A touch-screen graphical interface (GUI) written in Python to control Spotify on Raspberry Pi. Designed specifically for 3.5-inch displays (480x320), making it ideal for Car Audio projects or smart home radios.

(Replace this link with a real photo of your finished project!)

✨ Features
Touch-Friendly Interface: Large buttons and layout optimized for small screens.

Album Art: Displays the high-quality cover art of the current track.

Library Navigation: Browse your Playlists and "Liked Songs" directly from the Pi.

Volume Control: Touch slider with +/- buttons and a quick "MAX" button.

System Controls: Buttons to minimize the app or safely shutdown the Raspberry.

Powered by: Uses official Spotify APIs (via spotipy).

🛠️ Hardware Requirements
Raspberry Pi (3B+, 4, or Zero 2 W recommended).

3.5" Touch Display (e.g., Waveshare, MPI3508, or generic GPIO).

MicroSD Card (8GB+).

Audio Output (3.5mm Jack, USB DAC, or HiFiBerry).

⚙️ Software Prerequisites
Raspberry Pi OS (Desktop version, required for Tkinter).

Spotify Premium account.

Raspotify (or another Spotify Connect client) installed on the Pi to handle audio playback.

🚀 Step-by-Step Installation
1. System Preparation
Update your Raspberry Pi and install the necessary system dependencies for audio, images, and GUI.

Bash

sudo apt update && sudo apt upgrade -y
sudo apt install -y python3-pip python3-tk python3-pil.imagetk libopenjp2-7 libasound2-dev alsa-utils git
2. Install Audio Client (Raspotify)
This interface controls the music logic, but to actually hear it, the Raspberry must be visible as a Spotify Connect device.

Bash

curl -sL https://dtcooper.github.io/raspotify/install.sh | sh
After installation, you can edit /etc/raspotify/conf if you need to change the device name or bitrate.

3. Spotify Developer Configuration
To use the API, you need to generate credentials:

Go to the Spotify Developer Dashboard.

Create a new App.

Copy the Client ID and Client Secret.

In the App settings, add this Redirect URI: http://127.0.0.1:8888/callback

4. Download Project
Clone this repository (or create the folder manually):

Bash

mkdir ~/SpotifyPi
cd ~/SpotifyPi
5. Install Python Libraries
Bash

pip3 install spotipy requests pillow --break-system-packages
(Note: On recent Raspberry Pi OS versions, the --break-system-packages flag might be required, or you can use a virtual environment).

6. Configure Credentials
Create a file named config.py in the project folder:

Bash

nano config.py
Paste the following content, inserting your specific data:

Python

# config.py
CLIENT_ID = 'PASTE_YOUR_CLIENT_ID_HERE'
CLIENT_SECRET = 'PASTE_YOUR_CLIENT_SECRET_HERE'
REDIRECT_URI = 'http://127.0.0.1:8888/callback'
SCOPE = "user-read-currently-playing,user-read-playback-state,user-modify-playback-state,playlist-read-private,playlist-read-collaborative,user-library-read"
TARGET_DEVICE_NAME = "raspotify" # Or the name you gave your device
7. The Main Code
Download main.py (ensure you use the latest version of the code provided in this repo).

<details> <summary><b>👉 Click here to verify the main code structure</b></summary>

(Paste the Python code from our previous conversation here when creating the file on GitHub)

</details>

8. Autostart (Optional)
To launch the interface automatically when the Raspberry Pi turns on:

Bash

mkdir -p ~/.config/autostart
nano ~/.config/autostart/spotify-pi.desktop
Paste this content:

Ini, TOML

[Desktop Entry]
Type=Application
Name=SpotifyPi
Exec=python3 /home/pi/SpotifyPi/main.py
Terminal=false
(Make sure the path /home/pi/SpotifyPi/main.py matches your actual username and folder!)

🎮 How to Use
First Launch: The script will ask you to authenticate via a browser. Open the URL shown in the terminal on your PC/Phone to authorize the app.

Home Screen: Displays cover art, title, artist, and basic playback controls.

Library: Press ☰ to view your Playlists. Click on a playlist to open the tracklist, then click a song to play it.

Volume: Click the Speaker icon 🔊 to open the volume overlay (+ / - buttons). Press MAX for instant 100% volume.

Exit: Press - to close the interface or X to shutdown the Raspberry Pi.
