import customtkinter as ctk
import threading
import subprocess
import time
import sys
import os
import requests
import webbrowser
from bs4 import BeautifulSoup
from tkinter import messagebox, filedialog

# --- CONFIGURATION ---
ADB_PATH = "adb"
FASTBOOT_PATH = "fastboot"
SAVE_NAME = "recovery.img"
APP_VERSION = "v1.0"

# --- DEVICE DATABASE ---
CODENAME_MAP = {
    "Pixel": "sailfish", "Pixel XL": "marlin",
    "Pixel 2": "walleye", "Pixel 2 XL": "taimen",
    "Pixel 3": "blueline", "Pixel 3 XL": "crosshatch",
    "Pixel 3a": "sargo", "Pixel 3a XL": "bonito",
    "Pixel 4": "flame", "Pixel 4 XL": "coral",
    "Pixel 4a": "sunfish", "Pixel 4a 5G": "bramble",
    "Pixel 5": "redfin", "Pixel 5a": "barbet",
    "Pixel 6": "oriole", "Pixel 6 Pro": "raven", "Pixel 6a": "bluejay",
    "Pixel 7": "panther", "Pixel 7 Pro": "cheetah", "Pixel 7a": "lynx",
    "Pixel 8": "shiba", "Pixel 8 Pro": "husky", "Pixel 8a": "akita",
    "Pixel Fold": "felix", "Pixel Tablet": "tangorpro",
    "OnePlus 3": "oneplus3", "OnePlus 3T": "oneplus3",
    "OnePlus 5": "cheeseburger", "OnePlus 5T": "dumpling",
    "OnePlus 6": "enchilada", "OnePlus A6000": "enchilada", "OnePlus A6003": "enchilada",
    "OnePlus 6T": "fajita", "OnePlus A6010": "fajita", "OnePlus A6013": "fajita",
    "OnePlus 7": "guacamoleb", "OnePlus GM1900": "guacamoleb", "OnePlus GM1901": "guacamoleb",
    "OnePlus 7 Pro": "guacamole", "OnePlus GM1910": "guacamole", "OnePlus GM1917": "guacamole",
    "OnePlus 7T": "hotdogb", "OnePlus 7T Pro": "hotdog",
    "OnePlus 8": "instantnoodle", "OnePlus 8 Pro": "instantnoodlep", "OnePlus 8T": "kebab",
    "OnePlus 9": "lemonade", "OnePlus 9 Pro": "lemonadep", "OnePlus 9R": "lemonades",
    "OnePlus 10 Pro": "neul", "OnePlus 10T": "ovaltine",
    "OnePlus 11": "salami", "OnePlus 11R": "udon",
    "OnePlus 12": "pineapple", "OnePlus 12R": "ace3",
    "OnePlus Nord": "avicii", "OnePlus Nord 2": "denniz", "OnePlus Nord 3": "vitamin",
    "POCO F1": "beryllium", "POCOF1": "beryllium",
    "POCO F2 Pro": "lmi",
    "POCO F3": "alioth", "POCO F4": "munch",
    "POCO F5": "marble", "POCO F5 Pro": "mondrian",
    "POCO F6": "peridot", "POCO F6 Pro": "vermeer",
    "POCO X3": "surya", "POCO X3 Pro": "vayu",
    "POCO X4 Pro": "veux", "POCO X5 Pro": "redwood", "POCO X6 Pro": "duchamp",
    "Redmi Note 7": "lavender", "Redmi Note 7 Pro": "violet",
    "Redmi Note 8": "ginkgo", "Redmi Note 8 Pro": "begonia",
    "Redmi Note 9S": "curtana", "Redmi Note 9 Pro": "joyeuse",
    "Redmi Note 10": "mojito", "Redmi Note 10 Pro": "sweet",
    "Redmi Note 11": "spes", "Redmi Note 12": "tapas",
    "Mi 9": "cepheus", "Mi 9T": "davinci", "Mi 9T Pro": "raphael",
    "Mi 10": "umi", "Mi 10 Pro": "cmi",
    "Mi 11": "venus", "Mi 11 Ultra": "star",
    "Nothing Phone (1)": "spacewar",
    "Nothing Phone (2)": "pong",
    "Nothing Phone (2a)": "pacman",
    "Galaxy S10": "beyond1lte", "Galaxy S10+": "beyond2lte",
    "Galaxy S20": "hubit", "Galaxy S20 FE": "r8q",
    "Galaxy S21": "o1s", "Galaxy S21 Ultra": "p3s",
    "Galaxy S22 Ultra": "b0s", "Galaxy S23 Ultra": "dm3q", "Galaxy S24 Ultra": "e3q",
    "Moto G7": "river", "Moto G7 Plus": "lake",
    "Moto G32": "devon", "Moto G42": "hawao",
    "Edge 30": "dubai", "Edge 40": "rte",
    "Zenfone 8": "sake", "Zenfone 9": "zenith", "Zenfone 10": "euphoria",
    "ROG Phone 3": "obiwan", "ROG Phone 5": "odin", "ROG Phone 6": "ai2201"
}

APP_NAME_MAP = {
    "com.android.chrome": "Google Chrome",
    "com.google.android.youtube": "YouTube",
    "com.google.android.apps.youtube.music": "YouTube Music",
    "com.google.android.gm": "Gmail",
    "com.google.android.apps.maps": "Google Maps",
    "com.miui.analytics": "MIUI Analytics (Spyware)",
    "com.miui.calculator": "Mi Calculator",
    "com.facebook.katana": "Facebook",
    "com.facebook.system": "Facebook App Installer (Bloat)",
    "com.netflix.mediaclient": "Netflix",
    "com.amazon.mShop.android.shopping": "Amazon Shopping",
    "com.microsoft.office.officehubrow": "Microsoft Office",
    "com.google.android.apps.photos": "Google Photos",
    "com.google.android.googlequicksearchbox": "Google App",
    "com.miui.cleanmaster": "Cleaner (Bloat)",
    "com.miui.msa.global": "MSA (Ads System)"
}


class SmartFlasherLogic:
    def __init__(self, log_callback, progress_callback):
        self.device_info = {}
        self.log = log_callback
        self.update_progress = progress_callback
        self.headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)'}

    def run_command(self, command, stream_output=False):
        try:
            if stream_output:
                return subprocess.Popen(command, shell=True, stdout=subprocess.PIPE, stderr=subprocess.STDOUT,
                                        text=True)
            result = subprocess.run(command, shell=True, capture_output=True, text=True,
                                    creationflags=subprocess.CREATE_NO_WINDOW)
            return result.stdout.strip()
        except:
            return None

    # --- DEVICE DETECTION ---
    def resolve_codename(self, raw_name):
        clean_name = raw_name.replace(" ", "").lower()
        for key, value in CODENAME_MAP.items():
            if key.replace(" ", "").lower() == clean_name: return value
        return raw_name.lower()

    def detect_device(self):
        self.log("Scout Agent: Scanning...")
        adb_out = self.run_command(f"{ADB_PATH} devices")

        if "sideload" in adb_out:
            self.device_info = {'state': 'sideload', 'codename': 'Unknown (Sideload)'}
            self.log("SUCCESS: Device in Sideload Mode.")
            return True

        if "device" in adb_out and "list" not in adb_out.splitlines()[-1].lower():
            try:
                model = self.run_command(f"{ADB_PATH} shell getprop ro.product.model").strip()
                device = self.run_command(f"{ADB_PATH} shell getprop ro.product.device").strip()
                manuf = self.run_command(f"{ADB_PATH} shell getprop ro.product.manufacturer").strip().lower()

                if "samsung" in manuf:
                    self.log("WARNING: Samsung Detected. Operation Aborted.")
                    messagebox.showwarning("Samsung Device", "Samsung devices require ODIN.\nUse official tools.")
                    return False

                final_code = self.resolve_codename(model)
                if final_code == model.lower().replace(" ", ""): final_code = self.resolve_codename(device)

                self.device_info = {'codename': final_code, 'state': 'adb'}
                self.log(f"SUCCESS: Connected (ADB): {final_code}")
                return True
            except:
                pass

        fastboot_out = self.run_command(f"{FASTBOOT_PATH} devices")
        if "fastboot" in fastboot_out:
            prod = self.run_command(f"{FASTBOOT_PATH} getvar product")
            if "product:" in prod:
                raw = prod.split("product:")[1].strip().split()[0]
                codename = self.resolve_codename(raw)
                self.device_info = {'codename': codename, 'state': 'fastboot'}
                self.log(f"SUCCESS: Connected (Fastboot): {codename}")
                return True

        self.log("FAIL: No device found.")
        return False

    # --- FLASHER LOGIC ---
    def download_recovery(self):
        codename = self.device_info.get('codename')
        if not codename: return False
        self.log(f"Web Agent: Searching TWRP for '{codename}'...")
        base_url = f"https://dl.twrp.me/{codename}/"
        try:
            response = requests.get(base_url, headers=self.headers)
            if response.status_code == 404:
                self.log("Error: Device not found on TWRP.me.")
                return False
            soup = BeautifulSoup(response.text, 'html.parser')
            links = soup.find_all('a')
            latest_url = next((l.get('href') for l in links if l.get('href', '').endswith('.img.html')), None)
            if not latest_url:
                self.log("Error: No .img file found.")
                return False
            file_path = latest_url.replace('.html', '')
            download_url = f"https://dl.twrp.me{file_path}"
            referer_url = f"https://dl.twrp.me{latest_url}"
            self.log(f"Downloading: {file_path.split('/')[-1]}")
            dl_headers = self.headers.copy()
            dl_headers['Referer'] = referer_url
            r = requests.get(download_url, headers=dl_headers, stream=True)
            total_size = int(r.headers.get('content-length', 0))
            with open(SAVE_NAME, 'wb') as f:
                downloaded = 0
                for chunk in r.iter_content(chunk_size=8192):
                    f.write(chunk)
                    downloaded += len(chunk)
                    if total_size > 0: self.update_progress(downloaded / total_size)
            self.log("Download Complete.")
            return True
        except Exception as e:
            self.log(f"Download Error: {e}")
            return False

    def open_alt_recovery(self, mode):
        code = self.device_info.get('codename')
        if not code:
            messagebox.showerror("Error", "Detect device first!")
            return
        self.log(f"Opening browser for {mode}...")
        if "OrangeFox" in mode:
            webbrowser.open(f"https://orangefox.download/device/{code}")
        elif "Lineage" in mode:
            webbrowser.open(f"https://wiki.lineageos.org/devices/{code}/install")
        elif "Pixel" in mode:
            webbrowser.open(f"https://pixelos.net/download/{code}")

    def reboot_bootloader(self):
        self.log("Rebooting to Bootloader...")
        self.run_command(f"{ADB_PATH} reboot bootloader")
        time.sleep(5)

    def flash_recovery_boot(self):
        if not os.path.exists(SAVE_NAME):
            self.log("Error: No recovery.img found.")
            return
        self.log("Sending TWRP to device RAM...")
        self.run_command(f"{FASTBOOT_PATH} boot {SAVE_NAME}")
        self.log("Command Sent.")

    def sideload_file(self, file_path):
        name = os.path.basename(file_path)
        self.log(f"Sideload Agent: Installing '{name}'...")
        check = self.run_command(f"{ADB_PATH} devices")
        if "sideload" not in check:
            self.log("ERROR: Device not in Sideload mode.")
            return
        process = self.run_command(f"{ADB_PATH} sideload \"{file_path}\"", stream_output=True)
        for line in process.stdout:
            if "%" in line:
                try:
                    self.update_progress(int(line.split("~")[1].split("%")[0]) / 100)
                except:
                    pass
            print(line.strip())
        self.log(f"Finished installing {name}.")

    def open_rom_hub(self):
        code = self.device_info.get('codename')
        if not code: return
        self.log(f"Opening ROM pages for {code}...")
        webbrowser.open(f"https://wiki.lineageos.org/devices/{code}/")
        webbrowser.open(f"https://crdroid.net/{code}")
        webbrowser.open(f"https://get.pixelexperience.org/{code}")
        webbrowser.open(f"https://www.google.com/search?q=site:xda-developers.com+{code}+custom+rom")

    def download_magisk(self):
        self.log("Magisk Agent: Contacting GitHub...")
        try:
            url = "https://api.github.com/repos/topjohnwu/Magisk/releases/latest"
            data = requests.get(url, headers=self.headers).json()
            apk_url = next((a['browser_download_url'] for a in data['assets'] if a['name'].endswith('.apk')), None)
            if apk_url:
                r = requests.get(apk_url, stream=True)
                filename = "Magisk_Latest.zip"
                with open(filename, 'wb') as f:
                    for chunk in r.iter_content(chunk_size=8192): f.write(chunk)
                self.log(f"Success! Saved as {filename}")
                return filename
        except Exception as e:
            self.log(f"Magisk Error: {e}")

    def open_gapps(self):
        webbrowser.open("https://nikgapps.com/downloads")

    def get_readable_name(self, package_name):
        if package_name in APP_NAME_MAP: return APP_NAME_MAP[package_name]
        try:
            return package_name.split('.')[-1].capitalize()
        except:
            return package_name

    def get_packages(self, filter_text=""):
        self.log(f"Debloater: Scanning...")
        cmd = f"{ADB_PATH} shell pm list packages"
        if filter_text: cmd += f" | grep {filter_text}"
        result = self.run_command(cmd)
        if not result: return []
        packages = []
        for line in result.splitlines():
            if "package:" in line: packages.append(line.split("package:")[1].strip())
        return packages

    def uninstall_package(self, package_name):
        self.log(f"Nuking {package_name}...")
        res = self.run_command(f"{ADB_PATH} shell pm uninstall --user 0 {package_name}")
        if "Success" in res:
            self.log("SUCCESS.")
        else:
            self.log("FAIL.")

    def reinstall_package(self, package_name):
        self.log(f"Restoring {package_name}...")
        self.run_command(f"{ADB_PATH} shell cmd package install-existing {package_name}")

    def unlock_bootloader(self):
        self.log("Warning: Unlocking Bootloader...")
        self.run_command(f"{FASTBOOT_PATH} flashing unlock")
        self.run_command(f"{FASTBOOT_PATH} oem unlock")
        self.log("Action Sent. CHECK PHONE SCREEN to confirm.")

    def lock_bootloader(self):
        self.log("Warning: Relocking Bootloader...")
        self.run_command(f"{FASTBOOT_PATH} flashing lock")
        self.run_command(f"{FASTBOOT_PATH} oem lock")
        self.log("Action Sent. CHECK PHONE SCREEN to confirm.")


# --- GUI FRONTEND ---
ctk.set_appearance_mode("Dark")
ctk.set_default_color_theme("blue")


class App(ctk.CTk):
    def __init__(self):
        super().__init__()
        self.title(f"DroidForge {APP_VERSION} ")

        self.show_disclaimer()

        # Start Maximized
        self.after(0, lambda: self.state('zoomed'))

        self.logic = SmartFlasherLogic(self.log_msg, self.set_progress)
        self.rom_path = None
        self.addon_path = None
        self.selected_package = None

        self.grid_columnconfigure(0, weight=1)
        self.grid_rowconfigure(1, weight=1)

        self.header = ctk.CTkFrame(self)
        self.header.grid(row=0, column=0, sticky="ew", padx=20, pady=10)
        ctk.CTkLabel(self.header, text="DROID FORGE", font=("Roboto", 22, "bold")).pack(side="left", padx=20)
        self.lbl_status = ctk.CTkLabel(self.header, text="Status: Idle", text_color="gray")
        self.lbl_status.pack(side="right", padx=20)

        self.tabs = ctk.CTkTabview(self)
        self.tabs.grid(row=1, column=0, padx=20, pady=10, sticky="nsew")

        # TAB 1: RECOVERY
        self.tab_rec = self.tabs.add("Recovery")
        self.frame_rec_actions = ctk.CTkFrame(self.tab_rec)
        self.frame_rec_actions.pack(pady=10, fill="x", padx=20)

        ctk.CTkLabel(self.frame_rec_actions, text="DEVICE TOOLS", font=("Arial", 12, "bold")).pack(pady=5)
        ctk.CTkButton(self.frame_rec_actions, text="1. Detect Device", command=self.start_scan).pack(pady=5)
        self.btn_reboot = ctk.CTkButton(self.frame_rec_actions, text="2. Reboot to Bootloader", state="disabled",
                                        fg_color="#E67E22", command=self.start_reboot_bl)
        self.btn_reboot.pack(pady=5)

        ctk.CTkLabel(self.frame_rec_actions, text="Step 3: Select Recovery:", text_color="gray").pack()
        self.opt_recovery = ctk.CTkOptionMenu(self.frame_rec_actions,
                                              values=["TWRP (Auto)", "OrangeFox (Web)", "Lineage Recovery (Web)",
                                                      "Pixel Recovery (Web)"])
        self.opt_recovery.pack(pady=5)

        self.btn_dl_twrp = ctk.CTkButton(self.frame_rec_actions, text="Download / Find", state="disabled",
                                         command=self.start_download_logic)
        self.btn_dl_twrp.pack(pady=5)

        self.btn_flash = ctk.CTkButton(self.frame_rec_actions, text="4. Flash / Boot Selected .img", state="disabled",
                                       fg_color="red", command=self.start_boot)
        self.btn_flash.pack(pady=5)

        ctk.CTkLabel(self.frame_rec_actions, text="BOOTLOADER CONTROL", font=("Arial", 12, "bold")).pack(pady=(15, 5))
        self.btn_unlock = ctk.CTkButton(self.frame_rec_actions, text="UNLOCK BOOTLOADER", state="disabled",
                                        fg_color="#D35400", command=self.start_unlock)
        self.btn_unlock.pack(pady=5)
        self.btn_lock = ctk.CTkButton(self.frame_rec_actions, text="RELOCK BOOTLOADER", state="disabled",
                                      fg_color="#7F8C8D", command=self.start_lock)
        self.btn_lock.pack(pady=5)

        ctk.CTkFrame(self.tab_rec, height=2, fg_color="gray").pack(fill="x", padx=40, pady=10)
        self.btn_roms = ctk.CTkButton(self.tab_rec, text="Find Compatible ROMs (Web)", state="disabled",
                                      fg_color="#8E44AD", command=self.logic.open_rom_hub)
        self.btn_roms.pack(pady=10)

        # TAB 2: FLASH ROM
        self.tab_rom = self.tabs.add("Flash ROM")
        ctk.CTkLabel(self.tab_rom, text="Step 1: Boot into TWRP Recovery", font=("Arial", 12, "bold")).pack(pady=(5, 0))
        self.frame_warn = ctk.CTkFrame(self.tab_rom, fg_color="#3B0000", border_color="red", border_width=1)
        self.frame_warn.pack(pady=5, fill="x", padx=40)
        ctk.CTkLabel(self.frame_warn, text="Step 2: WIPE DATA (Crucial!)", text_color="#FF5555",
                     font=("Arial", 12, "bold")).pack()
        ctk.CTkLabel(self.frame_warn, text="On Phone: Wipe -> Format Data -> Type 'yes'", text_color="#FFaaaa").pack()
        ctk.CTkLabel(self.tab_rom, text="Step 3: On Phone, select 'Advanced -> ADB Sideload'",
                     font=("Arial", 12, "bold")).pack(pady=(10, 0))
        self.frame_rom_file = ctk.CTkFrame(self.tab_rom)
        self.frame_rom_file.pack(fill="x", padx=20, pady=10)
        ctk.CTkButton(self.frame_rom_file, text="Step 4: Select ROM (.zip)", command=self.select_rom).pack(side="left",
                                                                                                           padx=10)
        self.lbl_rom = ctk.CTkLabel(self.frame_rom_file, text="No file selected")
        self.lbl_rom.pack(side="left", padx=10)
        ctk.CTkButton(self.tab_rom, text="Step 5: FLASH ROM NOW", fg_color="green", height=40,
                      command=self.start_flash_rom).pack(pady=5, fill="x", padx=40)
        ctk.CTkFrame(self.tab_rom, height=2, fg_color="gray").pack(fill="x", padx=20, pady=10)
        ctk.CTkLabel(self.tab_rom, text="EXTRAS (Post-Install)", font=("Arial", 12, "bold")).pack()
        self.frame_extras = ctk.CTkFrame(self.tab_rom)
        self.frame_extras.pack(pady=5)
        ctk.CTkButton(self.frame_extras, text="Get Magisk", width=120, command=self.start_get_magisk).grid(row=0,
                                                                                                           column=0,
                                                                                                           padx=5,
                                                                                                           pady=5)
        ctk.CTkButton(self.frame_extras, text="GApps Guide", width=120, fg_color="#2980B9",
                      command=self.open_gapps_window).grid(row=0, column=1, padx=5, pady=5)
        ctk.CTkButton(self.frame_extras, text="Select Add-on", width=120, command=self.select_addon).grid(row=1,
                                                                                                          column=0,
                                                                                                          padx=5,
                                                                                                          pady=5)
        ctk.CTkButton(self.frame_extras, text="Flash Add-on", width=120, fg_color="#E67E22",
                      command=self.start_flash_addon).grid(row=1, column=1, padx=5, pady=5)
        self.lbl_addon = ctk.CTkLabel(self.tab_rom, text="Addon: None", text_color="gray")
        self.lbl_addon.pack()

        # TAB 3: DEBLOATER
        self.tab_bloat = self.tabs.add("Debloater")
        self.frame_search = ctk.CTkFrame(self.tab_bloat)
        self.frame_search.pack(fill="x", padx=10, pady=10)
        self.entry_search = ctk.CTkEntry(self.frame_search, placeholder_text="Search (e.g. google, facebook)")
        self.entry_search.pack(side="left", fill="x", expand=True, padx=5)
        ctk.CTkButton(self.frame_search, text="Scan", width=80, command=self.start_scan_apps).pack(side="right", padx=5)
        self.scroll_apps = ctk.CTkScrollableFrame(self.tab_bloat, height=150)
        self.scroll_apps.pack(fill="both", expand=True, padx=10, pady=5)
        self.frame_actions = ctk.CTkFrame(self.tab_bloat)
        self.frame_actions.pack(fill="x", padx=10, pady=10)
        self.lbl_selected = ctk.CTkLabel(self.frame_actions, text="Selected: None", text_color="yellow")
        self.lbl_selected.pack(pady=5)
        ctk.CTkButton(self.frame_actions, text="UNINSTALL APP", fg_color="red", command=self.nuke_app).pack(side="left",
                                                                                                            padx=20,
                                                                                                            expand=True)
        ctk.CTkButton(self.frame_actions, text="RESTORE APP", fg_color="green", command=self.restore_app).pack(
            side="right", padx=20, expand=True)

        # TAB 4: ABOUT
        self.tab_about = self.tabs.add("About")
        ctk.CTkLabel(self.tab_about, text="DROID FORGE", font=("Roboto", 30, "bold")).pack(pady=(40, 10))
        ctk.CTkLabel(self.tab_about, text=f"Version: {APP_VERSION}", font=("Arial", 16)).pack(pady=5)

        desc = ("The Ultimate Android Modding Toolkit.\n"
                "Automated Recovery Flashing, ROM Sideloading,\n"
                "Rooting, and Bloatware Removal.\n\n"
                "Designed for Enthusiasts.")
        ctk.CTkLabel(self.tab_about, text=desc, font=("Arial", 14), justify="center").pack(pady=20)

        ctk.CTkLabel(self.tab_about, text="DISCLAIMER", text_color="red", font=("Arial", 14, "bold")).pack(pady=(20, 5))
        warn = ("Modifying system partitions involves risk.\n"
                "The developers are not responsible for bricked devices,\n"
                "dead SD cards, or thermonuclear war.")
        ctk.CTkLabel(self.tab_about, text=warn, text_color="#FF8888").pack()

        ctk.CTkButton(self.tab_about, text="Visit Website", fg_color="transparent", border_width=1,
                      command=lambda: webbrowser.open("https://github.com/prashanthyadav1991")).pack(pady=30)

        self.console = ctk.CTkTextbox(self, height=100)
        self.console.grid(row=2, column=0, padx=20, pady=10, sticky="ew")
        self.progress = ctk.CTkProgressBar(self)
        self.progress.grid(row=3, column=0, padx=20, pady=10, sticky="ew")
        self.progress.set(0)

    def show_disclaimer(self):
        warning_text = (
            "DISCLAIMER & LIABILITY WAIVER\n\n"
            "You are about to modify your device's system software.\n"
            "This process involves risks including, but not limited to:\n\n"
            "• Permanent damage to the device (Bricking)\n"
            "• Loss of all data (Photos, Contacts, etc.)\n"
            "• Voiding of your manufacturer warranty\n\n"
            "By clicking 'I Agree', you acknowledge that DroidForge and its developers\n"
            "are NOT responsible for any damage caused to your device.\n\n"
            "You proceed at your own risk.\n\n"
            "Make sure you have enabled usb-debugging before continuing.\n\n"
        )
        if not messagebox.askyesno("WARNING", warning_text, icon='warning'): sys.exit()

    def log_msg(self, msg):
        self.console.insert("end", f"\n[{time.strftime('%H:%M')}] {msg}")
        self.console.see("end")

    def set_progress(self, val):
        self.progress.set(val)

    def start_scan(self):
        threading.Thread(target=self.scan_thread).start()

    def scan_thread(self):
        if self.logic.detect_device():
            self.lbl_status.configure(text=f"Connected: {self.logic.device_info['codename']}", text_color="#00FF00")
            self.btn_roms.configure(state="normal")
            self.btn_dl_twrp.configure(state="normal")
            self.btn_reboot.configure(state="normal")
            self.btn_flash.configure(state="normal")
            self.btn_unlock.configure(state="normal")
            self.btn_lock.configure(state="normal")
        else:
            self.lbl_status.configure(text="Disconnected", text_color="red")

    def start_download_logic(self):
        mode = self.opt_recovery.get()
        if "TWRP" in mode:
            threading.Thread(target=self.logic.download_recovery).start()
        else:
            self.logic.open_alt_recovery(mode)

    def start_reboot_bl(self):
        threading.Thread(target=self.logic.reboot_bootloader).start()

    def start_boot(self):
        threading.Thread(target=self.logic.flash_recovery_boot).start()

    def start_unlock(self):
        if messagebox.askyesno("UNLOCK WARNING",
                               "Unlocking Bootloader will WIPE ALL DATA.\n\nXiaomi users: Use official Mi Unlock Tool.\n\nAre you sure you want to proceed?"):
            threading.Thread(target=self.logic.unlock_bootloader).start()

    def start_lock(self):
        if messagebox.askyesno("LOCK WARNING",
                               "CRITICAL WARNING:\n\nDo NOT lock bootloader if you have a Custom ROM installed.\nIt will BRICK your device.\n\nOnly lock if you are on 100% Stock ROM.\n\nProceed?"):
            threading.Thread(target=self.logic.lock_bootloader).start()

    def select_rom(self):
        f = filedialog.askopenfilename(filetypes=[("Zip", "*.zip")])
        if f:
            self.rom_path = f
            self.lbl_rom.configure(text=os.path.basename(f))

    def start_flash_rom(self):
        if self.rom_path:
            if messagebox.askyesno("Safety Check",
                                   "Did you complete Step 2 (Format Data)?\nIf not, your phone will bootloop."):
                threading.Thread(target=self.logic.sideload_file, args=(self.rom_path,)).start()

    def start_get_magisk(self):
        threading.Thread(target=self.run_magisk_dl).start()

    def run_magisk_dl(self):
        f = self.logic.download_magisk()
        if f:
            self.addon_path = f
            self.lbl_addon.configure(text=f"Ready: {f}")
            messagebox.showinfo("Magisk", "Downloaded! Click 'Flash Add-on' to install.")

    def select_addon(self):
        f = filedialog.askopenfilename(filetypes=[("Zip", "*.zip")])
        if f:
            self.addon_path = f
            self.lbl_addon.configure(text=f"Ready: {os.path.basename(f)}")

    def start_flash_addon(self):
        if self.addon_path: threading.Thread(target=self.logic.sideload_file, args=(self.addon_path,)).start()

    def open_gapps_window(self):
        window = ctk.CTkToplevel(self)
        window.title("Google Apps Guide")
        window.geometry("500x400")
        window.attributes("-topmost", True)
        ctk.CTkLabel(window, text="How to Install GApps", font=("Arial", 18, "bold")).pack(pady=10)
        instructions = (
            "1. Most Custom ROMs do NOT include Google Apps.\n"
            "2. You must flash the ROM first.\n\n"
            "CRITICAL STEP (A/B Devices like Pixel/OnePlus):\n"
            "   After flashing ROM, you MUST 'Reboot to Recovery'\n"
            "   before flashing GApps. This switches the slot.\n\n"
            "3. If you boot to System without GApps, you cannot\n"
            "   add them later without Wiping Data again."
        )
        ctk.CTkLabel(window, text=instructions, justify="left", padx=20).pack(pady=10)
        ctk.CTkButton(window, text="Open Download Page (NikGapps)", fg_color="#2980B9",
                      command=self.logic.open_gapps).pack(pady=20)
        ctk.CTkButton(window, text="Close", fg_color="gray", command=window.destroy).pack(pady=10)

    def start_scan_apps(self):
        for w in self.scroll_apps.winfo_children(): w.destroy()
        threading.Thread(target=self.run_app_scan, args=(self.entry_search.get(),)).start()

    def run_app_scan(self, term):
        packages = self.logic.get_packages(term)
        if not packages: self.log_msg("No apps found.")
        for pkg in packages:
            human_name = self.logic.get_readable_name(pkg)
            display_text = f"{human_name}  ({pkg})"
            ctk.CTkButton(self.scroll_apps, text=display_text, fg_color="transparent", border_width=1, anchor="w",
                          command=lambda p=pkg: self.select_app(p)).pack(fill="x", pady=2)

    def select_app(self, app):
        self.selected_package = app
        name = self.logic.get_readable_name(app)
        self.lbl_selected.configure(text=f"Selected: {name}")

    def nuke_app(self):
        if self.selected_package:
            name = self.logic.get_readable_name(self.selected_package)
            if messagebox.askyesno("Nuke", f"Uninstall {name}?"):
                threading.Thread(target=self.logic.uninstall_package, args=(self.selected_package,)).start()

    def restore_app(self):
        if self.selected_package:
            threading.Thread(target=self.logic.reinstall_package, args=(self.selected_package,)).start()


if __name__ == "__main__":
    app = App()
    app.mainloop()
