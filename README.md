import tkinter as tk
from tkinter import ttk, messagebox, filedialog
import json
import os
import re
import hashlib
import shutil
from datetime import datetime, timedelta
import math
import random
import string
import smtplib
import ssl
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
import threading

try:
    from PIL import Image, ImageTk, ImageDraw, UnidentifiedImageError
except ImportError:
    import subprocess, sys
    subprocess.check_call([sys.executable, "-m", "pip", "install", "Pillow"])
    from PIL import Image, ImageTk, ImageDraw, UnidentifiedImageError

# ═══════════════════════════════════════════════════
# НАСТРОЙКИ ПОЧТЫ
# ═══════════════════════════════════════════════════
SMTP_EMAIL = "Kir-7800201904@yandex.ru"     # ваш email
SMTP_PASSWORD = "qxumbrfclsjkrrhk"          # ваш пароль приложения
SMTP_SERVER = "smtp.yandex.ru"              # Яндекс сервер
SMTP_PORT = 465
# ═══════════════════════════════════════════════════

DATA_FILE = os.path.join(os.path.expanduser("~"), ".fuel_calc_data.json")
PHOTO_DIR = os.path.join(os.path.expanduser("~"), ".fuel_calc_photos")
os.makedirs(PHOTO_DIR, exist_ok=True)

def load_data():
    if os.path.exists(DATA_FILE):
        try:
            with open(DATA_FILE, "r", encoding="utf-8") as f:
                return json.load(f)
        except (json.JSONDecodeError, IOError):
            return {"users": {}, "current_user": None, "settings": {}, "codes": {}}
    return {"users": {}, "current_user": None, "settings": {}, "codes": {}}

def save_data(data):
    try:
        with open(DATA_FILE, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
    except IOError:
        pass

def hash_pw(pw):
    return hashlib.sha256(pw.encode()).hexdigest()

def is_valid_email(email):
    return re.match(r"^[^@\s]+@[^@\s]+\.[^@\s]+$", email) is not None

def generate_verification_code():
    return ''.join(random.choices(string.digits, k=6))

def send_email_real(to_email, subject, body):
    """Реальная отправка email"""
    try:
        msg = MIMEMultipart()
        msg["From"] = SMTP_EMAIL
        msg["To"] = to_email
        msg["Subject"] = subject
        msg.attach(MIMEText(body, "plain", "utf-8"))

        context = ssl.create_default_context()
        with smtplib.SMTP_SSL(SMTP_SERVER, SMTP_PORT, context=context) as server:
            server.login(SMTP_EMAIL, SMTP_PASSWORD)
            server.sendmail(SMTP_EMAIL, to_email, msg.as_string())

        print(f"✓ Письмо отправлено на {to_email}")
        return True

    except smtplib.SMTPAuthenticationError:
        print(f"✗ Ошибка авторизации! Проверьте почту и пароль.")
        return False
    except Exception as e:
        print(f"✗ Ошибка отправки: {e}")
        return False

def send_verification_email(to_email, code):
    """Отправляет код подтверждения"""
    subject = "Fuel Calculator - Код подтверждения"
    body = f"""Здравствуйте!

Ваш код подтверждения: {code}

Код действителен 10 минут.

Если вы не запрашивали код — проигнорируйте письмо.

С уважением, Fuel Calculator
"""
    return send_email_real(to_email, subject, body)

# ─── Currencies ────────────────────────────────────────────────────────────────
CURRENCIES = {"₽ RUB": "₽", "$ USD": "$"}

# ─── Gas stations ──────────────────────────────────────────────────────────────
GAS_STATIONS_RUSSIA = [
    {"name": "Лукойл", "city": "Москва", "address": "Ленинградское ш., 1", "lat": 55.7558, "lon": 37.6173, "brand": "Лукойл", "price": 52.50},
    {"name": "Газпромнефть", "city": "Москва", "address": "Пр-т Мира, 100", "lat": 55.7512, "lon": 37.6184, "brand": "Газпромнефть", "price": 53.20},
    {"name": "Роснефть", "city": "Москва", "address": "Варшавское ш., 25", "lat": 55.7600, "lon": 37.6200, "brand": "Роснефть", "price": 51.80},
    {"name": "Лукойл", "city": "Санкт-Петербург", "address": "Невский пр., 50", "lat": 59.9343, "lon": 30.3351, "brand": "Лукойл", "price": 53.00},
    {"name": "Газпромнефть", "city": "Санкт-Петербург", "address": "Московский пр., 150", "lat": 59.9290, "lon": 30.3200, "brand": "Газпромнефть", "price": 54.00},
    {"name": "Роснефть", "city": "Казань", "address": "Пр-т Победы, 100", "lat": 55.7961, "lon": 49.1064, "brand": "Роснефть", "price": 50.50},
    {"name": "Лукойл", "city": "Екатеринбург", "address": "ул. Ленина, 50", "lat": 56.8389, "lon": 60.6057, "brand": "Лукойл", "price": 51.00},
    {"name": "Газпромнефть", "city": "Новосибирск", "address": "Красный пр., 100", "lat": 55.0302, "lon": 82.9204, "brand": "Газпромнефть", "price": 50.00},
    {"name": "Роснефть", "city": "Сочи", "address": "Курортный пр., 50", "lat": 43.5855, "lon": 39.7231, "brand": "Роснефть", "price": 55.00},
    {"name": "Лукойл", "city": "Краснодар", "address": "ул. Красная, 100", "lat": 45.0355, "lon": 38.9753, "brand": "Лукойл", "price": 52.00},
    {"name": "Газпромнефть", "city": "Владивосток", "address": "Океанский пр., 50", "lat": 43.1155, "lon": 131.8855, "brand": "Газпромнефть", "price": 54.50},
    {"name": "Роснефть", "city": "Нижний Новгород", "address": "ул. Б.Покровская, 50", "lat": 56.3268, "lon": 44.0065, "brand": "Роснефть", "price": 51.50},
]

# ─── Translations ──────────────────────────────────────────────────────────────
TRANS = {
    "ru": {
        "app_title": "Калькулятор расхода топлива",
        "calculator": "Калькулятор",
        "profile": "Профиль",
        "history": "История",
        "gas_stations": "Заправки",
        "about": "О приложении",
        "settings": "Настройки",
        "no_car": "— Без авто —",
        "car": "Автомобиль",
        "calculate": "Рассчитать",
        "clear": "Очистить",
        "distance_label": "Расстояние",
        "fuel_used": "Топлива",
        "fuel_price": "Цена за литр",
        "avg_consumption_label": "Средний расход",
        "calc_mode_consumption": "Расход на 100 км",
        "calc_mode_cost": "Стоимость поездки",
        "fuel_consumption": "Расход топлива",
        "trip_cost": "Стоимость",
        "login": "Войти",
        "register": "Регистрация",
        "email": "Email",
        "password": "Пароль",
        "repeat_password": "Повторите пароль",
        "create_account": "Создать аккаунт",
        "login_account": "Вход",
        "fill_all": "Заполните все поля",
        "invalid_email": "Введите корректный email",
        "password_short": "Минимум 6 символов",
        "passwords_mismatch": "Пароли не совпадают",
        "user_exists": "Пользователь уже существует",
        "wrong_credentials": "Неверный email или пароль",
        "forgot_password": "Забыли пароль?",
        "forgot_password_title": "Восстановление пароля",
        "send_code_btn": "Отправить код",
        "enter_code": "Введите код из письма",
        "verify_code": "Подтвердить",
        "invalid_code": "Неверный код",
        "verification_sent": "Код отправлен на почту! Проверьте папку Спам.",
        "new_password": "Новый пароль",
        "change_password_btn": "Сменить пароль",
        "password_changed": "Пароль изменён!",
        "sending": "Отправка...",
        "logout": "Выйти",
        "delete_account": "Удалить аккаунт",
        "my_cars": "Мои авто",
        "add_car": "+ Добавить",
        "no_cars": "Нет авто",
        "delete": "Удалить",
        "car_name": "Название",
        "choose_photo": "Выбрать фото",
        "save": "Сохранить",
        "enter_name": "Введите название",
        "history_title": "История",
        "login_for_history": "Войдите для просмотра.",
        "clear_all": "Очистить всё",
        "history_empty": "История пуста",
        "copy": "Копировать",
        "copied": "Скопировано!",
        "delete_entry": "Удалить",
        "settings_title": "Настройки",
        "theme": "Тема",
        "dark_theme": "Тёмная",
        "light_theme": "Светлая",
        "currency_setting": "Валюта",
        "language_setting": "Язык",
        "error": "Ошибка",
        "enter_numbers": "Введите числа!",
        "distance_positive": "Расстояние > 0!",
        "gs_title": "Заправки",
        "gs_city": "Город:",
        "gs_search": "Показать",
        "gs_to_calc": "В калькулятор",
        "gs_enter_distance": "Расстояние до заправки (км):",
        "gs_click_hint": "Нажмите на город на карте",
        "gs_all_cities": "Все города",
        "gs_fuel_price_hint": "Цена:",
        "about_title": "О приложении",
        "version": "Версия 5.0 • 2025",
        "about_section1": "Калькулятор расхода топлива с отправкой кодов на почту.",
        "footer": "© 2025 Fuel Calculator",
        "fuel_unit": "л",
        "no_cars_limit": "Максимум 5 авто",
        "quick_stats": "Статистика",
        "avg_consumption": "Средний расход",
        "distance": "Расстояние",
        "fuel": "Топливо",
        "cost": "Стоимость",
        "change_password": "Сменить пароль",
        "new_password_set": "Пароль установлен!",
        "delete_account_msg": "Удалить аккаунт безвозвратно?",
        "gs_avg_from_car": "Расход авто",
        "gs_no_avg": "Нет данных",
        "gs_selected_station": "Выбрано:",
        "gs_distance_note": "Введите расстояние",
        "delete_confirm": "Удалить?",
        "clear_history_confirm": "Очистить историю?",
        "gs_no_results": "Нет данных",
    }
}

def TR(key):
    return TRANS["ru"].get(key, key)

def get_currency_symbol():
    return CURRENCIES.get("₽ RUB", "₽")

# ─── Theme ─────────────────────────────────────────────────────────────────────
THEMES = {
    "dark": {"bg": "#0d1117", "bg2": "#161b22", "bg3": "#21262d", "fg": "#e6edf3",
             "fg2": "#8b949e", "accent": "#58a6ff", "green": "#3fb950", "yellow": "#d29922",
             "red": "#f85149", "border": "#30363d", "btn": "#238636", "btn2": "#21262d",
             "input_bg": "#0d1117", "chart_bg": "#161b22",
             "map_bg": "#1a2a3a", "map_land": "#2a4a2a"},
    "light": {"bg": "#f0f4f8", "bg2": "#ffffff", "bg3": "#e2e8f0", "fg": "#1a202c",
              "fg2": "#718096", "accent": "#3182ce", "green": "#38a169", "yellow": "#d69e2e",
              "red": "#e53e3e", "border": "#cbd5e0", "btn": "#3182ce", "btn2": "#e2e8f0",
              "input_bg": "#ffffff", "chart_bg": "#ffffff",
              "map_bg": "#e8f0e8", "map_land": "#c8e6c9"}
}

current_theme = "dark"
current_currency = "₽ RUB"

def T():
    return THEMES[current_theme]

# ─── Scroll helper ─────────────────────────────────────────────────────────────
def make_scrollable(parent):
    t = T()
    canvas = tk.Canvas(parent, bg=t["bg"], highlightthickness=0)
    scroll = ttk.Scrollbar(parent, orient="vertical", command=canvas.yview)
    canvas.configure(yscrollcommand=scroll.set)
    scroll.pack(side="right", fill="y")
    canvas.pack(side="left", fill="both", expand=True)
    inner = tk.Frame(canvas, bg=t["bg"])
    win = canvas.create_window((0, 0), window=inner, anchor="nw")
    inner.bind("<Configure>", lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
    canvas.bind("<Configure>", lambda e: canvas.itemconfig(win, width=e.width))

    def _on_mousewheel(event):
        if event.num == 4: canvas.yview_scroll(-1, "units")
        elif event.num == 5: canvas.yview_scroll(1, "units")
        else: canvas.yview_scroll(int(-1 * (event.delta / 120)), "units")

    def bind_mw(e): canvas.bind_all("<MouseWheel>", _on_mousewheel)
    def unbind_mw(e): canvas.unbind_all("<MouseWheel>")
    canvas.bind("<Enter>", bind_mw)
    canvas.bind("<Leave>", unbind_mw)
    return inner

# ─── Image helpers ─────────────────────────────────────────────────────────────
def make_placeholder(w=300, h=180):
    t = T()
    try:
        img = Image.new("RGB", (w, h), t["bg3"])
        draw = ImageDraw.Draw(img)
        draw.rectangle([0, 0, w-1, h-1], outline=t["border"], width=2)
        draw.text((w//2-15, h//2-8), TR("no_car"), fill=t["fg2"])
        return img
    except:
        return Image.new("RGB", (w, h), "#333")

def load_car_image(path, w=300, h=180):
    if not path or not isinstance(path, str) or not os.path.exists(path):
        return make_placeholder(w, h)
    try:
        if os.path.getsize(path) < 100: return make_placeholder(w, h)
    except: return make_placeholder(w, h)
    try:
        img = Image.open(path)
        img.verify()
        img = Image.open(path)
        if img.mode == 'RGBA':
            bg = Image.new('RGB', img.size, (0,0,0))
            bg.paste(img, mask=img.split()[3])
            img = bg
        elif img.mode != 'RGB': img = img.convert('RGB')
        sw, sh = img.size
        if sw <= 0 or sh <= 0: return make_placeholder(w, h)
        r = sw/sh
        nw, nh = (int(sh*w/sh), h) if r > w/h else (w, int(sw*w/sw))
        nw, nh = max(1, min(nw, 5000)), max(1, min(nh, 5000))
        img = img.resize((nw, nh), Image.LANCZOS)
        l, t = (nw-w)//2, (nh-h)//2
        return img.crop((l, t, l+w, t+h))
    except:
        return make_placeholder(w, h)

# ─── Main App ──────────────────────────────────────────────────────────────────
class FuelApp(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title(TR("app_title"))
        self.geometry("1100x820")
        self.minsize(900, 680)
        self.data = load_data()
        self.current_user = self.data.get("current_user")
        self._photo_refs = {}
        self._calc_mode = tk.StringVar(value="consumption")
        self._selected_station = None
        self._map_markers = []
        self._map_city_filter = tk.StringVar(value="")
        self._build_ui()

    def _save_settings(self):
        self.data.setdefault("settings", {})
        self.data["settings"] = {"theme": current_theme, "currency": current_currency}
        save_data(self.data)

    def _rebuild_ui(self):
        self.header.destroy()
        self.body.destroy()
        self._photo_refs.clear()
        self._map_markers.clear()
        self._build_ui()

    def _build_ui(self):
        t = T()
        self.configure(bg=t["bg"])

        self.header = tk.Frame(self, bg=t["bg2"], pady=10)
        self.header.pack(fill="x")
        tk.Label(self.header, text="⛽ " + TR("app_title"),
                 font=("Georgia", 16, "bold"), bg=t["bg2"], fg=t["fg"]).pack(side="left", padx=16)
        tk.Button(self.header, text="⚙", font=("Courier", 14), bg=t["bg3"], fg=t["fg"],
                  bd=0, padx=10, command=lambda: self._show("settings")).pack(side="right", padx=16)

        self.body = tk.Frame(self, bg=t["bg"])
        self.body.pack(fill="both", expand=True)

        # Навигация
        nav = tk.Frame(self.body, bg=t["bg2"], pady=6)
        nav.pack(fill="x")
        for name, icon in [("calculator","🧮"),("profile","👤"),("history","📋"),("gas_stations","⛽"),("about","ℹ️")]:
            tk.Button(nav, text=f"{icon} {TR(name)}", font=("Courier", 10),
                      bg=t["bg2"], fg=t["fg"], bd=0, padx=12, pady=6,
                      command=lambda n=name: self._show(n)).pack(side="left", padx=2)

        self.content = tk.Frame(self.body, bg=t["bg"])
        self.content.pack(fill="both", expand=True)
        self.sections = {}
        for name in ["calculator","profile","history","gas_stations","about","settings"]:
            self.sections[name] = tk.Frame(self.content, bg=t["bg"])
        self._build_calculator()
        self._build_profile()
        self._build_history()
        self._build_gas_stations()
        self._build_about()
        self._build_settings()
        self._show("calculator")

    def _show(self, name):
        for f in self.sections.values(): f.pack_forget()
        self.sections[name].pack(fill="both", expand=True)
        if name == "profile": self._refresh_profile()
        if name == "history": self._refresh_history()
        if name == "gas_stations": self._refresh_gas_stations()

    # ── EMAIL HELPERS ─────────────────────────────────────────────────────────
    def _save_code(self, email, code):
        self.data.setdefault("codes", {})[email] = {
            "code": code, "time": datetime.now().isoformat()}
        save_data(self.data)

    def _check_code(self, email, code):
        c = self.data.get("codes", {}).get(email)
        if c and c["code"] == code:
            try:
                if datetime.now() - datetime.fromisoformat(c["time"]) < timedelta(minutes=10):
                    return True
            except: pass
        return False

    def _send_code_thread(self, email, code, callback):
        def run():
            ok = send_verification_email(email, code)
            if ok: self._save_code(email, code)
            self.after(0, callback, ok)
        threading.Thread(target=run, daemon=True).start()

    # ── VERIFICATION DIALOG ───────────────────────────────────────────────────
    def _show_verify_dialog(self, email, purpose, password=None):
        t = T()
        code = generate_verification_code()
        dlg = tk.Toplevel(self)
        dlg.title("Подтверждение")
        dlg.geometry("380x300")
        dlg.configure(bg=t["bg2"])
        dlg.transient(self)
        dlg.grab_set()

        tk.Label(dlg, text="📧", font=("Courier", 36), bg=t["bg2"], fg=t["accent"]).pack(pady=8)
        tk.Label(dlg, text=f"Код отправлен на\n{email}",
                 font=("Courier", 10), bg=t["bg2"], fg=t["fg"]).pack()
        status = tk.Label(dlg, text=TR("sending"), font=("Courier", 10),
                          bg=t["bg2"], fg=t["yellow"])
        status.pack(pady=4)

        tk.Label(dlg, text=TR("enter_code"), font=("Courier", 10),
                 bg=t["bg2"], fg=t["fg2"]).pack(pady=(8, 0))
        code_entry = tk.Entry(dlg, font=("Courier", 18), bg=t["input_bg"],
                              fg=t["fg"], bd=1, relief="solid", justify="center", width=8)
        code_entry.pack(pady=6)
        code_entry.focus()

        err = tk.Label(dlg, text="", font=("Courier", 10), bg=t["bg2"], fg=t["red"])
        err.pack()

        def on_code_sent(ok):
            if ok:
                status.configure(text=TR("verification_sent"), fg=t["green"])
            else:
                status.configure(text=f"Не удалось отправить.\nКод: {code}", fg=t["yellow"])

        def verify():
            if self._check_code(email, code_entry.get().strip()):
                dlg.destroy()
                if purpose == "register" and password:
                    self.data.setdefault("users", {})[email] = {
                        "password": hash_pw(password), "cars": [], "history": []}
                    self.current_user = email
                    self.data["current_user"] = email
                    save_data(self.data)
                    self._refresh_car_combo()
                    self._refresh_profile()
                elif purpose == "login":
                    self.current_user = email
                    self.data["current_user"] = email
                    save_data(self.data)
                    self._refresh_car_combo()
                    self._refresh_profile()
                elif purpose == "reset":
                    dlg.destroy()
                    self._show_set_password_dialog(email)
            else:
                err.configure(text=TR("invalid_code"))

        tk.Button(dlg, text=TR("verify_code"), font=("Georgia", 11, "bold"),
                  bg=t["btn"], fg="white", bd=0, pady=8,
                  command=verify).pack(fill="x", padx=20, pady=8)
        dlg.bind("<Return>", lambda e: verify())

        self._send_code_thread(email, code, on_code_sent)

    def _show_set_password_dialog(self, email):
        t = T()
        dlg = tk.Toplevel(self)
        dlg.title(TR("forgot_password_title"))
        dlg.geometry("380x250")
        dlg.configure(bg=t["bg2"])
        dlg.transient(self)
        dlg.grab_set()

        tk.Label(dlg, text=TR("new_password"), font=("Georgia", 12, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack(pady=12)
        pw_entry = tk.Entry(dlg, font=("Courier", 12), show="•", bg=t["input_bg"],
                            fg=t["fg"], bd=1, relief="solid")
        pw_entry.pack(fill="x", padx=20, pady=4, ipady=6)
        pw_entry.focus()

        err = tk.Label(dlg, text="", font=("Courier", 10), bg=t["bg2"], fg=t["red"])
        err.pack()

        def set_pw():
            pw = pw_entry.get().strip()
            if len(pw) < 6:
                err.configure(text=TR("password_short"))
                return
            self.data["users"][email]["password"] = hash_pw(pw)
            save_data(self.data)
            messagebox.showinfo("OK", TR("new_password_set"))
            dlg.destroy()

        tk.Button(dlg, text=TR("change_password_btn"), font=("Georgia", 11, "bold"),
                  bg=t["btn"], fg="white", bd=0, pady=8,
                  command=set_pw).pack(fill="x", padx=20, pady=12)

    def _show_forgot_password_dialog(self):
        t = T()
        dlg = tk.Toplevel(self)
        dlg.title(TR("forgot_password_title"))
        dlg.geometry("380x220")
        dlg.configure(bg=t["bg2"])
        dlg.transient(self)
        dlg.grab_set()

        tk.Label(dlg, text=TR("forgot_password_title"), font=("Georgia", 13, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack(pady=12)
        tk.Label(dlg, text="Email:", font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", padx=20)
        email_entry = tk.Entry(dlg, font=("Courier", 12), bg=t["input_bg"],
                               fg=t["fg"], bd=1, relief="solid")
        email_entry.pack(fill="x", padx=20, pady=4, ipady=6)
        email_entry.focus()

        err = tk.Label(dlg, text="", font=("Courier", 10), bg=t["bg2"], fg=t["red"])
        err.pack()

        def send():
            email = email_entry.get().strip().lower()
            if not is_valid_email(email):
                err.configure(text=TR("invalid_email"))
                return
            if email not in self.data.get("users", {}):
                err.configure(text="Email не найден")
                return
            dlg.destroy()
            code = generate_verification_code()

            def on_sent(ok):
                if ok:
                    self._save_code(email, code)
                # Показываем диалог ввода кода
                self._show_reset_code_dialog(email, code)

            self._send_code_thread(email, code, on_sent)

        tk.Button(dlg, text=TR("send_code_btn"), font=("Georgia", 11, "bold"),
                  bg=t["btn"], fg="white", bd=0, pady=8,
                  command=send).pack(fill="x", padx=20, pady=12)

    def _show_reset_code_dialog(self, email, code):
        t = T()
        dlg = tk.Toplevel(self)
        dlg.title("Введите код")
        dlg.geometry("380x280")
        dlg.configure(bg=t["bg2"])
        dlg.transient(self)
        dlg.grab_set()

        tk.Label(dlg, text=TR("enter_code"), font=("Georgia", 12, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack(pady=12)
        code_entry = tk.Entry(dlg, font=("Courier", 18), bg=t["input_bg"],
                              fg=t["fg"], bd=1, relief="solid", justify="center", width=8)
        code_entry.pack(pady=6)
        code_entry.focus()

        err = tk.Label(dlg, text="", font=("Courier", 10), bg=t["bg2"], fg=t["red"])
        err.pack()

        def verify():
            if self._check_code(email, code_entry.get().strip()):
                dlg.destroy()
                self._show_set_password_dialog(email)
            else:
                err.configure(text=TR("invalid_code"))

        tk.Button(dlg, text=TR("verify_code"), font=("Georgia", 11, "bold"),
                  bg=t["btn"], fg="white", bd=0, pady=8,
                  command=verify).pack(fill="x", padx=20, pady=8)
        dlg.bind("<Return>", lambda e: verify())

    # ── CALCULATOR ────────────────────────────────────────────────────────────
    def _build_calculator(self):
        t = T()
        f = self.sections["calculator"]
        for w in f.winfo_children(): w.destroy()
        inner = make_scrollable(f)
        pad = tk.Frame(inner, bg=t["bg"])
        pad.pack(fill="both", expand=True, padx=20, pady=12)

        top = tk.Frame(pad, bg=t["bg"])
        top.pack(fill="x")
        left = tk.Frame(top, bg=t["bg2"])
        left.pack(side="left", fill="y", padx=(0, 16))

        ic = tk.Frame(left, bg=t["bg2"], width=300, height=180)
        ic.pack_propagate(False); ic.pack()
        self.calc_car_label = tk.Label(ic, bg=t["bg2"])
        self.calc_car_label.place(relx=0.5, rely=0.5, anchor="center")
        self._set_calc_car_image(None)

        tk.Label(left, text=TR("car")+":", font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(pady=(4,0))
        self.calc_car_var = tk.StringVar(value=TR("no_car"))
        self.calc_car_combo = ttk.Combobox(left, textvariable=self.calc_car_var,
                                           state="readonly", width=22, font=("Courier", 10))
        self.calc_car_combo.pack(padx=10, pady=4)
        self.calc_car_combo.bind("<<ComboboxSelected>>", self._on_car_selected)
        self._refresh_car_combo()

        right = tk.Frame(top, bg=t["bg2"])
        right.pack(side="left", fill="both", expand=True)

        tk.Label(right, text=TR("calculator"), font=("Georgia", 14, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=16, pady=(12,4))

        mf = tk.Frame(right, bg=t["bg2"]); mf.pack(fill="x", padx=16)
        tk.Radiobutton(mf, text=TR("calc_mode_consumption"), variable=self._calc_mode,
                       value="consumption", font=("Courier", 10), bg=t["bg2"], fg=t["fg"],
                       selectcolor=t["bg3"], command=self._build_calc_form).pack(side="left", padx=(0,12))
        tk.Radiobutton(mf, text=TR("calc_mode_cost"), variable=self._calc_mode,
                       value="cost", font=("Courier", 10), bg=t["bg2"], fg=t["fg"],
                       selectcolor=t["bg3"], command=self._build_calc_form).pack(side="left")

        self.calc_form_frame = tk.Frame(right, bg=t["bg2"])
        self.calc_form_frame.pack(fill="x", padx=16)
        self.dist_var = tk.StringVar()
        self.fuel_var = tk.StringVar()
        self.price_var = tk.StringVar()
        self.avg_var = tk.StringVar()
        self._build_calc_form()

        btn_row = tk.Frame(right, bg=t["bg2"]); btn_row.pack(fill="x", padx=16, pady=8)
        tk.Button(btn_row, text=TR("calculate"), font=("Georgia", 11, "bold"),
                  bg=t["btn"], fg="white", bd=0, pady=8,
                  command=self._calculate).pack(side="left", fill="x", expand=True, padx=(0,4))
        tk.Button(btn_row, text=TR("clear"), font=("Courier", 10),
                  bg=t["btn2"], fg=t["fg"], bd=0, pady=8,
                  command=self._clear_calc).pack(side="left", fill="x", expand=True)

        self.result_frame = tk.Frame(right, bg=t["bg3"])
        self.result_frame.pack(fill="x", padx=16, pady=8)
        self.result_container = tk.Frame(self.result_frame, bg=t["bg3"])
        self.result_container.pack(fill="x", padx=12, pady=10)
        self._update_result_widgets("consumption")

    def _update_result_widgets(self, mode):
        for w in self.result_container.winfo_children(): w.destroy()
        t = T()
        if mode == "consumption":
            tk.Label(self.result_container, text=TR("fuel_consumption"),
                     font=("Courier", 10), bg=t["bg3"], fg=t["fg2"]).pack(anchor="w")
            self.result_consumption_label = tk.Label(self.result_container, text="— л/100 км",
                                                     font=("Georgia", 18, "bold"), bg=t["bg3"], fg=t["green"])
            self.result_consumption_label.pack(anchor="w")
            self.result_cost_label = None
        else:
            tk.Label(self.result_container, text=TR("trip_cost"),
                     font=("Courier", 10), bg=t["bg3"], fg=t["fg2"]).pack(anchor="w")
            self.result_cost_label = tk.Label(self.result_container, text="—",
                                              font=("Georgia", 18, "bold"), bg=t["bg3"], fg=t["fg"])
            self.result_cost_label.pack(anchor="w")
            self.result_consumption_label = None

    def _build_calc_form(self):
        t = T()
        for w in self.calc_form_frame.winfo_children(): w.destroy()
        mode = self._calc_mode.get()
        sym = get_currency_symbol()

        def make_row(label, var, unit, icon):
            tk.Label(self.calc_form_frame, text=label, font=("Courier", 9),
                     bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", pady=(6,1))
            row = tk.Frame(self.calc_form_frame, bg=t["input_bg"], bd=1, relief="solid")
            row.pack(fill="x", pady=2)
            tk.Label(row, text=icon, font=("Courier", 12), bg=t["input_bg"], fg=t["fg2"], padx=6).pack(side="left")
            tk.Entry(row, textvariable=var, font=("Courier", 12), bg=t["input_bg"],
                     fg=t["fg"], bd=0).pack(side="left", fill="x", expand=True, pady=6)
            tk.Label(row, text=unit, font=("Courier", 9), bg=t["input_bg"], fg=t["fg2"], padx=6).pack(side="right")

        make_row(TR("distance_label"), self.dist_var, "км", "📏")
        if mode == "consumption":
            make_row(TR("fuel_used"), self.fuel_var, TR("fuel_unit"), "⛽")
            make_row(TR("fuel_price"), self.price_var, sym+"/"+TR("fuel_unit"), "💰")
        else:
            make_row(TR("avg_consumption_label"), self.avg_var, "л/100 км", "💧")
            make_row(TR("fuel_price"), self.price_var, sym+"/"+TR("fuel_unit"), "💰")

    def _set_calc_car_image(self, path):
        img = load_car_image(path, 300, 180)
        ph = ImageTk.PhotoImage(img)
        self._photo_refs["calc_car"] = ph
        self.calc_car_label.configure(image=ph, bg=T()["bg2"])

    def _refresh_car_combo(self):
        cars = [TR("no_car")]
        if self.current_user:
            for c in self.data["users"].get(self.current_user, {}).get("cars", []):
                cars.append(c["name"])
        self.calc_car_combo["values"] = cars
        self.calc_car_var.set(cars[0])

    def _on_car_selected(self, event=None):
        sel = self.calc_car_var.get()
        if sel == TR("no_car") or not self.current_user:
            self._set_calc_car_image(None); return
        for car in self.data["users"].get(self.current_user, {}).get("cars", []):
            if car["name"] == sel:
                self._set_calc_car_image(car.get("photo"))
                if car.get("avg_consumption"): self.avg_var.set(str(car["avg_consumption"]))
                return
        self._set_calc_car_image(None)

    def _calculate(self):
        sym = get_currency_symbol()
        mode = self._calc_mode.get()
        try:
            dist = float(self.dist_var.get().replace(",", "."))
            price = float(self.price_var.get().replace(",", "."))
            if mode == "consumption":
                fuel = float(self.fuel_var.get().replace(",", "."))
            else:
                avg_input = float(self.avg_var.get().replace(",", "."))
        except ValueError:
            messagebox.showerror(TR("error"), TR("enter_numbers")); return
        if dist <= 0:
            messagebox.showerror(TR("error"), TR("distance_positive")); return

        if mode == "consumption":
            consumption = (fuel / dist) * 100
            cost = fuel * price
            self.result_consumption_label.configure(text=f"{consumption:.2f} л/100 км")
        else:
            if avg_input <= 0:
                messagebox.showerror(TR("error"), "Расход > 0!"); return
            consumption = avg_input
            fuel = (avg_input / 100) * dist
            cost = fuel * price
            self.result_cost_label.configure(text=f"{cost:,.2f} {sym}")

        if self.current_user:
            entry = {"date": datetime.now().strftime("%d.%m.%Y %H:%M"),
                     "car": self.calc_car_var.get(), "distance": dist,
                     "fuel": round(fuel, 2), "price": price, "currency": sym,
                     "consumption": round(consumption, 2), "cost": round(cost, 2)}
            u = self.data["users"][self.current_user]
            u.setdefault("history", []).insert(0, entry)
            if self.calc_car_var.get() != TR("no_car"):
                for c in u.get("cars", []):
                    if c["name"] == self.calc_car_var.get():
                        c["avg_consumption"] = round(consumption, 2)
            save_data(self.data)

    def _clear_calc(self):
        for v in [self.dist_var, self.fuel_var, self.price_var, self.avg_var]:
            v.set("")
        if self.result_consumption_label: self.result_consumption_label.configure(text="— л/100 км")
        if self.result_cost_label: self.result_cost_label.configure(text="—")

    # ── PROFILE ───────────────────────────────────────────────────────────────
    def _build_profile(self):
        t = T()
        f = self.sections["profile"]
        for w in f.winfo_children(): w.destroy()
        inner = make_scrollable(f)
        pad = tk.Frame(inner, bg=t["bg"])
        pad.pack(fill="both", expand=True, padx=32, pady=20)
        if not self.current_user:
            self._build_login_form(pad)
        else:
            self._build_logged_in(pad)

    def _build_login_form(self, parent):
        t = T()
        self._auth_mode = tk.StringVar(value="login")
        tf = tk.Frame(parent, bg=t["bg"]); tf.pack()
        tk.Button(tf, text=TR("login"), font=("Georgia", 11, "bold"),
                  bg=t["btn"], fg="white", bd=0, padx=16, pady=4,
                  command=lambda: self._auth_mode.set("login") or build()).pack(side="left", padx=2)
        tk.Button(tf, text=TR("register"), font=("Georgia", 11),
                  bg=t["btn2"], fg=t["fg"], bd=0, padx=16, pady=4,
                  command=lambda: self._auth_mode.set("register") or build()).pack(side="left", padx=2)

        self._auth_form = tk.Frame(parent, bg=t["bg2"], padx=32, pady=20)
        self._auth_form.pack(fill="x", pady=16)

        def build():
            for w in self._auth_form.winfo_children(): w.destroy()
            mode = self._auth_mode.get()
            tk.Label(self._auth_form, text=TR("create_account") if mode=="register" else TR("login_account"),
                     font=("Georgia", 14, "bold"), bg=t["bg2"], fg=t["fg"]).pack(pady=(0,12))

            tk.Label(self._auth_form, text=TR("email"), font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w")
            email_entry = tk.Entry(self._auth_form, font=("Courier", 12), bg=t["input_bg"],
                                   fg=t["fg"], bd=1, relief="solid")
            email_entry.pack(fill="x", pady=4, ipady=4)

            tk.Label(self._auth_form, text=TR("password"), font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w")
            pw_entry = tk.Entry(self._auth_form, font=("Courier", 12), show="•", bg=t["input_bg"],
                                fg=t["fg"], bd=1, relief="solid")
            pw_entry.pack(fill="x", pady=4, ipady=4)

            pw2_entry = None
            if mode == "register":
                tk.Label(self._auth_form, text=TR("repeat_password"), font=("Courier", 10),
                         bg=t["bg2"], fg=t["fg2"]).pack(anchor="w")
                pw2_entry = tk.Entry(self._auth_form, font=("Courier", 12), show="•",
                                     bg=t["input_bg"], fg=t["fg"], bd=1, relief="solid")
                pw2_entry.pack(fill="x", pady=4, ipady=4)

            # Forgot password
            if mode == "login":
                tk.Button(self._auth_form, text=TR("forgot_password"), font=("Courier", 8, "underline"),
                          bg=t["bg2"], fg=t["accent"], bd=0, cursor="hand2",
                          command=self._show_forgot_password_dialog).pack(anchor="e", pady=2)

            err = tk.Label(self._auth_form, text="", font=("Courier", 10), bg=t["bg2"], fg=t["red"])
            err.pack(pady=4)

            def submit():
                email = email_entry.get().strip().lower()
                pw = pw_entry.get()
                if not email or not pw: err.configure(text=TR("fill_all")); return
                if not is_valid_email(email): err.configure(text=TR("invalid_email")); return
                if len(pw) < 6: err.configure(text=TR("password_short")); return

                if mode == "register":
                    if pw2_entry and pw != pw2_entry.get():
                        err.configure(text=TR("passwords_mismatch")); return
                    if email in self.data.get("users", {}):
                        err.configure(text=TR("user_exists")); return
                    self._show_verify_dialog(email, "register", pw)
                else:
                    users = self.data.get("users", {})
                    if email not in users or users[email]["password"] != hash_pw(pw):
                        err.configure(text=TR("wrong_credentials")); return
                    self._show_verify_dialog(email, "login")

            tk.Button(self._auth_form, text=TR("login") if mode=="login" else TR("register"),
                      font=("Georgia", 11, "bold"), bg=t["btn"], fg="white", bd=0, pady=8,
                      command=submit).pack(fill="x", pady=8)

        build()

    def _build_logged_in(self, parent):
        t = T()
        tk.Label(parent, text=f"👤 {self.current_user}", font=("Georgia", 13, "bold"),
                 bg=t["bg"], fg=t["fg"]).pack(anchor="w")
        btn_row = tk.Frame(parent, bg=t["bg"]); btn_row.pack(fill="x", pady=8)
        tk.Button(btn_row, text=TR("logout"), font=("Courier", 9),
                  bg=t["btn2"], fg=t["fg"], bd=0, padx=10, pady=4,
                  command=self._logout).pack(side="left", padx=(0,8))
        tk.Button(btn_row, text=TR("delete_account"), font=("Courier", 9),
                  bg=t["red"], fg="white", bd=0, padx=10, pady=4,
                  command=self._delete_account).pack(side="left")

        # Cars
        tk.Frame(parent, bg=t["border"], height=1).pack(fill="x", pady=12)
        ch = tk.Frame(parent, bg=t["bg"]); ch.pack(fill="x")
        tk.Label(ch, text=TR("my_cars"), font=("Georgia", 12, "bold"),
                 bg=t["bg"], fg=t["fg"]).pack(side="left")
        tk.Button(ch, text=TR("add_car"), font=("Courier", 9),
                  bg=t["btn"], fg="white", bd=0, padx=10, pady=4,
                  command=self._add_car).pack(side="right")
        self.cars_container = tk.Frame(parent, bg=t["bg"])
        self.cars_container.pack(fill="x", pady=8)
        self._render_cars()

    def _add_car(self):
        cars = self.data["users"][self.current_user].get("cars", [])
        if len(cars) >= 5:
            messagebox.showwarning(TR("error"), TR("no_cars_limit")); return

        t = T()
        dlg = tk.Toplevel(self)
        dlg.title(TR("add_car"))
        dlg.geometry("400x350")
        dlg.configure(bg=t["bg2"])
        dlg.transient(self)
        dlg.grab_set()

        tk.Label(dlg, text=TR("car_name"), font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", padx=16, pady=(12,0))
        name_var = tk.StringVar()
        tk.Entry(dlg, textvariable=name_var, font=("Courier", 12), bg=t["input_bg"],
                 fg=t["fg"], bd=1).pack(fill="x", padx=16, pady=4, ipady=4)

        photo_var = tk.StringVar()
        tk.Label(dlg, text=TR("choose_photo"), font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", padx=16, pady=(8,0))
        tk.Button(dlg, text="📷 Выбрать", font=("Courier", 10),
                  bg=t["btn2"], fg=t["fg"], bd=0, pady=4,
                  command=lambda: photo_var.set(filedialog.askopenfilename(
                      filetypes=[("Images", "*.jpg *.jpeg *.png")]) or "")).pack(anchor="w", padx=16)

        err = tk.Label(dlg, text="", font=("Courier", 9), bg=t["bg2"], fg=t["red"])
        err.pack(pady=4)

        def save():
            name = name_var.get().strip()
            if not name: err.configure(text=TR("enter_name")); return
            car = {"name": name, "photo": photo_var.get() or None}
            self.data["users"][self.current_user].setdefault("cars", []).append(car)
            save_data(self.data)
            dlg.destroy()
            self._render_cars()
            self._refresh_car_combo()

        tk.Button(dlg, text=TR("save"), font=("Georgia", 11, "bold"),
                  bg=t["btn"], fg="white", bd=0, pady=8,
                  command=save).pack(fill="x", padx=16, pady=12)

    def _render_cars(self):
        t = T()
        for w in self.cars_container.winfo_children(): w.destroy()
        if not self.current_user: return
        cars = self.data["users"][self.current_user].get("cars", [])
        if not cars:
            tk.Label(self.cars_container, text=TR("no_cars"),
                     font=("Courier", 10), bg=t["bg"], fg=t["fg2"]).pack()
            return
        row = tk.Frame(self.cars_container, bg=t["bg"]); row.pack(fill="x")
        for i, car in enumerate(cars):
            if i > 0 and i % 5 == 0:
                row = tk.Frame(self.cars_container, bg=t["bg"]); row.pack(fill="x")
            card = tk.Frame(row, bg=t["bg2"], padx=6, pady=6)
            card.pack(side="left", padx=4, pady=4)
            ib = tk.Frame(card, bg=t["bg2"], width=100, height=65)
            ib.pack_propagate(False); ib.pack()
            img = load_car_image(car.get("photo"), 100, 65)
            ph = ImageTk.PhotoImage(img)
            self._photo_refs[f"car_{i}"] = ph
            tk.Label(ib, image=ph, bg=t["bg2"]).place(relx=0.5, rely=0.5, anchor="center")
            tk.Label(card, text=car["name"], font=("Courier", 9, "bold"), bg=t["bg2"], fg=t["fg"]).pack()
            tk.Button(card, text=TR("delete"), font=("Courier", 8),
                      bg=t["red"], fg="white", bd=0, padx=6, pady=2,
                      command=lambda idx=i: self._delete_car(idx)).pack()

    def _delete_car(self, idx):
        if messagebox.askyesno(TR("delete_confirm"), "Удалить авто?"):
            del self.data["users"][self.current_user]["cars"][idx]
            save_data(self.data)
            self._render_cars()
            self._refresh_car_combo()

    def _logout(self):
        self.current_user = None
        self.data["current_user"] = None
        save_data(self.data)
        self._refresh_car_combo()
        self._refresh_profile()

    def _delete_account(self):
        if messagebox.askyesno(TR("delete_confirm"), TR("delete_account_msg")):
            del self.data["users"][self.current_user]
            self.current_user = None
            self.data["current_user"] = None
            save_data(self.data)
            self._refresh_car_combo()
            self._refresh_profile()

    def _refresh_profile(self):
        f = self.sections["profile"]
        for w in f.winfo_children(): w.destroy()
        self._build_profile()

    # ── HISTORY ───────────────────────────────────────────────────────────────
    def _build_history(self):
        t = T()
        f = self.sections["history"]
        for w in f.winfo_children(): w.destroy()
        inner = make_scrollable(f)
        pad = tk.Frame(inner, bg=t["bg"])
        pad.pack(fill="both", expand=True, padx=24, pady=16)

        tk.Label(pad, text=TR("history_title"), font=("Georgia", 14, "bold"),
                 bg=t["bg"], fg=t["fg"]).pack(anchor="w")
        if not self.current_user:
            tk.Label(pad, text=TR("login_for_history"), font=("Courier", 11),
                     bg=t["bg"], fg=t["fg2"]).pack(pady=20)
            return

        hdr = tk.Frame(pad, bg=t["bg"]); hdr.pack(fill="x", pady=4)
        tk.Button(hdr, text=TR("clear_all"), font=("Courier", 8),
                  bg=t["red"], fg="white", bd=0, padx=8, pady=3,
                  command=self._clear_history).pack(side="right")

        self._hist_inner = tk.Frame(pad, bg=t["bg"])
        self._hist_inner.pack(fill="x", pady=8)
        self._render_history()

    def _render_history(self):
        t = T()
        if not hasattr(self, '_hist_inner'): return
        for w in self._hist_inner.winfo_children(): w.destroy()
        if not self.current_user: return
        history = self.data["users"][self.current_user].get("history", [])
        if not history:
            tk.Label(self._hist_inner, text=TR("history_empty"),
                     font=("Courier", 10), bg=t["bg"], fg=t["fg2"]).pack(pady=20)
            return

        for i, entry in enumerate(history):
            card = tk.Frame(self._hist_inner, bg=t["bg2"], pady=6)
            card.pack(fill="x", pady=4, ipadx=8)

            sym = entry.get("currency", "₽")
            txt = (f"📅 {entry['date']}  |  🚗 {entry.get('car','—')}\n"
                   f"📏 {entry['distance']} км  ⛽ {entry['fuel']} л  "
                   f"💰 {entry['price']} {sym}/л\n"
                   f"💧 {entry['consumption']} л/100км  💳 {entry['cost']:,.2f} {sym}")

            tk.Label(card, text=txt, font=("Courier", 9), bg=t["bg2"], fg=t["fg"],
                     justify="left").pack(side="left", padx=12, pady=4)

            btn_col = tk.Frame(card, bg=t["bg2"])
            btn_col.pack(side="right", padx=8)

            copy_text = f"Дата: {entry['date']}\nАвто: {entry.get('car','—')}\n" \
                        f"Расстояние: {entry['distance']} км\nТопливо: {entry['fuel']} л\n" \
                        f"Цена: {entry['price']} {sym}/л\nРасход: {entry['consumption']} л/100км\n" \
                        f"Стоимость: {entry['cost']:,.2f} {sym}"

            def copy_entry(t=copy_text):
                self.clipboard_clear(); self.clipboard_append(t)
                messagebox.showinfo("OK", TR("copied"))

            tk.Button(btn_col, text=TR("copy"), font=("Courier", 8),
                      bg=t["green"], fg="white", bd=0, width=8,
                      command=copy_entry).pack(fill="x", pady=1)
            tk.Button(btn_col, text=TR("delete_entry"), font=("Courier", 8),
                      bg=t["red"], fg="white", bd=0, width=8,
                      command=lambda idx=i: self._del_hist(idx)).pack(fill="x", pady=1)

    def _del_hist(self, idx):
        del self.data["users"][self.current_user]["history"][idx]
        save_data(self.data)
        self._render_history()

    def _clear_history(self):
        if messagebox.askyesno(TR("clear_history_confirm"), "Удалить всю историю?"):
            self.data["users"][self.current_user]["history"] = []
            save_data(self.data)
            self._render_history()

    def _refresh_history(self):
        f = self.sections["history"]
        for w in f.winfo_children(): w.destroy()
        self._build_history()

    # ── GAS STATIONS ──────────────────────────────────────────────────────────
    def _build_gas_stations(self):
        t = T()
        f = self.sections["gas_stations"]
        for w in f.winfo_children(): w.destroy()
        inner = make_scrollable(f)
        pad = tk.Frame(inner, bg=t["bg"])
        pad.pack(fill="both", expand=True, padx=32, pady=16)

        tk.Label(pad, text=TR("gs_title"), font=("Georgia", 16, "bold"),
                 bg=t["bg"], fg=t["fg"]).pack(anchor="w", pady=(0,8))

        # Search
        sc = tk.Frame(pad, bg=t["bg2"], pady=12)
        sc.pack(fill="x", pady=8)
        tk.Label(sc, text=TR("gs_city"), font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", padx=16)
        sr = tk.Frame(sc, bg=t["bg2"]); sr.pack(fill="x", padx=16)
        cities = [TR("gs_all_cities")] + sorted(set(s["city"] for s in GAS_STATIONS_RUSSIA))
        self._map_city_filter = tk.StringVar(value=cities[0])
        ttk.Combobox(sr, textvariable=self._map_city_filter, values=cities,
                     state="readonly", font=("Courier", 10), width=18).pack(side="left", padx=(0,8))
        tk.Button(sr, text=TR("gs_search"), font=("Courier", 10),
                  bg=t["btn"], fg="white", bd=0, padx=12, pady=4,
                  command=self._draw_map).pack(side="left")

        # Map
        self.map_canvas = tk.Canvas(pad, bg=t["map_bg"], height=400,
                                    highlightthickness=1, highlightbackground=t["border"])
        self.map_canvas.pack(fill="x", pady=8)
        self.map_canvas.bind("<Button-1>", self._on_map_click)

        # Info
        self.station_info_text = tk.Label(pad, text=TR("gs_click_hint"),
                                          font=("Courier", 10), bg=t["bg"], fg=t["fg2"], justify="left")
        self.station_info_text.pack(anchor="w", pady=8)

        # Distance + price
        card = tk.Frame(pad, bg=t["bg2"], pady=12)
        card.pack(fill="x", pady=8)
        tk.Label(card, text=TR("gs_enter_distance"), font=("Courier", 10),
                 bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", padx=16)
        dr = tk.Frame(card, bg=t["input_bg"], bd=1); dr.pack(fill="x", padx=16, pady=4)
        tk.Label(dr, text="📏", bg=t["input_bg"], fg=t["fg2"], padx=6).pack(side="left")
        self.gs_dist_var = tk.StringVar()
        tk.Entry(dr, textvariable=self.gs_dist_var, font=("Courier", 12),
                 bg=t["input_bg"], fg=t["fg"], bd=0).pack(side="left", fill="x", expand=True, pady=6)
        tk.Label(dr, text="км", bg=t["input_bg"], fg=t["fg2"], padx=6).pack(side="right")

        tk.Label(card, text=TR("gs_fuel_price_hint"), font=("Courier", 10),
                 bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", padx=16, pady=(8,0))
        pr = tk.Frame(card, bg=t["input_bg"], bd=1); pr.pack(fill="x", padx=16, pady=4)
        tk.Label(pr, text="💰", bg=t["input_bg"], fg=t["fg2"], padx=6).pack(side="left")
        self.gs_price_var = tk.StringVar()
        tk.Entry(pr, textvariable=self.gs_price_var, font=("Courier", 12),
                 bg=t["input_bg"], fg=t["fg"], bd=0).pack(side="left", fill="x", expand=True, pady=6)
        tk.Label(pr, text=f"{get_currency_symbol()}/л", bg=t["input_bg"], fg=t["fg2"], padx=6).pack(side="right")

        tk.Button(pad, text=TR("gs_to_calc"), font=("Georgia", 11, "bold"),
                  bg=t["btn"], fg="white", bd=0, pady=10,
                  command=self._gs_to_calc).pack(fill="x", pady=8)

        self.after(200, self._draw_map)

    def _get_filtered_stations(self):
        city = self._map_city_filter.get()
        if city in (TR("gs_all_cities"), "Все города", "All cities"):
            return GAS_STATIONS_RUSSIA
        return [s for s in GAS_STATIONS_RUSSIA if s["city"] == city]

    def _lat_lon_to_xy(self, lat, lon, w, h):
        lat_min, lat_max = 41.0, 82.0
        lon_min, lon_max = 19.0, 170.0
        x = ((lon - lon_min) / (lon_max - lon_min)) * (w - 60) + 30
        y = ((lat_max - lat) / (lat_max - lat_min)) * (h - 60) + 30
        return x, y

    def _draw_map(self):
        c = self.map_canvas
        t = T()
        c.delete("all")
        self._map_markers = []
        W, H = c.winfo_width(), c.winfo_height()
        if W < 10 or H < 10: W, H = 700, 380

        c.create_rectangle(0, 0, W, H, fill=t["map_bg"], outline="")

        # Контур РФ
        outline = [(41,43),(41,45),(37,46),(36,47),(32,46),(28,44),(22,44),(20,55),(19,60),(20,65),
                   (28,69),(32,70),(40,69),(45,68),(50,72),(60,75),(70,76),(80,74),(90,73),(100,74),
                   (110,74),(120,72),(130,70),(140,68),(150,65),(160,62),(170,60),(170,55),(160,50),
                   (150,48),(140,45),(135,43),(130,42),(120,40),(110,42),(100,45),(90,50),(80,55),
                   (70,55),(60,50),(50,45),(45,42),(41,43)]
        pts = []
        for lat, lon in outline:
            x, y = self._lat_lon_to_xy(lat, lon, W, H)
            pts.extend([x, y])
        if pts: c.create_polygon(pts, fill=t["map_land"], outline=t["border"], width=1.5)

        # Города
        stations = self._get_filtered_stations()
        groups = {}
        for s in stations:
            city = s["city"]
            if city not in groups:
                groups[city] = {"lat": s["lat"], "lon": s["lon"], "stations": [], "count": 0}
            groups[city]["count"] += 1
            groups[city]["stations"].append(s)

        for city, data in groups.items():
            x, y = self._lat_lon_to_xy(data["lat"], data["lon"], W, H)
            r = max(8, min(20, 6 + data["count"] * 3))
            cid = c.create_oval(x-r, y-r, x+r, y+r, fill=t["accent"], outline=t["bg2"], width=2)
            c.create_text(x, y-r-12, text=city, fill=t["fg"], font=("Courier", 8, "bold"))
            c.create_text(x, y, text=str(data["count"]), fill="white", font=("Courier", 9, "bold"))
            self._map_markers.append({"city": city, "x": x, "y": y, "radius": r,
                                      "stations": data["stations"], "canvas_id": cid})

        if self._selected_station:
            m = self._selected_station
            c.create_oval(m["x"]-m["radius"]-4, m["y"]-m["radius"]-4,
                          m["x"]+m["radius"]+4, m["y"]+m["radius"]+4,
                          outline="#ffaa00", width=3)

    def _on_map_click(self, event):
        closest, best_d = None, 9999
        for m in self._map_markers:
            d = math.sqrt((event.x - m["x"])**2 + (event.y - m["y"])**2)
            if d < m["radius"] + 5 and d < best_d:
                closest, best_d = m, d
        if closest:
            self._selected_station = closest
            sts = closest["stations"]
            txt = f"🏙 {closest['city']}\n"
            for s in sts[:5]:
                txt += f"⛽ {s['brand']} - {s['address']}\n   💰 {s['price']} {get_currency_symbol()}/л\n"
            if len(sts) > 5: txt += f"... и ещё {len(sts)-5}\n"
            self.station_info_text.configure(text=txt, fg=T()["fg"])
            if sts: self.gs_price_var.set(str(sts[0]["price"]))
            self._draw_map()

    def _gs_to_calc(self):
        ds = self.gs_dist_var.get().strip()
        ps = self.gs_price_var.get().strip()
        if not ds: messagebox.showwarning(TR("error"), TR("gs_enter_distance")); return
        self._show("calculator")
        if ds: self.dist_var.set(ds)
        if ps: self.price_var.set(ps)

    def _refresh_gas_stations(self):
        if hasattr(self, 'map_canvas') and self.map_canvas.winfo_exists():
            self._draw_map()

    # ── ABOUT ─────────────────────────────────────────────────────────────────
    def _build_about(self):
        t = T()
        f = self.sections["about"]
        for w in f.winfo_children(): w.destroy()
        inner = make_scrollable(f)
        pad = tk.Frame(inner, bg=t["bg"])
        pad.pack(padx=48, pady=32)

        hero = tk.Frame(pad, bg=t["bg2"], pady=24)
        hero.pack(fill="x", pady=(0,20))
        tk.Label(hero, text="⛽", font=("Courier", 40), bg=t["bg2"], fg=t["accent"]).pack()
        tk.Label(hero, text=TR("app_title"), font=("Georgia", 20, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack()
        tk.Label(hero, text=TR("version"), font=("Courier", 10),
                 bg=t["bg2"], fg=t["fg2"]).pack()

        for title, body in [
            ("✉ Отправка кодов", "Коды подтверждения отправляются на email через Яндекс.Почту."),
            ("🗺 Карта", "Офлайн-карта с заправками России. Данные можно обновлять."),
            ("🔒 Безопасность", "Пароли шифруются (SHA-256). Данные локально."),
        ]:
            sec = tk.Frame(pad, bg=t["bg2"])
            sec.pack(fill="x", pady=6, ipadx=12, ipady=8)
            tk.Label(sec, text=title, font=("Georgia", 12, "bold"),
                     bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=12)
            tk.Label(sec, text=body, font=("Courier", 10), bg=t["bg2"],
                     fg=t["fg2"], wraplength=600, justify="left").pack(anchor="w", padx=12)

        tk.Label(pad, text=TR("footer"), font=("Courier", 9),
                 bg=t["bg"], fg=t["fg2"]).pack(pady=12)

    # ── SETTINGS ──────────────────────────────────────────────────────────────
    def _build_settings(self):
        t = T()
        f = self.sections["settings"]
        for w in f.winfo_children(): w.destroy()
        inner = make_scrollable(f)
        pad = tk.Frame(inner, bg=t["bg"])
        pad.pack(fill="both", expand=True, padx=48, pady=24)

        tk.Label(pad, text=TR("settings_title"), font=("Georgia", 16, "bold"),
                 bg=t["bg"], fg=t["fg"]).pack(anchor="w", pady=(0,16))

        # Theme
        tc = tk.Frame(pad, bg=t["bg2"], pady=12)
        tc.pack(fill="x", pady=6)
        tk.Label(tc, text=TR("theme"), font=("Georgia", 12, "bold"), bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=16)
        tr = tk.Frame(tc, bg=t["bg2"]); tr.pack(padx=16, pady=8, anchor="w")
        tk.Button(tr, text=TR("dark_theme"), font=("Courier", 10), bg="#1f2937", fg="#e5e7eb",
                  bd=0, padx=16, pady=6, command=lambda: self._apply_theme("dark")).pack(side="left", padx=(0,8))
        tk.Button(tr, text=TR("light_theme"), font=("Courier", 10), bg="#e2e8f0", fg="#1a202c",
                  bd=0, padx=16, pady=6, command=lambda: self._apply_theme("light")).pack(side="left")

        # Currency
        cc = tk.Frame(pad, bg=t["bg2"], pady=12)
        cc.pack(fill="x", pady=6)
        tk.Label(cc, text=TR("currency_setting"), font=("Georgia", 12, "bold"), bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=16)
        cv = tk.StringVar(value=current_currency)
        ttk.Combobox(cc, textvariable=cv, values=list(CURRENCIES.keys()),
                     state="readonly", width=12, font=("Courier", 10)).pack(anchor="w", padx=16, pady=6)
        cv.trace("w", lambda *a: self._apply_currency(cv.get()))

    def _apply_theme(self, name):
        global current_theme
        current_theme = name
        self._save_settings()
        self._rebuild_ui()
        self._show("settings")

    def _apply_currency(self, val):
        global current_currency
        current_currency = val
        self._save_settings()
        if hasattr(self, 'calc_form_frame'):
            self._build_calc_form()


if __name__ == "__main__":
    app = FuelApp()
    app.mainloop()
