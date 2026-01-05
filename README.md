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

import tkinter as tk
from PIL import Image, ImageTk, ImageDraw
import spotipy
from spotipy.oauth2 import SpotifyOAuth
import requests
from io import BytesIO
import threading
import time
import subprocess
import os
import hashlib
import sys

# --- CONFIGURAZIONE ---
try:
    sys.path.append(os.path.dirname(os.path.abspath(__file__)))
    from config import CLIENT_ID, CLIENT_SECRET, REDIRECT_URI, SCOPE, TARGET_DEVICE_NAME
except ImportError:
    print("Avviso: config.py non trovato.")
    sys.exit(1)

CACHE_DIR = ".img_cache"
if not os.path.exists(CACHE_DIR):
    os.makedirs(CACHE_DIR)

sp = spotipy.Spotify(auth_manager=SpotifyOAuth(
    client_id=CLIENT_ID, client_secret=CLIENT_SECRET, redirect_uri=REDIRECT_URI,
    scope=SCOPE, cache_path=os.path.join(os.path.dirname(__file__), ".spotify_cache"),
    open_browser=False
))

# --- COLORI ---
BG_COLOR = "#000000"        
ACCENT_COLOR = "#1DB954"    
GRAY_COLOR = "#B3B3B3"
WHITE_COLOR = "#FFFFFF"
PURPLE_COLOR = "#4c1bf5"

def format_time(ms):
    if ms is None: return "0:00"
    seconds = int((ms / 1000) % 60)
    minutes = int((ms / (1000 * 60)) % 60)
    return f"{minutes}:{seconds:02d}"

# --- GESTORE IMMAGINI ---
class ImageLoader:
    @staticmethod
    def get_image(url, size, radius=0):
        if not url: return None
        filename = hashlib.md5(url.encode('utf-8')).hexdigest() + ".png"
        file_path = os.path.join(CACHE_DIR, filename)
        try:
            image = None
            if os.path.exists(file_path):
                try:
                    image = Image.open(file_path)
                    image.load()
                except: pass
            
            if image is None:
                try:
                    response = requests.get(url, timeout=3)
                    image = Image.open(BytesIO(response.content))
                    image.save(file_path)
                except: return None

            image = image.resize(size, Image.Resampling.LANCZOS)
            
            if radius > 0:
                mask = Image.new('L', size, 0)
                draw = ImageDraw.Draw(mask)
                draw.rounded_rectangle((0, 0) + size, radius=radius, fill=255)
                image.putalpha(mask)
            
            return ImageTk.PhotoImage(image)
        except Exception as e:
            return None

# --- APP PRINCIPALE ---
class SpotifyCarUI:
    def __init__(self, root):
        self.root = root
        self.screen_w = 480
        self.screen_h = 320
        
        self.root.geometry(f"{self.screen_w}x{self.screen_h}+0+0")
        self.root.overrideredirect(True)
        self.root.configure(bg=BG_COLOR)
        
        self.container = tk.Frame(root, bg=BG_COLOR)
        self.container.pack(fill="both", expand=True)
        
        self.frames = {}
        for F in (PlayerView, PlaylistListView, TrackListView):
            page_name = F.__name__
            frame = F(parent=self.container, controller=self)
            self.frames[page_name] = frame
            frame.grid(row=0, column=0, sticky="nsew")

        self.container.grid_rowconfigure(0, weight=1)
        self.container.grid_columnconfigure(0, weight=1)

        self.show_frame("PlayerView")
        self.start_spotify_loop()

    def show_frame(self, page_name, data=None):
        frame = self.frames[page_name]
        frame.tkraise()
        if page_name == "PlaylistListView":
            frame.load_playlists()
        elif page_name == "TrackListView" and data:
            frame.load_tracks(data)

    def start_spotify_loop(self):
        t = threading.Thread(target=self.loop_check, daemon=True)
        t.start()

    def loop_check(self):
        while True:
            try:
                current = sp.current_user_playing_track()
                if current and current['item']:
                    self.root.after(0, self.frames["PlayerView"].update_from_spotify, current)
            except: pass
            time.sleep(2.0)

# --- PLAYER VIEW ---
class PlayerView(tk.Frame):
    def __init__(self, parent, controller):
        tk.Frame.__init__(self, parent, bg=BG_COLOR)
        self.controller = controller
        self.canvas = tk.Canvas(self, bg=BG_COLOR, highlightthickness=0)
        self.canvas.pack(fill="both", expand=True)
        
        w, h = 480, 320
        
        # --- HEADER ---
        btn_lib = tk.Button(self, text="☰ Libreria", font=("Arial", 10, "bold"),
                           bg="#222", fg="white", bd=1, relief="solid",
                           command=lambda: controller.show_frame("PlaylistListView"))
        self.canvas.create_window(10, 10, window=btn_lib, anchor="nw", width=80, height=30)

        btn_kill = tk.Button(self, text="-", font=("Arial", 12, "bold"),
                            bg="#222", fg="white", bd=1, relief="solid",
                            command=sys.exit)
        self.canvas.create_window(w-85, 10, window=btn_kill, anchor="nw", width=35, height=30)

        btn_off = tk.Button(self, text="✕", font=("Arial", 10, "bold"),
                           bg="#222", fg="white", bd=1, relief="solid", activebackground="red",
                           command=self.confirm_shutdown)
        self.canvas.create_window(w-45, 10, window=btn_off, anchor="nw", width=35, height=30)


        # --- COPERTINA ---
        self.cover_label = tk.Label(self, bg=BG_COLOR)
        self.canvas.create_window(w/2, 100, window=self.cover_label, anchor="center")
        self.cover_img_ref = None 

        # --- TESTO ---
        self.text_title = self.canvas.create_text(w/2, 165, text="In attesa...", 
                                                  font=("Arial", 14, "bold"), fill="white", width=400)
        self.text_artist = self.canvas.create_text(w/2, 185, text="Spotify Pi", 
                                                   font=("Arial", 11), fill="#DDD", width=400)

        # --- PROGRESS BAR ---
        self.bar_bg = self.canvas.create_rectangle(40, 215, 440, 218, fill="#333", outline="")
        self.bar_fg = self.canvas.create_rectangle(40, 215, 40, 218, fill=WHITE_COLOR, outline="")
        self.txt_curr = self.canvas.create_text(40, 225, text="0:00", fill="#888", anchor="w", font=("Arial", 8))
        self.txt_tot = self.canvas.create_text(440, 225, text="0:00", fill="#888", anchor="e", font=("Arial", 8))

        # --- CONTROLLI ---
        y_btns = 275
        cx = w / 2
        
        self.btn_prev = self.canvas.create_text(cx - 70, y_btns, text="⏮", font=("Arial", 25), fill="white")
        self.play_txt = self.canvas.create_text(cx, y_btns, text="▶", font=("Arial", 28, "bold"), fill="white")
        self.btn_next = self.canvas.create_text(cx + 70, y_btns, text="⏭", font=("Arial", 25), fill="white")
        
        # TASTO MAX
        self.btn_max = tk.Button(self, text="MAX", font=("Arial", 10, "bold"), bg="#333", fg="white", 
                                command=self.set_max_volume)
        self.canvas.create_window(40, y_btns, window=self.btn_max, width=50, height=35)

        # ICONA VOLUME
        self.btn_vol = self.canvas.create_text(w-40, y_btns, text="🔊", font=("Arial", 24), fill="white")
        
        # --- LOGICA VOLUME ---
        self.is_vol_visible = False
        self.vol_timer = None
        
        # Binding
        self.canvas.tag_bind(self.play_txt, "<Button-1>", lambda e: self.toggle_play())
        self.canvas.tag_bind(self.btn_prev, "<Button-1>", lambda e: sp.previous_track())
        self.canvas.tag_bind(self.btn_next, "<Button-1>", lambda e: sp.next_track())
        self.canvas.tag_bind(self.btn_vol, "<Button-1>", lambda e: self.open_volume_controls())

        self.last_img_url = ""

    def open_volume_controls(self):
        if self.is_vol_visible:
            self.reset_vol_timer()
            return
        self.is_vol_visible = True
        
        # Overlay nero sui controlli
        self.vol_overlay_bg = self.canvas.create_rectangle(0, 240, 480, 320, fill=BG_COLOR, outline="")
        
        y_pos = 275
        self.btn_vol_down = tk.Button(self, text="-", font=("Arial", 18, "bold"), bg="#333", fg="white",
                                      command=lambda: self.change_volume(-10))
        self.canvas.create_window(100, y_pos, window=self.btn_vol_down, width=50, height=40)
        
        self.btn_vol_up = tk.Button(self, text="+", font=("Arial", 18, "bold"), bg="#333", fg="white",
                                    command=lambda: self.change_volume(+10))
        self.canvas.create_window(380, y_pos, window=self.btn_vol_up, width=50, height=40)
        
        self.vol_bar_bg = self.canvas.create_rectangle(150, y_pos-10, 330, y_pos+10, fill="#333", outline="white")
        self.vol_bar_fg = self.canvas.create_rectangle(150, y_pos-10, 150, y_pos+10, fill=ACCENT_COLOR, outline="")
        self.lbl_vol_pct = self.canvas.create_text(240, y_pos, text="VOL", font=("Arial", 10, "bold"), fill="white")
        
        self.current_volume = 50 # Default safe
        try:
             # Prova a leggere il volume attuale se possibile
             pass 
        except: pass
        
        self.update_vol_bar_graphic()
        self.reset_vol_timer()

    def close_volume_controls(self):
        if not self.is_vol_visible: return
        self.canvas.delete(self.vol_overlay_bg)
        self.canvas.delete(self.vol_bar_bg)
        self.canvas.delete(self.vol_bar_fg)
        self.canvas.delete(self.lbl_vol_pct)
        if self.btn_vol_down: self.btn_vol_down.destroy()
        if self.btn_vol_up: self.btn_vol_up.destroy()
        self.is_vol_visible = False

    def reset_vol_timer(self):
        if self.vol_timer: self.after_cancel(self.vol_timer)
        self.vol_timer = self.after(3500, self.close_volume_controls)

    def change_volume(self, step):
        self.reset_vol_timer()
        self.current_volume += step
        if self.current_volume > 100: self.current_volume = 100
        if self.current_volume < 0: self.current_volume = 0
        threading.Thread(target=self._apply_volume).start()
        self.update_vol_bar_graphic()

    def _apply_volume(self):
        try:
            sp.volume(self.current_volume)
            subprocess.run(f"amixer -c 1 set Speaker {self.current_volume}% unmute", shell=True)
        except: pass

    def update_vol_bar_graphic(self):
        width = 180
        pct = self.current_volume / 100
        x_end = 150 + (width * pct)
        self.canvas.coords(self.vol_bar_fg, 150, 265, x_end, 285)
        self.canvas.itemconfig(self.lbl_vol_pct, text=f"{self.current_volume}%")

    def set_max_volume(self):
        self.current_volume = 100
        threading.Thread(target=self._apply_volume).start()
        self.btn_max.config(bg=ACCENT_COLOR)
        self.after(500, lambda: self.btn_max.config(bg="#333"))

    def confirm_shutdown(self):
        win = tk.Toplevel(self)
        win.geometry("280x120+100+100")
        win.overrideredirect(True); win.configure(bg="#222")
        tk.Label(win, text="Spegnere?", font=("Arial", 14), bg="#222", fg="white").pack(pady=20)
        f = tk.Frame(win, bg="#222"); f.pack()
        tk.Button(f, text="NO", command=win.destroy).pack(side="left", padx=10)
        tk.Button(f, text="SÌ", bg="red", fg="white", command=lambda: subprocess.run("sudo shutdown now", shell=True)).pack(side="left", padx=10)

    def update_from_spotify(self, current):
        try:
            track = current['item']
            is_playing = current['is_playing']
            progress = current['progress_ms']
            duration = track['duration_ms']
            
            self.canvas.itemconfig(self.play_txt, text="⏸" if is_playing else "▶")
            self.canvas.itemconfig(self.text_title, text=track['name'])
            self.canvas.itemconfig(self.text_artist, text=track['artists'][0]['name'])
            
            # Update Cover (100x100)
            img_url = track['album']['images'][0]['url']
            if img_url != self.last_img_url:
                self.last_img_url = img_url
                photo = ImageLoader.get_image(img_url, (100, 100), radius=5)
                self.cover_label.config(image=photo)
                self.cover_img_ref = photo 

            pct = progress / duration
            self.canvas.coords(self.bar_fg, 40, 215, 40 + (400 * pct), 218)
            self.canvas.itemconfig(self.txt_curr, text=format_time(progress))
            self.canvas.itemconfig(self.txt_tot, text=format_time(duration))
        except: pass

    def toggle_play(self):
        try:
            if sp.current_user_playing_track()['is_playing']: sp.pause_playback()
            else: sp.start_playback()
        except: pass


# --- PLAYLIST VIEW ---
class PlaylistListView(tk.Frame):
    def __init__(self, parent, controller):
        tk.Frame.__init__(self, parent, bg=BG_COLOR)
        self.controller = controller
        
        # Header
        hdr = tk.Frame(self, bg="#121212", height=40)
        hdr.pack(fill="x")
        tk.Button(hdr, text="⬅ Player", bg="#333", fg="white", bd=0, font=("Arial", 10),
                  command=lambda: controller.show_frame("PlayerView")).pack(side="left", padx=10, pady=5)
        tk.Label(hdr, text="La tua Libreria", bg="#121212", fg="white", font=("Arial", 12, "bold")).pack(side="left", padx=10)

        # Scroll Area
        self.canvas = tk.Canvas(self, bg=BG_COLOR, highlightthickness=0)
        self.scrollbar = tk.Scrollbar(self, orient="vertical", command=self.canvas.yview)
        self.scroll_frame = tk.Frame(self.canvas, bg=BG_COLOR)
        
        self.scroll_frame.bind("<Configure>", lambda e: self.canvas.configure(scrollregion=self.canvas.bbox("all")))
        self.canvas.create_window((0, 0), window=self.scroll_frame, anchor="nw", width=460)
        self.canvas.configure(yscrollcommand=self.scrollbar.set)
        
        self.canvas.pack(side="left", fill="both", expand=True)
        self.scrollbar.pack(side="right", fill="y")
        self.images_ref = [] 

    def load_playlists(self):
        for w in self.scroll_frame.winfo_children(): w.destroy()
        self.images_ref = []
        threading.Thread(target=self._fetch).start()

    def _fetch(self):
        try:
            pl_list = [{"id": "liked", "name": "Brani Preferiti", "img_url": None, "uri": None}]
            items = sp.current_user_playlists(limit=20)['items']
            for item in items:
                 img = item['images'][0]['url'] if item['images'] else None
                 pl_list.append({"id": item['id'], "name": item['name'], "img_url": img, "uri": item['uri']})
            
            self.after(0, self._populate, pl_list)
        except: pass

    def _populate(self, pl_list):
        for data in pl_list:
            row = tk.Frame(self.scroll_frame, bg=BG_COLOR, pady=5)
            row.pack(fill="x", expand=True)
            
            icon_cv = tk.Canvas(row, width=50, height=50, bg=BG_COLOR, highlightthickness=0)
            icon_cv.pack(side="left", padx=(0, 10))
            
            if data['id'] == 'liked':
                icon_cv.configure(bg=PURPLE_COLOR)
                icon_cv.create_text(25, 25, text="♥", fill="white", font=("Arial", 20))
            elif data['img_url']:
                photo = ImageLoader.get_image(data['img_url'], (50, 50), radius=5)
                if photo:
                    icon_cv.create_image(0, 0, image=photo, anchor="nw")
                    self.images_ref.append(photo)
            else:
                icon_cv.configure(bg="#333")

            tk.Label(row, text=data['name'], font=("Arial", 12), bg=BG_COLOR, fg="white", anchor="w").pack(side="left", fill="both", expand=True)

            # Il click apre la lista canzoni (TrackListView)
            for w in [row, icon_cv] + row.winfo_children():
                w.bind("<Button-1>", lambda e, d=data: self.controller.show_frame("TrackListView", d))
            
            tk.Frame(self.scroll_frame, height=1, bg="#333").pack(fill="x")


# --- TRACKLIST VIEW (Nuova Schermata: Lista Canzoni) ---
class TrackListView(tk.Frame):
    def __init__(self, parent, controller):
        tk.Frame.__init__(self, parent, bg=BG_COLOR)
        self.controller = controller
        
        # Header
        hdr = tk.Frame(self, bg="#121212", height=40)
        hdr.pack(fill="x")
        # Tasto Indietro
        tk.Button(hdr, text="⬅ Playlist", bg="#333", fg="white", bd=0, font=("Arial", 10),
                  command=lambda: controller.show_frame("PlaylistListView")).pack(side="left", padx=10, pady=5)
        
        self.lbl_title = tk.Label(hdr, text="Canzoni", bg="#121212", fg="white", font=("Arial", 12, "bold"))
        self.lbl_title.pack(side="left", padx=10)

        # Scroll Area
        self.canvas = tk.Canvas(self, bg=BG_COLOR, highlightthickness=0)
        self.scrollbar = tk.Scrollbar(self, orient="vertical", command=self.canvas.yview)
        self.scroll_frame = tk.Frame(self.canvas, bg=BG_COLOR)
        
        self.scroll_frame.bind("<Configure>", lambda e: self.canvas.configure(scrollregion=self.canvas.bbox("all")))
        self.canvas.create_window((0, 0), window=self.scroll_frame, anchor="nw", width=460)
        self.canvas.configure(yscrollcommand=self.scrollbar.set)
        
        self.canvas.pack(side="left", fill="both", expand=True)
        self.scrollbar.pack(side="right", fill="y")
        self.current_uris = []
        self.context_uri = None

    def load_tracks(self, playlist_data):
        self.lbl_title.config(text=playlist_data['name'][:25]) # Aggiorna titolo
        for w in self.scroll_frame.winfo_children(): w.destroy()
        self.current_uris = []
        self.context_uri = playlist_data.get('uri') # Può essere None per i Liked
        
        threading.Thread(target=self._fetch, args=(playlist_data,)).start()

    def _fetch(self, playlist_data):
        try:
            tracks_list = []
            
            # SE SONO I PREFERITI
            if playlist_data['id'] == 'liked':
                results = sp.current_user_saved_tracks(limit=50)
                for item in results['items']:
                    track = item['track']
                    tracks_list.append(track)
                    self.current_uris.append(track['uri'])
            
            # SE E' UNA PLAYLIST NORMALE
            else:
                results = sp.playlist_tracks(playlist_data['id'], limit=50)
                for item in results['items']:
                    track = item['track']
                    if track: # a volte è None
                        tracks_list.append(track)
                        self.current_uris.append(track['uri'])

            self.after(0, self._populate, tracks_list)
        except Exception as e:
            print("Err tracks:", e)

    def _populate(self, tracks_list):
        for i, track in enumerate(tracks_list):
            row = tk.Frame(self.scroll_frame, bg=BG_COLOR, pady=8)
            row.pack(fill="x", expand=True)
            
            # Numero
            tk.Label(row, text=str(i+1), font=("Arial", 10), bg=BG_COLOR, fg=GRAY_COLOR, width=3).pack(side="left")
            
            # Titolo e Artista
            info_frame = tk.Frame(row, bg=BG_COLOR)
            info_frame.pack(side="left", fill="both", expand=True)
            
            tk.Label(info_frame, text=track['name'], font=("Arial", 11, "bold"), bg=BG_COLOR, fg="white", anchor="w").pack(fill="x")
            tk.Label(info_frame, text=track['artists'][0]['name'], font=("Arial", 9), bg=BG_COLOR, fg=GRAY_COLOR, anchor="w").pack(fill="x")
            
            # Click per riprodurre
            for w in [row, info_frame] + info_frame.winfo_children():
                w.bind("<Button-1>", lambda e, uri=track['uri']: self.play_track(uri))
                
            tk.Frame(self.scroll_frame, height=1, bg="#222").pack(fill="x")

    def play_track(self, uri):
        try:
            # Se abbiamo un context_uri (Playlist normale), usiamo quello + offset
            if self.context_uri:
                sp.start_playback(context_uri=self.context_uri, offset={"uri": uri})
            else:
                # Per i PREFERITI (che non hanno context), proviamo a passare la lista
                try:
                    idx = self.current_uris.index(uri)
                    subset = self.current_uris[idx : idx+50]
                    sp.start_playback(uris=subset)
                except:
                    sp.start_playback(uris=[uri])

            self.controller.show_frame("PlayerView")
        except Exception as e:
            print("Play err:", e)

if __name__ == "__main__":
    root = tk.Tk()
    app = SpotifyCarUI(root)
    root.mainloop()

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
