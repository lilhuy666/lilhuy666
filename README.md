import ttkbootstrap as ttk
from ttkbootstrap.constants import *
from datetime import datetime
import json, os
import hashlib
import secrets

# ===================== STYLE =====================
MAIN_BG = "#f8f9fa"  
AUTH_BG = "#f8f9fa"   
BG = "#0b1220"
PANEL = "#111a2e"
CARD = "#162238"
ACCENT = "#4f8cff"
ACCENT2 = "#22c55e"
TEXT = "#e5e7eb"
SUB = "#94a3b8"
DANGER = "#ef4444"
PROFILE_BG = "#162238"  
AVATAR_BG = "#253145"

DATA_FILE = "data.json"
BACKUP_FILE = "data.json.backup"

# ===================== DATA =====================
data = {"users": {}}
current_user = None

def hash_password(password, salt=None):
    """Хеширование пароля с солью"""
    if salt is None:
        salt = secrets.token_hex(16)
    hashed = hashlib.pbkdf2_hmac('sha256', password.encode(), salt.encode(), 100000)
    return f"{salt}:{hashed.hex()}"

def verify_password(stored_password, provided_password):
    """Проверка пароля"""
    salt, _ = stored_password.split(':')
    return hash_password(provided_password, salt) == stored_password

def load_data():
    global data
    if os.path.exists(DATA_FILE):
        try:
            with open(DATA_FILE, "r", encoding="utf-8") as f:
                data = json.load(f)
            if "users" not in data:
                data = {"users": {}}
        except Exception as e:
            ttk.messagebox.showerror("Ошибка", f"Не удалось загрузить данные: {e}")
            data = {"users": {}}
    create_backup()

def create_backup():
    """Создание бэкапа данных"""
    try:
        with open(DATA_FILE, "r", encoding="utf-8") as f_in:
            with open(BACKUP_FILE, "w", encoding="utf-8") as f_out:
                f_out.write(f_in.read())
    except:
        pass

def save_data():
    try:
        create_backup()
        with open(DATA_FILE, "w", encoding="utf-8") as f:
            json.dump(data, f, indent=2, ensure_ascii=False)
    except Exception as e:
        ttk.messagebox.showerror("Ошибка", f"Не удалось сохранить данные: {e}")

# ===================== WINDOW =====================
root = ttk.Window(themename="superhero")
root.title("CalculatCar")
root.geometry("1200x750")

main = ttk.Frame(root)
main.pack(fill="both", expand=True)

# ===================== HEADER =====================
header = ttk.Frame(main, height=70)
header.pack(fill="x")
header.pack_propagate(False)

menu_window = None

def toggle_menu():
    global menu_window

    if menu_window and menu_window.winfo_exists():
        menu_window.destroy()
        return

    x = root.winfo_x() + 20
    y = root.winfo_y() + 70

    menu_window = ttk.Window(root)
    menu_window.overrideredirect(True)
    menu_window.configure(bg=PANEL)
    menu_window.geometry(f"220x240+{x}+{y}")

    def close(e=None):
        if menu_window:
            menu_window.destroy()

    menu_window.bind("<FocusOut>", close)
    menu_window.bind("<Escape>", close)

    def nav(text, cmd):
        ttk.Button(menu_window,
                   text=text,
                   style='Info.TButton',
                   command=lambda: [cmd(), close()]
                  ).pack(fill="x", pady=5)

    nav("Калькулятор", show_calc)
    nav("Профиль", show_profile)
    nav("История", show_history)
    nav("Настройки", show_settings)
    nav("О программе", show_about)

    menu_window.focus_force()

ttk.Button(header, text="☰", style='Primary.TButton',
           command=toggle_menu).pack(side="left", padx=15)

ttk.Label(header, text="CalculatCar",
          style='Primary.TLabel',
          font=("Arial", 20, "bold")).pack(side="left")

user_label = ttk.Label(header, text="", style='Primary.TLabel')
user_label.pack(side="right", padx=15)

# ===================== CONTENT =====================
content = ttk.Frame(main, style='TFrame', background=BG)
content.pack(fill="both", expand=True)

def clear():
    for w in content.winfo_children():
        w.destroy()

def update_user():
    if current_user:
        user_label.config(text=f"Пользователь: {current_user}")
    else:
        user_label.config(text="Гость")

# ... (Остальной код функций show_auth, show_profile и т.д. остается тем же,
# но внутри них нужно заменить tk. на ttk. и использовать стили .configure(style='...')
# Например: button.configure(style='Accent.TButton') )
