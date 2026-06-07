import os
import sys
import subprocess
from threading import Thread
import customtkinter as ctk
from PIL import Image
import minecraft_launcher_lib

# Klasör yolları ve Minecraft ana dizini
MINECRAFT_DIR = os.path.join(os.getenv("APPDATA"), ".minecraft")

class YunusLauncherSmoothPremium(ctk.CTk):
    def __init__(self):
        super().__init__()

        # Pencere Ayarları - BÜYÜTÜLEBİLİR VE ESNEK
        self.title("YunusLauncher v3.5 - Smooth & Scalable Edition")
        self.geometry("980x620")
        self.minsize(920, 590)
        self.resizable(True, True)
        
        # Minecraft Teması & Pürüzsüz Büyük Font Ayarları
        ctk.set_appearance_mode("dark")
        self.smooth_font_sm = ("Segoe UI", 13, "bold")      # Küçük yazılar büyütüldü ve yumuşatıldı
        self.smooth_font = ("Segoe UI", 15, "bold")         # Ana yazılar büyütüldü ve yumuşatıldı
        self.title_font = ("Segoe UI", 22, "bold")          # Başlık pürüzsüzleştirildi

        # Ana Grid Yapısı: Sol taraf menü (0), Sağ taraf ana içerik (1)
        self.grid_columnconfigure(1, weight=1)
        self.grid_rowconfigure(0, weight=1)

        # =================================================================
        # 1. SOL MENÜ PANELİ (Sidebar)
        # =================================================================
        self.sidebar = ctk.CTkFrame(self, width=200, fg_color="#111111", corner_radius=0, border_width=0)
        self.sidebar.grid(row=0, column=0, sticky="nws")
        self.sidebar.grid_propagate(False)

        self.sb_title = ctk.CTkLabel(self.sidebar, text="MINECRAFT\nJAVA EDITION", font=self.smooth_font, text_color="#2ecc71")
        self.sb_title.pack(pady=(25, 35), padx=10)

        self.btn_news = ctk.CTkButton(self.sidebar, text="📰  Haberler", font=self.smooth_font, anchor="w", fg_color="transparent", hover_color="#222222", corner_radius=4, height=45)
        self.btn_news.pack(fill="x", padx=8, pady=3)

        self.btn_mod_folder = ctk.CTkButton(self.sidebar, text="📂  Mod Klasörü", font=self.smooth_font, anchor="w", fg_color="transparent", hover_color="#222222", corner_radius=4, height=45, command=self.open_mods_folder)
        self.btn_mod_folder.pack(fill="x", padx=8, pady=3)

        self.btn_settings = ctk.CTkButton(self.sidebar, text="⚙️  Ayarlar", font=self.smooth_font, anchor="w", fg_color="transparent", hover_color="#222222", corner_radius=4, height=45)
        self.btn_settings.pack(fill="x", padx=8, pady=3)

        # =================================================================
        # 2. SAĞ ANA İÇERİK ALANI (Görsel ve Alt Oynatma Barı)
        # =================================================================
        self.main_area = ctk.CTkFrame(self, fg_color="#1a1a1a", corner_radius=0)
        self.main_area.grid(row=0, column=1, sticky="nsew")
        self.main_area.grid_columnconfigure(0, weight=1)
        self.main_area.grid_rowconfigure(0, weight=1) # Görsel alanı esnek
        self.main_area.grid_rowconfigure(1, weight=0) # Alt bar sabit

        # Arka Plan Görsel Taşıyıcısı
        self.bg_label = ctk.CTkLabel(self.main_area, text="")
        self.bg_label.grid(row=0, column=0, sticky="nsew")

        # Görseli Yükle ve Dinamik Ölçeklendir
        bg_image_path = os.path.join(os.path.dirname(__file__), "arka_plan.png")
        if os.path.exists(bg_image_path):
            self.raw_bg = Image.open(bg_image_path)
            self.bind("<Configure>", self.update_background_size)

        # =================================================================
        # 3. ALT OYNATMA BARI (Gelişmiş Kontrol Paneli)
        # =================================================================
        self.bottom_bar = ctk.CTkFrame(self.main_area, height=130, fg_color="#1e1e1e", corner_radius=0, border_width=1, border_color="#2d2d2d")
        self.bottom_bar.grid(row=1, column=0, sticky="ew")
        self.bottom_bar.grid_propagate(False)
        
        # Elemanların tam sığması için Alt Bar Grid esneklikleri ayarlandı
        self.bottom_bar.grid_columnconfigure(0, weight=1) 
        self.bottom_bar.grid_columnconfigure(1, weight=1) 
        self.bottom_bar.grid_columnconfigure(2, weight=1) 

        # --- SOL KISIM: Kullanıcı Adı ve Sürüm Menüsü ---
        self.left_controls = ctk.CTkFrame(self.bottom_bar, fg_color="transparent")
        self.left_controls.grid(row=0, column=0, padx=20, pady=15, sticky="w")

        self.user_entry = ctk.CTkEntry(self.left_controls, placeholder_text="Kullanıcı Adı", width=160, height=35, font=self.smooth_font_sm, corner_radius=4, border_color="#4a4a4a")
        self.user_entry.grid(row=0, column=0, padx=5, pady=5)
        self.user_entry.insert(0, "YunusEmre")

        self.version_list = self.get_all_versions()
        self.version_dropdown = ctk.CTkOptionMenu(
            self.left_controls, width=160, height=35, font=self.smooth_font_sm, corner_radius=4,
            values=self.version_list, fg_color="#333333", button_color="#444444", button_hover_color="#2ecc71"
        )
        self.version_dropdown.grid(row=0, column=1, padx=5, pady=5)

        self.progress_label = ctk.CTkLabel(self.left_controls, text="Oynamaya Hazır", font=("Segoe UI", 12, "italic"), text_color="#2ecc71", width=330, anchor="w")
        self.progress_label.grid(row=1, column=0, columnspan=2, padx=5, pady=5)

        # --- ORTA KISIM: Eklenti Motoru (Fabric, Forge vb.) ---
        self.mid_controls = ctk.CTkFrame(self.bottom_bar, fg_color="#262626", corner_radius=6, border_width=1, border_color="#3a3a3a")
        self.mid_controls.grid(row=0, column=1, padx=10, pady=15, sticky="ew")

        self.mod_title = ctk.CTkLabel(self.mid_controls, text="Eklenti Motoru (Oto İndir)", font=self.smooth_font_sm, text_color="#e74c3c")
        self.mod_title.grid(row=0, column=0, columnspan=4, pady=4)

        # Butonlar daha pürüzsüz ve büyük hale getirildi
        self.btn_fabric = ctk.CTkButton(self.mid_controls, text="Fabric", font=self.smooth_font_sm, width=75, height=28, corner_radius=4, fg_color="#34495e", hover_color="#2ecc71", command=lambda: self.start_installer("fabric"))
        self.btn_fabric.grid(row=1, column=0, padx=4, pady=4)

        self.btn_forge = ctk.CTkButton(self.mid_controls, text="Forge", font=self.smooth_font_sm, width=75, height=28, corner_radius=4, fg_color="#34495e", hover_color="#2ecc71", command=lambda: self.start_installer("forge"))
        self.btn_forge.grid(row=1, column=1, padx=4, pady=4)

        self.btn_quilt = ctk.CTkButton(self.mid_controls, text="Quilt", font=self.smooth_font_sm, width=75, height=28, corner_radius=4, fg_color="#34495e", hover_color="#2ecc71", command=lambda: self.start_installer("quilt"))
        self.btn_quilt.grid(row=1, column=2, padx=4, pady=4)

        self.btn_neoforge = ctk.CTkButton(self.mid_controls, text="Neo", font=self.smooth_font_sm, width=65, height=28, corner_radius=4, fg_color="#34495e", hover_color="#2ecc71", command=lambda: self.start_installer("neoforge"))
        self.btn_neoforge.grid(row=1, column=3, padx=4, pady=4)

        # --- SAĞ KISIM: Dev OYUNA GİR (PLAY) Butonu ve Progress Bar ---
        self.right_controls = ctk.CTkFrame(self.bottom_bar, fg_color="transparent")
        self.right_controls.grid(row=0, column=2, padx=25, pady=15, sticky="e")

        self.progress_bar = ctk.CTkProgressBar(self.right_controls, width=200, height=8, corner_radius=4, progress_color="#2ecc71")
        self.progress_bar.set(0)
        self.progress_bar.grid(row=0, column=0, pady=(0, 8))

        self.play_button = ctk.CTkButton(
            self.right_controls, text="PLAY", font=("Segoe UI", 20, "bold"),
            width=200, height=50, corner_radius=4, fg_color="#27ae60", hover_color="#2ecc71",
            command=self.start_launch_thread
        )
        self.play_button.grid(row=1, column=0, pady=(0, 2))

    def update_background_size(self, event):
        """Pencere boyutu tam ekran veya manuel büyütüldükçe resmi kusursuz sığdırır"""
        if hasattr(self, 'raw_bg'):
            width = self.main_area.winfo_width()
            height = self.main_area.winfo_height() - 130 # Alt bar payı düşülüyor
            if width > 10 and height > 10:
                ctk_img = ctk.CTkImage(light_image=self.raw_bg, dark_image=self.raw_bg, size=(width, height))
                self.bg_label.configure(image=ctk_img)

    def get_all_versions(self):
        try:
            versions = [v["id"] for v in minecraft_launcher_lib.utils.get_version_list()]
            if os.path.exists(os.path.join(MINECRAFT_DIR, "versions")):
                local_versions = os.listdir(os.path.join(MINECRAFT_DIR, "versions"))
                for lv in local_versions:
                    if lv not in versions:
                        versions.insert(0, lv)
            return versions
        except:
            return ["1.21.4", "1.20.1", "1.16.5", "1.8.9"]

    def open_mods_folder(self):
        mods_path = os.path.join(MINECRAFT_DIR, "mods")
        if not os.path.exists(mods_path):
            os.makedirs(mods_path)
        os.startfile(mods_path)

    def start_installer(self, loader_type):
        self.progress_label.configure(text=f"{loader_type.upper()} Hazırlanıyor...", text_color="#f1c40f")
        Thread(target=self.install_loader_logic, args=(loader_type,), daemon=True).start()

    def install_loader_logic(self, loader_type):
        current_version = self.version_dropdown.get()
        self.progress_bar.set(0.3)
        try:
            if loader_type == "fabric":
                minecraft_launcher_lib.fabric.install_fabric(current_version, MINECRAFT_DIR)
            elif loader_type == "forge":
                minecraft_launcher_lib.forge.install_forge_version(current_version, MINECRAFT_DIR)
            elif loader_type == "quilt":
                minecraft_launcher_lib.quilt.install_quilt(current_version, MINECRAFT_DIR)
            elif loader_type == "neoforge":
                os.system(f"start https://neoforge.net/")
            
            self.progress_bar.set(1)
            self.progress_label.configure(text=f"{loader_type.upper()} Başarıyla Kuruldu!", text_color="#2ecc71")
            self.version_dropdown.configure(values=self.get_all_versions())
        except Exception as e:
            self.progress_label.configure(text="Hata! Bu sürüm desteklenmiyor olabilir.", text_color="#e74c3c")
            self.progress_bar.set(0)

    def start_launch_thread(self):
        self.play_button.configure(state="disabled")
        self.progress_label.configure(text="Dosyalar kontrol ediliyor...", text_color="#f1c40f")
        Thread(target=self.launch_minecraft, daemon=True).start()

    def launch_minecraft(self):
        username = self.user_entry.get()
        selected_version = self.version_dropdown.get()

        if not username.strip():
            self.progress_label.configure(text="Hata: Kullanıcı adı boş olamaz!", text_color="#e74c3c")
            self.play_button.configure(state="normal")
            return

        try:
            self.progress_bar.set(0.4)
            callback = {
                "setStatus": lambda text: self.progress_label.configure(text=text),
                "setProgress": lambda progress: self.progress_bar.set(int(progress) / 100)
            }
            
            minecraft_launcher_lib.install.install_minecraft_version(selected_version, MINECRAFT_DIR, callback=callback)
            
            options = {"username": username, "uuid": "", "token": ""}
            launch_command = minecraft_launcher_lib.command.get_minecraft_command(selected_version, MINECRAFT_DIR, options)
            
            self.progress_label.configure(text="Oyun açılıyor, lütfen bekleyin...", text_color="#2ecc71")
            self.progress_bar.set(1)
            self.withdraw() # Launcher penceresini gizle

            # SİYAH CMD/JAVA EKRANINI GİZLEME AYARI
            startupinfo = subprocess.STARTUPINFO()
            startupinfo.dwFlags |= subprocess.STARTF_USESHOWWINDOW
            
            subprocess.run(launch_command, startupinfo=startupinfo)
            
            # Oyun kapatılınca geri yükle
            self.deiconify()
            self.progress_label.configure(text="Oynamaya Hazır!", text_color="#2ecc71")
            self.progress_bar.set(0)
            self.play_button.configure(state="normal")

        except Exception as e:
            self.deiconify()
            self.progress_label.configure(text="Bir hata oluştu!", text_color="#e74c3c")
            self.play_button.configure(state="normal")

if __name__ == "__main__":
    app = YunusLauncherSmoothPremium()
    app.mainloop()
