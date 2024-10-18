# youtube-playlist-downloader
A Python-based YouTube Playlist Downloader with GUI using Tkinter and yt-dlp.



import os
import tkinter as tk
from tkinter import ttk, messagebox, filedialog, scrolledtext
import yt_dlp
import threading
from queue import Queue
import re

class YouTubePlaylistDownloader:
    def __init__(self, master):
        self.master = master
        self.master.title("YouTube Playlist Downloader")
        self.master.geometry("900x700")
        
        # Initialize Variables
        self.download_folder = ""
        self.is_multi_download = False
        self.max_concurrent_downloads = 5
        self.download_queue = Queue()
        self.active_downloads = 0
        self.stop_event = threading.Event()
        self.song_entries = []  # List to store (title, url) tuples
        
        # URL Entry
        self.label_url = tk.Label(master, text="Çalma Listesi URL'si:")
        self.label_url.pack(pady=5)
        self.entry_url = tk.Entry(master, width=100)
        self.entry_url.pack(pady=5)
        
        # Format Selection
        self.format_var = tk.StringVar(value="mp3")
        self.frame_format = tk.Frame(master)
        self.frame_format.pack(pady=5)
        self.radio_mp3 = tk.Radiobutton(self.frame_format, text="MP3 (320 kbps)", variable=self.format_var, value="mp3")
        self.radio_mp4 = tk.Radiobutton(self.frame_format, text="MP4 (1080p)", variable=self.format_var, value="mp4")
        self.radio_mp3.pack(side=tk.LEFT, padx=10)
        self.radio_mp4.pack(side=tk.LEFT, padx=10)
        
        # Buttons Frame
        self.frame_buttons = tk.Frame(master)
        self.frame_buttons.pack(pady=5)
        self.button_list_songs = tk.Button(self.frame_buttons, text="Şarkıları Listele", command=self.list_songs)
        self.button_list_songs.pack(side=tk.LEFT, padx=5)
        self.button_select_folder = tk.Button(self.frame_buttons, text="Klasör Seç", command=self.select_folder)
        self.button_select_folder.pack(side=tk.LEFT, padx=5)
        self.button_toggle_multi = tk.Button(self.frame_buttons, text="Çoklu İndirme Aç", command=self.toggle_multi_download)
        self.button_toggle_multi.pack(side=tk.LEFT, padx=5)
        self.button_download_selected = tk.Button(self.frame_buttons, text="Seçilenleri İndir", command=self.download_selected)
        self.button_download_selected.pack(side=tk.LEFT, padx=5)
        self.button_stop = tk.Button(self.frame_buttons, text="İndirmeyi Durdur", command=self.stop_downloads, state=tk.DISABLED)
        self.button_stop.pack(side=tk.LEFT, padx=5)
        self.button_select_all = tk.Button(self.frame_buttons, text="Tümünü Seç", command=self.select_all_songs)
        self.button_select_all.pack(side=tk.LEFT, padx=5)
        
        # Status Frame
        self.frame_status = tk.Frame(master)
        self.frame_status.pack(pady=5)
        self.label_total = tk.Label(self.frame_status, text="Toplam Şarkı Sayısı: 0")
        self.label_total.pack(side=tk.LEFT, padx=10)
        self.label_remaining = tk.Label(self.frame_status, text="Kalan İndirme Sayısı: 0")
        self.label_remaining.pack(side=tk.LEFT, padx=10)
        
        # Listbox for Songs
        self.listbox_songs = tk.Listbox(master, selectmode=tk.MULTIPLE, width=100, height=20)
        self.listbox_songs.pack(pady=10)
        
        # Progress Bar
        self.progress_var = tk.DoubleVar()
        self.progress_bar = ttk.Progressbar(master, variable=self.progress_var, maximum=100)
        self.progress_bar.pack(fill=tk.X, padx=20, pady=5)
        
        # Log Text Box
        self.log_text = scrolledtext.ScrolledText(master, width=110, height=15, state='disabled')
        self.log_text.pack(pady=10)
        
        # Initialize a lock for thread-safe GUI updates
        self.lock = threading.Lock()
        
    def log(self, message):
        """Log mesajını günceller"""
        def append():
            self.log_text.config(state='normal')
            self.log_text.insert(tk.END, message + "\n")
            self.log_text.see(tk.END)
            self.log_text.config(state='disabled')
        self.master.after(0, append)
    
    def sanitize_filename(self, name):
        """Dosya adını temizler ve güvenli hale getirir"""
        return re.sub(r'[\\/*?:"<>|]', "", name)
    
    def select_folder(self):
        """Kullanıcıdan indirme klasörünü seçmesini sağlar"""
        folder = filedialog.askdirectory()
        if folder:
            self.download_folder = folder
            self.log(f"İndirilecek klasör: {self.download_folder}")
    
    def list_songs(self):
        """Girilen URL'den şarkıları listeler"""
        url = self.entry_url.get().strip()
        if not url:
            messagebox.showerror("Hata", "Lütfen geçerli bir URL girin.")
            return
        
        self.listbox_songs.delete(0, tk.END)  # Önceki listeyi temizle
        self.song_entries = []  # Reset song entries
        self.log("Şarkılar listeleniyor...")
        self.label_total.config(text="Toplam Şarkı Sayısı: 0")
        self.label_remaining.config(text="Kalan İndirme Sayısı: 0")
        
        def fetch_songs():
            ydl_opts = {
                'quiet': True,
                'extract_flat': 'in_playlist',  # Ensure we are extracting playlist entries
                'force_generic_extractor': True,
            }
            try:
                with yt_dlp.YoutubeDL(ydl_opts) as ydl:
                    info_dict = ydl.extract_info(url, download=False)
                    if 'entries' not in info_dict:
                        raise ValueError("Geçerli bir çalma listesi URL'si girin.")
                    for entry in info_dict['entries']:
                        title = entry.get('title', 'Başlık Yok')
                        video_url = entry.get('url', None)
                        if video_url:
                            # Tam video URL'sini oluşturmak için youtube.com video id ekliyoruz
                            if video_url.startswith("http"):
                                full_url = video_url
                            else:
                                full_url = f"https://www.youtube.com/watch?v={video_url}"
                            self.song_entries.append((title, full_url))
                            sanitized_title = self.sanitize_filename(title)
                            # GUI güncellemesini ana thread üzerinden yap
                            self.master.after(0, lambda t=title: self.listbox_songs.insert(tk.END, t))
                total_songs = len(self.song_entries)
                self.master.after(0, lambda: self.label_total.config(text=f"Toplam Şarkı Sayısı: {total_songs}"))
                self.master.after(0, lambda: self.label_remaining.config(text=f"Kalan İndirme Sayısı: {total_songs}"))
                self.log("Şarkılar başarıyla listelendi.")
            except Exception as e:
                self.log(f"Şarkılar listelenirken hata oluştu: {e}")
                self.master.after(0, lambda: messagebox.showerror("Hata", f"Şarkılar listelenirken hata oluştu: {e}"))
        
        threading.Thread(target=fetch_songs, daemon=True).start()
    
    def toggle_multi_download(self):
        """Çoklu indirme özelliğini açar veya kapatır"""
        self.is_multi_download = not self.is_multi_download
        if self.is_multi_download:
            self.button_toggle_multi.config(text="Çoklu İndirme Kapat")
            self.log("Çoklu indirme etkinleştirildi. Aynı anda 5 indirme yapılacak.")
        else:
            self.button_toggle_multi.config(text="Çoklu İndirme Aç")
            self.log("Çoklu indirme devre dışı bırakıldı. İndirmeler sırayla yapılacak.")
    
    def select_all_songs(self):
        """Listbox'taki tüm şarkıları seçer"""
        self.listbox_songs.select_set(0, tk.END)
        self.log("Tüm şarkılar seçildi.")
    
    def download_selected(self):
        """Seçilen şarkıları indirir"""
        selected_indices = self.listbox_songs.curselection()
        if not selected_indices:
            messagebox.showwarning("Uyarı", "İndirilecek şarkıları seçin.")
            return
        
        if not self.download_folder:
            messagebox.showwarning("Uyarı", "İndirme klasörünü seçin.")
            return
        
        selected_songs = [self.song_entries[i] for i in selected_indices]
        remaining_songs = len(selected_songs)
        self.log(f"{remaining_songs} şarkı indirilmeye başlandı.")
        
        self.button_download_selected.config(state=tk.DISABLED)
        self.button_stop.config(state=tk.NORMAL)
        
        # Reset Progress Bar
        self.progress_var.set(0)
        
        # Update remaining count
        self.master.after(0, lambda: self.label_remaining.config(text=f"Kalan İndirme Sayısı: {remaining_songs}"))
        
        # Initialize Download Queue
        for song in selected_songs:
            self.download_queue.put(song)
        
        # Start Worker Threads
        for _ in range(self.max_concurrent_downloads if self.is_multi_download else 1):
            threading.Thread(target=self.download_worker, daemon=True).start()
    
    def download_worker(self):
        """İndirme işlemlerini yöneten işçi thread'i"""
        while not self.download_queue.empty() and not self.stop_event.is_set():
            song_title, song_url = self.download_queue.get()
            sanitized_title = self.sanitize_filename(song_title)
            file_extension = "mp3" if self.format_var.get() == "mp3" else "mp4"
            file_path = os.path.join(self.download_folder, f"{sanitized_title}.{file_extension}")
            
            if os.path.exists(file_path):
                self.log(f"{song_title} zaten indirildi. Atlanıyor.")
                self.update_remaining_count()
                self.download_queue.task_done()
                continue
            
            self.active_downloads += 1
            self.log(f"{song_title} indiriliyor...")
            ydl_opts = self.get_ydl_opts(sanitized_title, song_url)
            try:
                with yt_dlp.YoutubeDL(ydl_opts) as ydl:
                    ydl.download([song_url])
                self.log(f"{song_title} başarıyla indirildi.")
                # İndirme tamamlandıktan sonra listeden sil
                self.master.after(0, lambda st=song_title: self.remove_song_from_list(st))
            except Exception as e:
                self.log(f"{song_title} indirirken hata oluştu: {e}")
            finally:
                self.active_downloads -= 1
                self.update_remaining_count()
                self.download_queue.task_done()
        
        # If no active downloads and queue is empty, re-enable buttons
        if self.active_downloads == 0 and self.download_queue.empty():
            self.master.after(0, lambda: self.button_download_selected.config(state=tk.NORMAL))
            self.master.after(0, lambda: self.button_stop.config(state=tk.DISABLED))
            if self.stop_event.is_set():
                self.log("İndirme işlemi durduruldu.")
            else:
                self.log("Tüm indirmeler tamamlandı.")
    
    def get_ydl_opts(self, title, url):
        """İndirme seçeneklerini belirler"""
        format_choice = self.format_var.get()
        if format_choice == "mp3":
            return {
                'format': 'bestaudio/best',
                'outtmpl': os.path.join(self.download_folder, f'{title}.%(ext)s'),
                'postprocessors': [{
                    'key': 'FFmpegExtractAudio',
                    'preferredcodec': 'mp3',
                    'preferredquality': '320',  # En iyi kalite
                }],
                'quiet': True,
                'progress_hooks': [self.progress_hook],
            }
        else:
            return {
                'format': 'bestvideo[height<=1080]+bestaudio/best',
                'merge_output_format': 'mp4',
                'outtmpl': os.path.join(self.download_folder, f'{title}.%(ext)s'),
                'postprocessors': [{
                    'key': 'FFmpegVideoConvertor',
                    'preferedformat': 'mp4',
                }],
                'quiet': True,
                'progress_hooks': [self.progress_hook],
            }
    
    def progress_hook(self, d):
        """İndirme ilerlemesini günceller"""
        if d['status'] == 'downloading':
            percent = d.get('_percent_str', '0.0%')
            try:
                percent_value = float(percent.strip('%'))
                self.master.after(0, lambda: self.progress_var.set(percent_value))
            except ValueError:
                pass
        elif d['status'] == 'finished':
            self.master.after(0, lambda: self.progress_var.set(100))
    
    def update_remaining_count(self):
        """Kalan indirme sayısını günceller"""
        current_remaining = self.download_queue.qsize()
        self.master.after(0, lambda: self.label_remaining.config(text=f"Kalan İndirme Sayısı: {current_remaining}"))
    
    def remove_song_from_list(self, song_title):
        """İndirilen şarkıyı listeden siler"""
        try:
            index = self.listbox_songs.get(0, tk.END).index(song_title)
            self.listbox_songs.delete(index)
            self.log(f"{song_title} listeden silindi.")
            # Update total and remaining counts
            self.master.after(0, lambda: self.label_total.config(text=f"Toplam Şarkı Sayısı: {self.listbox_songs.size()}"))
            self.master.after(0, lambda: self.label_remaining.config(text=f"Kalan İndirme Sayısı: {self.download_queue.qsize()}"))
        except ValueError:
            pass  # Şarkı listede bulunamazsa hiçbir şey yapma
    
    def stop_downloads(self):
        """İndirme işlemlerini durdurur"""
        self.stop_event.set()
        self.log("İndirme işlemi durduruluyor...")
        self.button_stop.config(state=tk.DISABLED)

if __name__ == "__main__":
    root = tk.Tk()
    app = YouTubePlaylistDownloader(root)
    root.mainloop()


