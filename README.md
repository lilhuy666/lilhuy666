import tkinter as tk
from tkinter import ttk, messagebox, filedialog
import json
import os
import re
import hashlib
import shutil
from datetime import datetime, timedelta
import math

try:
    from PIL import Image, ImageTk, ImageDraw
except ImportError:
    import subprocess, sys
    subprocess.check_call([sys.executable, "-m", "pip", "install", "Pillow"])
    from PIL import Image, ImageTk, ImageDraw

try:
    import tkintermapview
    MAP_AVAILABLE = True
except ImportError:
    try:
        import subprocess, sys
        subprocess.check_call([sys.executable, "-m", "pip", "install", "tkintermapview"])
        import tkintermapview
        MAP_AVAILABLE = True
    except Exception:
        MAP_AVAILABLE = False

# ─── Data storage ──────────────────────────────────────────────────────────────
DATA_FILE = os.path.join(os.path.expanduser("~"), ".fuel_calc_data.json")
PHOTO_DIR = os.path.join(os.path.expanduser("~"), ".fuel_calc_photos")
os.makedirs(PHOTO_DIR, exist_ok=True)

def load_data():
    if os.path.exists(DATA_FILE):
        with open(DATA_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    return {"users": {}, "current_user": None, "settings": {}}

def save_data(data):
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

def hash_pw(pw):
    return hashlib.sha256(pw.encode()).hexdigest()

def is_valid_email(email):
    return re.match(r"^[^@\s]+@[^@\s]+\.[^@\s]+$", email) is not None

# ─── Currencies ────────────────────────────────────────────────────────────────
CURRENCIES = {
    "₽ RUB": "₽",
    "$ USD": "$",
    "€ EUR": "€",
    "£ GBP": "£",
    "₴ UAH": "₴",
    "₸ KZT": "₸",
    "¥ JPY": "¥",
    "₩ KRW": "₩",
}

# ─── Translations ──────────────────────────────────────────────────────────────
TRANSLATIONS = {
    "ru": {
        "app_title": "Калькулятор расхода топлива",
        "app_subtitle": "Расчёт расхода топлива для вашего автомобиля",
        "calculator": "Калькулятор",
        "profile": "Профиль",
        "history": "История",
        "gas_stations": "Заправки",
        "about": "О приложении",
        "settings": "Настройки",
        "no_car": "— Без авто —",
        "car": "Автомобиль",
        "quick_stats": "Быстрая статистика",
        "avg_consumption": "💧  Средний расход",
        "distance": "Расстояние",
        "fuel": "Топлива",
        "cost": "Стоимость",
        "calculate": "🧮  Рассчитать",
        "clear": "🔄  Очистить",
        "fuel_consumption": "💧  Расход топлива",
        "trip_cost": "💳  Стоимость поездки",
        "distance_label": "Расстояние",
        "fuel_used": "Израсходовано топлива",
        "fuel_price": "Цена топлива за литр",
        "avg_consumption_label": "Средний расход",
        "calc_mode_consumption": "Средний расход на 100 км",
        "calc_mode_cost": "Стоимость поездки",
        "login": "Войти",
        "register": "Регистрация",
        "create_account": "Создать аккаунт",
        "login_account": "Вход в аккаунт",
        "email": "Email",
        "password": "Пароль (мин. 6 символов)",
        "repeat_password": "Повторите пароль",
        "fill_all": "Заполните все поля",
        "invalid_email": "Введите корректный email",
        "password_short": "Пароль должен содержать минимум 6 символов",
        "passwords_mismatch": "Пароли не совпадают",
        "user_exists": "Пользователь уже существует",
        "wrong_credentials": "Неверный email или пароль",
        "logout": "Выйти",
        "delete_account": "🗑 Удалить аккаунт",
        "change_password": "🔒  Сменить пароль",
        "new_password": "Новый пароль (мин. 6 символов)",
        "repeat_new_password": "Повторите новый пароль",
        "change_password_btn": "Сменить пароль",
        "password_changed": "✓ Пароль изменён",
        "my_cars": "🚗  Мои автомобили",
        "add_car": "+ Добавить авто",
        "no_cars": "Автомобили не добавлены",
        "delete": "🗑 Удалить",
        "car_name": "Название автомобиля",
        "no_photo": "Нет фото",
        "choose_photo": "📷  Выбрать фото",
        "photo_not_chosen": "Фото не выбрано",
        "save": "💾  Сохранить",
        "enter_name": "Введите название",
        "add_car_title": "Добавить автомобиль",
        "history_title": "📋  История расчётов",
        "login_for_history": "Войдите в аккаунт для просмотра истории.",
        "clear_all": "🗑  Очистить всё",
        "history_empty": "История пуста",
        "copy": "📋 Копировать",
        "copied": "Скопировано",
        "copied_msg": "Расчёт скопирован в буфер обмена!",
        "delete_entry": "✕ Удалить",
        "delete_confirm": "Удалить?",
        "delete_car_confirm": "Удалить этот автомобиль?",
        "clear_history_confirm": "Очистить?",
        "clear_history_msg": "Удалить всю историю?",
        "delete_account_confirm": "Удалить аккаунт?",
        "delete_account_msg": "Вы уверены? Все данные (история, автомобили) будут удалены безвозвратно.",
        "settings_title": "⚙  Настройки",
        "theme": "🎨  Тема оформления",
        "theme_desc": "Выберите цветовую схему приложения",
        "dark_theme": "🌙  Тёмная",
        "light_theme": "☀️  Светлая",
        "current_theme": "Текущая тема:",
        "dark": "Тёмная 🌙",
        "light": "Светлая ☀️",
        "currency_setting": "💱  Валюта",
        "currency_desc": "Выберите валюту для расчётов",
        "language_setting": "🌐  Язык / Language",
        "language_desc": "Выберите язык интерфейса",
        "current": "Текущее:",
        "error": "Ошибка",
        "enter_numbers": "Введите корректные числа!",
        "distance_positive": "Расстояние должно быть больше 0!",
        "distance_positive2": "Расстояние и расход должны быть больше 0!",
        "fuel_chart_title": "График расхода топлива",
        "chart_days": "Дней:",
        "no_history_chart": "Нет данных для графика",
        "login_for_chart": "Войдите для просмотра графика",
        "per_100": "л/100",
        "distance_unit": "км",
        "about_title": "О приложении",
        "version": "Версия 2.3  •  2024",
        "about_section1_title": "🎯  О приложении",
        "about_section1": "Это профессиональное приложение для расчёта расхода топлива вашего автомобиля. Создано для водителей, которые хотят следить за эффективностью транспортного средства и планировать затраты на топливо.",
        "about_section2_title": "✨  Возможности",
        "about_section2": "• Точный расчёт расхода топлива в л/100 км\n• Расчёт стоимости поездки с выбором валюты\n• Хранение нескольких автомобилей с фотографиями\n• Полная история всех расчётов с возможностью копирования\n• Личный профиль с регистрацией по email\n• Светлая и тёмная темы оформления\n• Поиск заправок на встроенной карте с расчётом расстояния",
        "about_section3_title": "🚗  Как использовать",
        "about_section3": "Режим «Средний расход»:\n1. Введите пройденное расстояние\n2. Укажите израсходованное топливо в литрах\n3. Введите цену топлива за литр\n4. Нажмите «Рассчитать»\n\nРежим «Стоимость поездки»:\n1. Введите расстояние поездки\n2. Укажите средний расход автомобиля\n3. Введите цену топлива за литр\n4. Нажмите «Рассчитать»",
        "about_section4_title": "🔒  Безопасность",
        "about_section4": "Регистрация только по email. Пароли зашифрованы (SHA-256). Все данные хранятся локально на устройстве. Никакие данные не передаются на серверы.",
        "about_section5_title": "📊  Интерпретация результатов",
        "about_section5": "🟢  До 8 л/100 км  — экономичный расход\n🟡  8–12 л/100 км  — средний расход\n🔴  Более 12 л/100 км — высокий расход",
        "footer": "© 2024 Калькулятор расхода топлива. Все права защищены.",
        "fuel_unit": "л",
        "price_unit": "за л",
        "gs_title": "⛽  Заправки",
        "gs_city": "Город / адрес:",
        "gs_search": "🔍  Найти заправки",
        "gs_to_calc": "🧮  В калькулятор",
        "gs_no_car": "Выберите авто для автозаполнения",
        "gs_select_station": "Кликните на маркер заправки на карте",
        "gs_login_hint": "Войдите для доступа к данным авто",
        "gs_hint": "Введите город или адрес и нажмите «Найти заправки».\nЗатем кликните на маркер заправки для выбора и расчёта расстояния.",
        "gs_fuel_price_hint": "Цена топлива на заправке:",
        "gs_distance_to": "Расстояние до заправки",
        "gs_enter_distance": "Расстояние до заправки (км):",
        "gs_send_to_calc": "Отправить в калькулятор",
        "gs_avg_from_car": "Средний расход авто",
        "gs_no_avg": "Нет данных о расходе",
        "gs_selected_station": "Выбранная заправка:",
        "gs_calc_distance": "📍 Расстояние рассчитано по карте",
        "gs_click_hint": "Кликните на маркер ⛽ на карте для выбора заправки",
        "gs_map_search_hint": "Поиск заправок на карте...",
        "gs_no_map": "Установите tkintermapview для карты:\npip install tkintermapview",
        "countries": [
            "Россия", "США", "Германия", "Франция", "Великобритания",
            "Украина", "Казахстан", "Беларусь", "Польша", "Италия",
            "Испания", "Нидерланды", "Швеция", "Финляндия", "Норвегия",
            "Австрия", "Швейцария", "Чехия", "Венгрия", "Румыния",
            "Турция", "ОАЭ", "Китай", "Япония", "Австралия", "Канада"
        ],
    },
    "en": {
        "app_title": "Fuel Consumption Calculator",
        "app_subtitle": "Calculate fuel consumption for your vehicle",
        "calculator": "Calculator",
        "profile": "Profile",
        "history": "History",
        "gas_stations": "Gas Stations",
        "about": "About",
        "settings": "Settings",
        "no_car": "— No car —",
        "car": "Vehicle",
        "quick_stats": "Quick Stats",
        "avg_consumption": "💧  Avg. consumption",
        "distance": "Distance",
        "fuel": "Fuel",
        "cost": "Cost",
        "calculate": "🧮  Calculate",
        "clear": "🔄  Clear",
        "fuel_consumption": "💧  Fuel consumption",
        "trip_cost": "💳  Trip cost",
        "distance_label": "Distance",
        "fuel_used": "Fuel used",
        "fuel_price": "Fuel price per litre",
        "avg_consumption_label": "Avg. consumption",
        "calc_mode_consumption": "Avg. consumption per 100 km",
        "calc_mode_cost": "Trip cost",
        "login": "Login",
        "register": "Register",
        "create_account": "Create Account",
        "login_account": "Sign In",
        "email": "Email",
        "password": "Password (min. 6 chars)",
        "repeat_password": "Repeat password",
        "fill_all": "Fill in all fields",
        "invalid_email": "Enter a valid email",
        "password_short": "Password must be at least 6 characters",
        "passwords_mismatch": "Passwords do not match",
        "user_exists": "User already exists",
        "wrong_credentials": "Invalid email or password",
        "logout": "Logout",
        "delete_account": "🗑 Delete account",
        "change_password": "🔒  Change password",
        "new_password": "New password (min. 6 chars)",
        "repeat_new_password": "Repeat new password",
        "change_password_btn": "Change password",
        "password_changed": "✓ Password changed",
        "my_cars": "🚗  My vehicles",
        "add_car": "+ Add vehicle",
        "no_cars": "No vehicles added",
        "delete": "🗑 Delete",
        "car_name": "Vehicle name",
        "no_photo": "No photo",
        "choose_photo": "📷  Choose photo",
        "photo_not_chosen": "No photo selected",
        "save": "💾  Save",
        "enter_name": "Enter name",
        "add_car_title": "Add vehicle",
        "history_title": "📋  Calculation history",
        "login_for_history": "Log in to view history.",
        "clear_all": "🗑  Clear all",
        "history_empty": "History is empty",
        "copy": "📋 Copy",
        "copied": "Copied",
        "copied_msg": "Calculation copied to clipboard!",
        "delete_entry": "✕ Delete",
        "delete_confirm": "Delete?",
        "delete_car_confirm": "Delete this vehicle?",
        "clear_history_confirm": "Clear?",
        "clear_history_msg": "Delete all history?",
        "delete_account_confirm": "Delete account?",
        "delete_account_msg": "Are you sure? All data (history, vehicles) will be permanently deleted.",
        "settings_title": "⚙  Settings",
        "theme": "🎨  Theme",
        "theme_desc": "Choose the color scheme of the app",
        "dark_theme": "🌙  Dark",
        "light_theme": "☀️  Light",
        "current_theme": "Current theme:",
        "dark": "Dark 🌙",
        "light": "Light ☀️",
        "currency_setting": "💱  Currency",
        "currency_desc": "Choose currency for calculations",
        "language_setting": "🌐  Language / Язык",
        "language_desc": "Choose interface language",
        "current": "Current:",
        "error": "Error",
        "enter_numbers": "Enter valid numbers!",
        "distance_positive": "Distance must be greater than 0!",
        "distance_positive2": "Distance and consumption must be greater than 0!",
        "fuel_chart_title": "Fuel consumption chart",
        "chart_days": "Days:",
        "no_history_chart": "No data for chart",
        "login_for_chart": "Log in to view chart",
        "per_100": "L/100",
        "distance_unit": "km",
        "about_title": "About",
        "version": "Version 2.3  •  2024",
        "about_section1_title": "🎯  About",
        "about_section1": "A professional application for calculating your vehicle's fuel consumption. Built for drivers who want to track their vehicle's efficiency and plan fuel costs.",
        "about_section2_title": "✨  Features",
        "about_section2": "• Accurate fuel consumption calculation\n• Trip cost calculation with currency selection\n• Store multiple vehicles with photos\n• Full calculation history with copy option\n• Personal profile with email registration\n• Light and dark themes\n• Language selection\n• Gas station map with distance calculation",
        "about_section3_title": "🚗  How to use",
        "about_section3": "Consumption mode:\n1. Enter distance travelled\n2. Enter fuel used in litres\n3. Enter fuel price per litre\n4. Press Calculate\n\nTrip cost mode:\n1. Enter trip distance\n2. Enter vehicle avg. consumption\n3. Enter fuel price per litre\n4. Press Calculate",
        "about_section4_title": "🔒  Security",
        "about_section4": "Email-only registration. Passwords are encrypted (SHA-256). All data is stored locally on your device. No data is sent to any server.",
        "about_section5_title": "📊  Result interpretation",
        "about_section5": "🟢  Up to 8 L/100 km  — economical\n🟡  8–12 L/100 km  — average\n🔴  Over 12 L/100 km  — high consumption",
        "footer": "© 2024 Fuel Consumption Calculator. All rights reserved.",
        "fuel_unit": "L",
        "price_unit": "per L",
        "gs_title": "⛽  Gas Stations",
        "gs_city": "City / address:",
        "gs_search": "🔍  Find Gas Stations",
        "gs_to_calc": "🧮  To Calculator",
        "gs_no_car": "Select a vehicle for autofill",
        "gs_select_station": "Click on a gas station marker on the map",
        "gs_login_hint": "Log in to access vehicle data",
        "gs_hint": "Enter a city or address and click Find Gas Stations.\nThen click on a station marker to select it and calculate the distance.",
        "gs_fuel_price_hint": "Fuel price at the station:",
        "gs_distance_to": "Distance to station",
        "gs_enter_distance": "Distance to station (km):",
        "gs_send_to_calc": "Send to Calculator",
        "gs_avg_from_car": "Vehicle avg. consumption",
        "gs_no_avg": "No consumption data",
        "gs_selected_station": "Selected station:",
        "gs_calc_distance": "📍 Distance calculated from map",
        "gs_click_hint": "Click on a ⛽ marker on the map to select a station",
        "gs_map_search_hint": "Searching for gas stations on map...",
        "gs_no_map": "Install tkintermapview for map:\npip install tkintermapview",
        "countries": [
            "Russia", "USA", "Germany", "France", "United Kingdom",
            "Ukraine", "Kazakhstan", "Belarus", "Poland", "Italy",
            "Spain", "Netherlands", "Sweden", "Finland", "Norway",
            "Austria", "Switzerland", "Czech Republic", "Hungary", "Romania",
            "Turkey", "UAE", "China", "Japan", "Australia", "Canada"
        ],
    }
}

# ─── Theme ─────────────────────────────────────────────────────────────────────
THEMES = {
    "dark": {
        "bg":        "#0d1117",
        "bg2":       "#161b22",
        "bg3":       "#21262d",
        "fg":        "#e6edf3",
        "fg2":       "#8b949e",
        "accent":    "#58a6ff",
        "green":     "#3fb950",
        "yellow":    "#d29922",
        "red":       "#f85149",
        "border":    "#30363d",
        "btn":       "#238636",
        "btn_hover": "#2ea043",
        "btn2":      "#21262d",
        "btn2_hover":"#30363d",
        "input_bg":  "#0d1117",
        "chart_bg":  "#161b22",
        "chart_line":"#58a6ff",
        "chart_fill":"#1c2d3f",
        "chart_grid":"#21262d",
    },
    "light": {
        "bg":        "#f0f4f8",
        "bg2":       "#ffffff",
        "bg3":       "#e2e8f0",
        "fg":        "#1a202c",
        "fg2":       "#718096",
        "accent":    "#3182ce",
        "green":     "#38a169",
        "yellow":    "#d69e2e",
        "red":       "#e53e3e",
        "border":    "#cbd5e0",
        "btn":       "#3182ce",
        "btn_hover": "#2b6cb0",
        "btn2":      "#e2e8f0",
        "btn2_hover":"#cbd5e0",
        "input_bg":  "#ffffff",
        "chart_bg":  "#ffffff",
        "chart_line":"#3182ce",
        "chart_fill":"#ebf4ff",
        "chart_grid":"#e2e8f0",
    }
}

current_theme    = "dark"
current_language = "ru"
current_currency = "₽ RUB"

def T():
    return THEMES[current_theme]

def TR(key):
    lang = TRANSLATIONS.get(current_language, TRANSLATIONS["ru"])
    return lang.get(key, TRANSLATIONS["ru"].get(key, key))

def get_currency_symbol():
    return CURRENCIES.get(current_currency, "₽")

# ─── Scroll helper ─────────────────────────────────────────────────────────────
def bind_mousewheel(widget, canvas):
    """Bind mouse wheel scrolling to canvas via any child widget."""
    def _on_mousewheel(event):
        if event.num == 4:
            canvas.yview_scroll(-1, "units")
        elif event.num == 5:
            canvas.yview_scroll(1, "units")
        else:
            canvas.yview_scroll(int(-1 * (event.delta / 120)), "units")

    def _bind_to_mousewheel(e):
        canvas.bind_all("<MouseWheel>", _on_mousewheel)
        canvas.bind_all("<Button-4>", _on_mousewheel)
        canvas.bind_all("<Button-5>", _on_mousewheel)

    def _unbind_from_mousewheel(e):
        canvas.unbind_all("<MouseWheel>")
        canvas.unbind_all("<Button-4>")
        canvas.unbind_all("<Button-5>")

    widget.bind("<Enter>", _bind_to_mousewheel)
    widget.bind("<Leave>", _unbind_from_mousewheel)

# ─── Haversine distance ────────────────────────────────────────────────────────
def haversine_km(lat1, lon1, lat2, lon2):
    R = 6371.0
    dlat = math.radians(lat2 - lat1)
    dlon = math.radians(lon2 - lon1)
    a = math.sin(dlat/2)**2 + math.cos(math.radians(lat1)) * math.cos(math.radians(lat2)) * math.sin(dlon/2)**2
    return R * 2 * math.atan2(math.sqrt(a), math.sqrt(1 - a))

# ─── Helper: placeholder image ─────────────────────────────────────────────────
def make_placeholder(w=300, h=180, text=""):
    t = T()
    img = Image.new("RGB", (w, h), t["bg3"])
    draw = ImageDraw.Draw(img)
    draw.rectangle([0, 0, w-1, h-1], outline=t["border"], width=2)
    if text:
        draw.text((w//2 - 25, h//2 - 8), text, fill=t["fg2"])
    return img

def load_car_image(path, w=300, h=180):
    try:
        img = Image.open(path).convert("RGB")
        src_w, src_h = img.size
        src_ratio = src_w / src_h
        dst_ratio = w / h
        if src_ratio > dst_ratio:
            new_h = h
            new_w = int(src_w * h / src_h)
        else:
            new_w = w
            new_h = int(src_h * w / src_w)
        img = img.resize((new_w, new_h), Image.LANCZOS)
        left = (new_w - w) // 2
        top  = (new_h - h) // 2
        img = img.crop((left, top, left + w, top + h))
        return img
    except Exception:
        return make_placeholder(w, h)

# ─── Main Application ──────────────────────────────────────────────────────────
class FuelApp(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title(TR("app_title"))
        self.geometry("1100x820")
        self.minsize(900, 680)
        self.resizable(True, True)

        self.data = load_data()
        self.current_user = self.data.get("current_user")

        settings = self.data.get("settings", {})
        global current_theme, current_language, current_currency
        current_theme    = settings.get("theme",    "dark")
        current_language = settings.get("language", "ru")
        current_currency = settings.get("currency", "₽ RUB")

        self._photo_refs   = {}
        self._calc_mode    = tk.StringVar(value="consumption")

        # Map state
        self._map_markers = []
        self._map_user_marker = None
        self._map_selected_station = None
        self._map_selected_lat = None
        self._map_selected_lon = None
        self._map_user_lat = None
        self._map_user_lon = None

        self._build_ui()

    def _save_settings(self):
        self.data.setdefault("settings", {})
        self.data["settings"]["theme"]    = current_theme
        self.data["settings"]["language"] = current_language
        self.data["settings"]["currency"] = current_currency
        save_data(self.data)

    # ── Layout ──────────────────────────────────────────────────────────────────
    def _build_ui(self):
        t = T()
        self.configure(bg=t["bg"])

        self.header = tk.Frame(self, bg=t["bg2"], pady=12)
        self.header.pack(fill="x", side="top")

        self.menu_btn = tk.Button(self.header, text="☰", font=("Courier", 16, "bold"),
                                  bg=t["bg3"], fg=t["fg"], bd=0, padx=10, pady=4,
                                  cursor="hand2", command=self._toggle_sidebar)
        self.menu_btn.pack(side="left", padx=16)

        tk.Label(self.header, text=TR("app_title"),
                 font=("Georgia", 18, "bold"), bg=t["bg2"], fg=t["fg"]).pack(side="left", padx=8)

        tk.Label(self.header, text=TR("app_subtitle"),
                 font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(side="left", padx=4)

        self.settings_btn = tk.Button(self.header, text="⚙", font=("Courier", 16),
                                       bg=t["bg3"], fg=t["fg"], bd=0, padx=10, pady=4,
                                       cursor="hand2", command=lambda: self._show_section("settings"))
        self.settings_btn.pack(side="right", padx=16)

        self.body = tk.Frame(self, bg=t["bg"])
        self.body.pack(fill="both", expand=True)

        self.sidebar = tk.Frame(self.body, bg=t["bg2"], width=200)
        self.sidebar.pack_propagate(False)
        self._sidebar_visible = False
        self._build_sidebar()

        self.content = tk.Frame(self.body, bg=t["bg"])
        self.content.pack(side="left", fill="both", expand=True)

        self.sections = {}
        for name in ["calculator", "profile", "history", "gas_stations", "about", "settings"]:
            frame = tk.Frame(self.content, bg=t["bg"])
            self.sections[name] = frame

        self._build_calculator()
        self._build_profile()
        self._build_history()
        self._build_gas_stations()
        self._build_about()
        self._build_settings()

        self._show_section("calculator")

    def _rebuild_ui(self):
        self.header.destroy()
        self.body.destroy()
        self._photo_refs.clear()
        self._build_ui()

    def _build_sidebar(self):
        t = T()
        for w in self.sidebar.winfo_children():
            w.destroy()
        items = [
            (TR("calculator"),   "calculator"),
            (TR("profile"),      "profile"),
            (TR("history"),      "history"),
            (TR("gas_stations"), "gas_stations"),
            (TR("about"),        "about"),
        ]
        tk.Frame(self.sidebar, bg=t["bg2"], height=20).pack()
        for label, key in items:
            icon = {
                "calculator":   "🧮",
                "profile":      "👤",
                "history":      "📋",
                "gas_stations": "⛽",
                "about":        "ℹ️"
            }.get(key, "")
            btn = tk.Button(self.sidebar, text=f"{icon}  {label}", font=("Courier", 12),
                            bg=t["bg2"], fg=t["fg"], bd=0, padx=20, pady=12,
                            anchor="w", cursor="hand2",
                            activebackground=t["bg3"], activeforeground=t["fg"],
                            command=lambda k=key: self._show_section(k))
            btn.pack(fill="x")
            btn.bind("<Enter>", lambda e, b=btn: b.configure(bg=T()["bg3"]))
            btn.bind("<Leave>", lambda e, b=btn: b.configure(bg=T()["bg2"]))
        tk.Frame(self.sidebar, bg=t["border"], height=1).pack(fill="x", padx=16, pady=8)

    def _toggle_sidebar(self):
        if self._sidebar_visible:
            self.sidebar.pack_forget()
        else:
            self.sidebar.pack(side="left", fill="y", before=self.content)
        self._sidebar_visible = not self._sidebar_visible

    def _show_section(self, name):
        for k, f in self.sections.items():
            f.pack_forget()
        self.sections[name].pack(fill="both", expand=True)
        if name == "history":
            self._refresh_history()
        if name == "profile":
            self._refresh_profile()
        if name == "gas_stations":
            self._refresh_gas_stations()

    # ── Calculator ──────────────────────────────────────────────────────────────
    def _build_calculator(self):
        t = T()
        f = self.sections["calculator"]
        for w in f.winfo_children(): w.destroy()

        # Outer scrollable canvas
        outer_canvas = tk.Canvas(f, bg=t["bg"], highlightthickness=0)
        outer_scroll = ttk.Scrollbar(f, orient="vertical", command=outer_canvas.yview)
        outer_canvas.configure(yscrollcommand=outer_scroll.set)
        outer_scroll.pack(side="right", fill="y")
        outer_canvas.pack(side="left", fill="both", expand=True)

        outer = tk.Frame(outer_canvas, bg=t["bg"])
        win_id = outer_canvas.create_window((0, 0), window=outer, anchor="nw")
        outer.bind("<Configure>", lambda e: outer_canvas.configure(scrollregion=outer_canvas.bbox("all")))
        outer_canvas.bind("<Configure>", lambda e: outer_canvas.itemconfig(win_id, width=e.width))
        bind_mousewheel(outer, outer_canvas)

        inner = tk.Frame(outer, bg=t["bg"])
        inner.pack(fill="both", expand=True, padx=24, pady=16)

        top = tk.Frame(inner, bg=t["bg"])
        top.pack(fill="x")

        # Left: car image + selector
        left = tk.Frame(top, bg=t["bg2"], bd=0, relief="flat")
        left.pack(side="left", fill="y", padx=(0, 16))

        img_container = tk.Frame(left, bg=t["bg2"], width=300, height=180)
        img_container.pack_propagate(False)
        img_container.pack(padx=0, pady=0)

        self.calc_car_label = tk.Label(img_container, bg=t["bg2"])
        self.calc_car_label.place(relx=0.5, rely=0.5, anchor="center")
        self._set_calc_car_image(None)

        tk.Label(left, text=TR("car") + ":", font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(pady=(8,0))
        self.calc_car_var = tk.StringVar(value=TR("no_car"))
        self.calc_car_combo = ttk.Combobox(left, textvariable=self.calc_car_var,
                                            state="readonly", width=22, font=("Courier", 10))
        self.calc_car_combo.pack(padx=12, pady=(2, 8))
        self.calc_car_combo.bind("<<ComboboxSelected>>", self._on_car_selected)
        self._refresh_car_combo()

        # Quick stats
        stats_frame = tk.Frame(left, bg=t["bg3"])
        stats_frame.pack(fill="x", padx=8, pady=8, ipady=8)
        tk.Label(stats_frame, text=TR("quick_stats"),
                 font=("Georgia", 12, "bold"), bg=t["bg3"], fg=t["fg"]).pack(anchor="w", padx=12, pady=(8,4))

        self.stat_labels = {}
        stats = [
            ("avg",  TR("avg_consumption"),  "— л/100 км", t["green"]),
            ("dist", f"📏  {TR('distance')}",  "— км",       t["accent"]),
            ("fuel", f"⛽  {TR('fuel')}",       "— л",        t["yellow"]),
            ("cost", f"💳  {TR('cost')}",       "—",          t["red"]),
        ]
        for key, label, default, color in stats:
            row = tk.Frame(stats_frame, bg=t["bg3"])
            row.pack(fill="x", padx=12, pady=3)
            tk.Label(row, text=label, font=("Courier", 10), bg=t["bg3"], fg=t["fg2"],
                     width=22, anchor="w").pack(side="left")
            lbl = tk.Label(row, text=default, font=("Courier", 10, "bold"),
                           bg=t["bg3"], fg=color)
            lbl.pack(side="right")
            self.stat_labels[key] = lbl

        # Right: calculator form
        right = tk.Frame(top, bg=t["bg2"])
        right.pack(side="left", fill="both", expand=True)

        tk.Label(right, text=TR("calculator"), font=("Georgia", 15, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=20, pady=(16, 4))

        mode_frame = tk.Frame(right, bg=t["bg2"])
        mode_frame.pack(fill="x", padx=20, pady=(0, 8))

        self._calc_mode = tk.StringVar(value="consumption")
        mode_btn1 = tk.Radiobutton(mode_frame, text=TR("calc_mode_consumption"),
                                    variable=self._calc_mode, value="consumption",
                                    font=("Courier", 10), bg=t["bg2"], fg=t["fg"],
                                    selectcolor=t["bg3"], activebackground=t["bg2"],
                                    cursor="hand2", command=self._on_mode_changed)
        mode_btn1.pack(side="left", padx=(0, 16))
        mode_btn2 = tk.Radiobutton(mode_frame, text=TR("calc_mode_cost"),
                                    variable=self._calc_mode, value="cost",
                                    font=("Courier", 10), bg=t["bg2"], fg=t["fg"],
                                    selectcolor=t["bg3"], activebackground=t["bg2"],
                                    cursor="hand2", command=self._on_mode_changed)
        mode_btn2.pack(side="left")

        self.calc_form_frame = tk.Frame(right, bg=t["bg2"])
        self.calc_form_frame.pack(fill="x", padx=20)

        self.dist_var  = tk.StringVar(value="")
        self.fuel_var  = tk.StringVar(value="")
        self.price_var = tk.StringVar(value="")
        self.avg_var   = tk.StringVar(value="")

        self._build_calc_form()

        btn_row = tk.Frame(right, bg=t["bg2"])
        btn_row.pack(fill="x", padx=20, pady=8)
        tk.Button(btn_row, text=TR("calculate"),
                  font=("Georgia", 12, "bold"), bg=t["btn"], fg="white", bd=0, pady=10,
                  cursor="hand2", command=self._calculate).pack(side="left", fill="x", expand=True, padx=(0, 8))
        tk.Button(btn_row, text=TR("clear"),
                  font=("Courier", 11), bg=t["btn2"], fg=t["fg"], bd=0, pady=10,
                  cursor="hand2", command=self._clear_calc).pack(side="left", fill="x", expand=True)

        self.result_frame = tk.Frame(right, bg=t["bg3"])
        self.result_frame.pack(fill="x", padx=20, pady=(4, 8))

        res_left = tk.Frame(self.result_frame, bg=t["bg3"])
        res_left.pack(side="left", fill="x", expand=True, padx=16, pady=12)
        tk.Label(res_left, text=TR("fuel_consumption"), font=("Courier", 10), bg=t["bg3"], fg=t["fg2"]).pack(anchor="w")
        self.result_consumption_label = tk.Label(res_left, text="— л/100 км",
                                                  font=("Georgia", 18, "bold"), bg=t["bg3"], fg=t["green"])
        self.result_consumption_label.pack(anchor="w")

        res_right = tk.Frame(self.result_frame, bg=t["bg3"])
        res_right.pack(side="left", fill="x", expand=True, padx=16, pady=12)
        tk.Label(res_right, text=TR("trip_cost"), font=("Courier", 10), bg=t["bg3"], fg=t["fg2"]).pack(anchor="w")
        self.result_cost_label = tk.Label(res_right, text="—",
                                           font=("Georgia", 18, "bold"), bg=t["bg3"], fg=t["fg"])
        self.result_cost_label.pack(anchor="w")

        # ── Fuel chart (larger) ──
        chart_outer = tk.Frame(inner, bg=t["bg2"])
        chart_outer.pack(fill="x", pady=(12, 0))

        chart_header = tk.Frame(chart_outer, bg=t["bg2"])
        chart_header.pack(fill="x", padx=16, pady=(10, 4))
        tk.Label(chart_header, text=TR("fuel_chart_title"), font=("Georgia", 13, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack(side="left")

        days_frame = tk.Frame(chart_header, bg=t["bg2"])
        days_frame.pack(side="right")
        tk.Label(days_frame, text=TR("chart_days"), font=("Courier", 10),
                 bg=t["bg2"], fg=t["fg2"]).pack(side="left", padx=(0, 6))
        self.chart_days_var = tk.StringVar(value="30")
        days_combo = ttk.Combobox(days_frame, textvariable=self.chart_days_var,
                                   values=["7", "14", "30", "60", "90"],
                                   state="readonly", width=5, font=("Courier", 10))
        days_combo.pack(side="left")
        days_combo.bind("<<ComboboxSelected>>", lambda e: self._draw_fuel_chart())

        # Increased chart height from 160 to 280
        self.chart_canvas = tk.Canvas(chart_outer, bg=t["chart_bg"], height=280,
                                       highlightthickness=1, highlightbackground=t["border"])
        self.chart_canvas.pack(fill="x", padx=16, pady=(0, 16))
        self.chart_canvas.bind("<Configure>", lambda e: self._draw_fuel_chart())
        self.chart_canvas.bind("<Motion>", self._on_chart_hover)
        self.chart_canvas.bind("<Leave>", self._on_chart_leave)
        self._chart_tooltip = None
        self._chart_data_points = []

        self.after(100, self._draw_fuel_chart)

    def _build_calc_form(self):
        t = T()
        for w in self.calc_form_frame.winfo_children(): w.destroy()
        mode = self._calc_mode.get() if hasattr(self, '_calc_mode') else "consumption"

        def make_entry_row(parent, label_text, var, unit_text, icon):
            tk.Label(parent, text=label_text, font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", pady=(8,2))
            row = tk.Frame(parent, bg=t["input_bg"], bd=1, relief="solid")
            row.pack(fill="x", pady=2)
            tk.Label(row, text=icon, font=("Courier", 13), bg=t["input_bg"], fg=t["fg2"], padx=8).pack(side="left")
            tk.Entry(row, textvariable=var, font=("Courier", 13),
                     bg=t["input_bg"], fg=t["fg"], insertbackground=t["fg"],
                     bd=0, relief="flat").pack(side="left", fill="x", expand=True, pady=8)
            tk.Label(row, text=unit_text, font=("Courier", 10), bg=t["input_bg"], fg=t["fg2"], padx=8).pack(side="right")

        make_entry_row(self.calc_form_frame, TR("distance_label"), self.dist_var, "км", "📏")

        if mode == "consumption":
            make_entry_row(self.calc_form_frame, TR("fuel_used"), self.fuel_var, TR("fuel_unit"), "⛽")
            make_entry_row(self.calc_form_frame, TR("fuel_price"), self.price_var, get_currency_symbol() + "/" + TR("fuel_unit"), "💰")
        else:
            make_entry_row(self.calc_form_frame, TR("avg_consumption_label"), self.avg_var, "л/100 км", "💧")
            make_entry_row(self.calc_form_frame, TR("fuel_price"), self.price_var, get_currency_symbol() + "/" + TR("fuel_unit"), "💰")

    def _on_mode_changed(self):
        self._build_calc_form()

    def _set_calc_car_image(self, photo_path):
        t = T()
        if photo_path and os.path.exists(photo_path):
            img = load_car_image(photo_path, 300, 180)
        else:
            img = make_placeholder(300, 180)
        photo = ImageTk.PhotoImage(img)
        self._photo_refs["calc_car"] = photo
        self.calc_car_label.configure(image=photo, bg=t["bg2"])

    def _refresh_car_combo(self):
        cars = [TR("no_car")]
        if self.current_user and self.current_user in self.data.get("users", {}):
            user = self.data["users"][self.current_user]
            cars += [c["name"] for c in user.get("cars", [])]
        self.calc_car_combo["values"] = cars
        self.calc_car_var.set(cars[0])

    def _on_car_selected(self, event=None):
        sel = self.calc_car_var.get()
        if sel == TR("no_car") or not self.current_user:
            self._set_calc_car_image(None)
            self._draw_fuel_chart()
            return
        user = self.data["users"].get(self.current_user, {})
        for car in user.get("cars", []):
            if car["name"] == sel:
                self._set_calc_car_image(car.get("photo"))
                avg = car.get("avg_consumption")
                if avg and hasattr(self, 'avg_var'):
                    self.avg_var.set(str(avg))
                self._draw_fuel_chart()
                return
        self._set_calc_car_image(None)
        self._draw_fuel_chart()

    def _calculate(self):
        sym = get_currency_symbol()
        mode = self._calc_mode.get()
        try:
            dist  = float(self.dist_var.get().replace(",", "."))
            price = float(self.price_var.get().replace(",", "."))
            if mode == "consumption":
                fuel = float(self.fuel_var.get().replace(",", "."))
            else:
                avg_input = float(self.avg_var.get().replace(",", "."))
        except ValueError:
            messagebox.showerror(TR("error"), TR("enter_numbers"))
            return

        if dist <= 0:
            messagebox.showerror(TR("error"), TR("distance_positive"))
            return

        if mode == "consumption":
            consumption = (fuel / dist) * 100
            cost = fuel * price
        else:
            if avg_input <= 0:
                messagebox.showerror(TR("error"), TR("distance_positive2"))
                return
            consumption = avg_input
            fuel = (avg_input / 100) * dist
            cost = fuel * price

        self.result_consumption_label.configure(text=f"{consumption:.2f} л/100 км")
        self.result_cost_label.configure(text=f"{cost:,.2f} {sym}")

        self.stat_labels["avg"].configure(text=f"{consumption:.2f} л/100 км")
        self.stat_labels["dist"].configure(text=f"{dist:.0f} км")
        self.stat_labels["fuel"].configure(text=f"{fuel:.1f} л")
        self.stat_labels["cost"].configure(text=f"{cost:,.2f} {sym}")

        if self.current_user:
            sel_car = self.calc_car_var.get()
            entry = {
                "date":        datetime.now().strftime("%d.%m.%Y %H:%M"),
                "car":         sel_car,
                "distance":    dist,
                "fuel":        round(fuel, 2),
                "price":       price,
                "currency":    sym,
                "consumption": round(consumption, 2),
                "cost":        round(cost, 2),
            }
            user = self.data["users"][self.current_user]
            user.setdefault("history", []).insert(0, entry)

            if sel_car != TR("no_car"):
                for car in user.get("cars", []):
                    if car["name"] == sel_car:
                        car["avg_consumption"] = round(consumption, 2)
                        break

            save_data(self.data)
            self._draw_fuel_chart()

    def _clear_calc(self):
        self.dist_var.set("")
        self.fuel_var.set("")
        self.price_var.set("")
        self.avg_var.set("")
        self.result_consumption_label.configure(text="— л/100 км")
        self.result_cost_label.configure(text="—")
        for k, v in {"avg": "— л/100 км", "dist": "— км", "fuel": "— л", "cost": "—"}.items():
            self.stat_labels[k].configure(text=v)

    # ── Smart Fuel Chart ────────────────────────────────────────────────────────
    def _on_chart_hover(self, event):
        if not self._chart_data_points:
            return
        c = self.chart_canvas
        best = None
        best_dist = 999999
        for (cx, cy, dt, val) in self._chart_data_points:
            d = abs(event.x - cx)
            if d < best_dist:
                best_dist = d
                best = (cx, cy, dt, val)
        if best and best_dist < 40:
            cx, cy, dt, val = best
            c.delete("tooltip")
            txt = f"{dt.strftime('%d.%m.%Y')}  {val:.1f} л/100"
            pad = 6
            tw = len(txt) * 7 + pad * 2
            th = 22
            tx = min(cx - tw // 2, c.winfo_width() - tw - 4)
            tx = max(tx, 4)
            ty = cy - th - 8
            if ty < 4:
                ty = cy + 12
            c.create_rectangle(tx, ty, tx + tw, ty + th,
                                fill=T()["bg3"], outline=T()["accent"], width=1, tags="tooltip")
            c.create_text(tx + tw // 2, ty + th // 2, text=txt,
                          fill=T()["fg"], font=("Courier", 9), tags="tooltip")
            r = 6
            c.delete("tooltip_dot")
            c.create_oval(cx-r, cy-r, cx+r, cy+r,
                          fill=T()["accent"], outline=T()["bg2"], width=2, tags="tooltip_dot")

    def _on_chart_leave(self, event):
        if hasattr(self, 'chart_canvas'):
            self.chart_canvas.delete("tooltip")
            self.chart_canvas.delete("tooltip_dot")

    def _draw_fuel_chart(self):
        if not hasattr(self, 'chart_canvas'):
            return
        c = self.chart_canvas
        t = T()
        c.delete("all")
        self._chart_data_points = []
        W = c.winfo_width()
        H = c.winfo_height()
        if W < 10 or H < 10:
            return

        pad_l, pad_r, pad_t, pad_b = 56, 24, 24, 40

        if not self.current_user:
            c.create_text(W//2, H//2, text=TR("login_for_chart"),
                          fill=t["fg2"], font=("Courier", 12))
            return

        selected_car = self.calc_car_var.get() if hasattr(self, 'calc_car_var') else TR("no_car")
        try:
            days_limit = int(self.chart_days_var.get())
        except Exception:
            days_limit = 30

        history = self.data["users"][self.current_user].get("history", [])
        cutoff  = datetime.now() - timedelta(days=days_limit)

        filtered = []
        for entry in history:
            try:
                dt = datetime.strptime(entry["date"], "%d.%m.%Y %H:%M")
            except Exception:
                continue
            if dt < cutoff:
                continue
            car_match = (selected_car == TR("no_car") or entry.get("car") == selected_car)
            if car_match:
                filtered.append((dt, entry["consumption"]))

        if not filtered:
            c.create_text(W//2, H//2, text=TR("no_history_chart"),
                          fill=t["fg2"], font=("Courier", 12))
            return

        filtered.sort(key=lambda x: x[0])
        values = [v for _, v in filtered]
        min_v  = max(0, min(values) - 1)
        max_v  = max(values) + 1
        rng    = max_v - min_v if max_v != min_v else 1

        chart_w = W - pad_l - pad_r
        chart_h = H - pad_t - pad_b

        def px(i):
            n = len(filtered)
            return pad_l + (i / max(n - 1, 1)) * chart_w

        def py(v):
            return pad_t + (1 - (v - min_v) / rng) * chart_h

        # Grid lines
        for gi in range(6):
            gv = min_v + gi * rng / 5
            gy = py(gv)
            c.create_line(pad_l, gy, W - pad_r, gy,
                          fill=t["chart_grid"], dash=(4, 6), width=1)
            c.create_text(pad_l - 6, gy, text=f"{gv:.1f}",
                          anchor="e", fill=t["fg2"], font=("Courier", 9))

        # Color zones
        y_green  = py(8)
        y_yellow = py(12)
        y_bottom = H - pad_b
        if y_green > pad_t:
            c.create_rectangle(pad_l, pad_t, W - pad_r, min(y_green, y_bottom),
                                fill="#1a2f1a", outline="")
        if y_yellow > y_green:
            c.create_rectangle(pad_l, max(y_green, pad_t), W - pad_r, min(y_yellow, y_bottom),
                                fill="#2a2410", outline="")
        c.create_rectangle(pad_l, max(y_yellow, pad_t), W - pad_r, y_bottom,
                            fill="#2a1212", outline="")

        # Fill under curve
        pts = []
        for i, (_, v) in enumerate(filtered):
            pts += [px(i), py(v)]
        if len(pts) >= 4:
            poly_pts = [pad_l, H - pad_b] + pts + [W - pad_r, H - pad_b]
            c.create_polygon(poly_pts, fill=t["chart_fill"], outline="")

        # Line segments with color coding
        for i in range(len(filtered) - 1):
            v1 = filtered[i][1]
            v2 = filtered[i+1][1]
            avg_v = (v1 + v2) / 2
            if avg_v <= 8:
                lcolor = t["green"]
            elif avg_v <= 12:
                lcolor = t["yellow"]
            else:
                lcolor = t["red"]
            x1, y1 = px(i),   py(v1)
            x2, y2 = px(i+1), py(v2)
            c.create_line(x1, y1, x2, y2, fill=lcolor, width=2.5, smooth=True)

        # Dots
        for i, (dt, v) in enumerate(filtered):
            x, y = px(i), py(v)
            r = 5
            dot_color = t["green"] if v <= 8 else (t["yellow"] if v <= 12 else t["red"])
            c.create_oval(x-r, y-r, x+r, y+r, fill=dot_color, outline=t["bg2"], width=2)
            self._chart_data_points.append((x, y, dt, v))

        # Axes
        c.create_line(pad_l, H - pad_b, W - pad_r, H - pad_b, fill=t["border"], width=1)
        c.create_line(pad_l, pad_t, pad_l, H - pad_b, fill=t["border"], width=1)

        # X-axis labels
        n = len(filtered)
        step = max(1, n // 8)
        for i in range(0, n, step):
            dt, _ = filtered[i]
            x = px(i)
            c.create_line(x, H - pad_b, x, H - pad_b + 5, fill=t["fg2"], width=1)
            c.create_text(x, H - pad_b + 14, text=dt.strftime("%d.%m"),
                          fill=t["fg2"], font=("Courier", 8))

        # Y-axis label
        c.create_text(pad_l - 40, pad_t + chart_h // 2, text="л/100",
                      fill=t["fg2"], font=("Courier", 8), angle=90)

        # Legend
        legend_x = W - pad_r - 130
        legend_items = [("≤8 эконом", t["green"]), ("8-12 норма", t["yellow"]), (">12 высокий", t["red"])]
        for li, (ltxt, lc) in enumerate(legend_items):
            lx = legend_x + li * 44
            c.create_rectangle(lx, pad_t + 2, lx + 10, pad_t + 12, fill=lc, outline="")
            c.create_text(lx + 13, pad_t + 7, text=ltxt, fill=t["fg2"], font=("Courier", 7), anchor="w")

        # Stats overlay
        if values:
            avg_all = sum(values) / len(values)
            c.create_text(W - pad_r - 4, pad_t + 6,
                          text=f"avg: {avg_all:.1f} л/100",
                          anchor="ne", fill=t["fg2"], font=("Courier", 9))

    # ── Profile ─────────────────────────────────────────────────────────────────
    def _build_profile(self):
        t = T()
        f = self.sections["profile"]
        for w in f.winfo_children(): w.destroy()

        canvas = tk.Canvas(f, bg=t["bg"], highlightthickness=0)
        scroll = ttk.Scrollbar(f, orient="vertical", command=canvas.yview)
        canvas.configure(yscrollcommand=scroll.set)
        scroll.pack(side="right", fill="y")
        canvas.pack(side="left", fill="both", expand=True)

        inner = tk.Frame(canvas, bg=t["bg"])
        win = canvas.create_window((0, 0), window=inner, anchor="nw")
        inner.bind("<Configure>", lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
        canvas.bind("<Configure>", lambda e: canvas.itemconfig(win, width=e.width))
        bind_mousewheel(inner, canvas)

        pad = tk.Frame(inner, bg=t["bg"])
        pad.pack(fill="both", expand=True, padx=32, pady=24)

        if not self.current_user:
            self._build_login_form(pad)
        else:
            self._build_logged_in_profile(pad)

    def _build_login_form(self, parent):
        t = T()
        self._auth_mode = tk.StringVar(value="login")

        tab_frame = tk.Frame(parent, bg=t["bg"])
        tab_frame.pack(pady=(0, 16))

        def switch(mode):
            self._auth_mode.set(mode)
            build()

        tk.Button(tab_frame, text=TR("login"), font=("Georgia", 12, "bold"),
                  bg=t["btn"], fg="white", bd=0, padx=20, pady=6,
                  cursor="hand2", command=lambda: switch("login")).pack(side="left", padx=4)
        tk.Button(tab_frame, text=TR("register"), font=("Georgia", 12),
                  bg=t["btn2"], fg=t["fg"], bd=0, padx=20, pady=6,
                  cursor="hand2", command=lambda: switch("register")).pack(side="left", padx=4)

        self._auth_form_frame = tk.Frame(parent, bg=t["bg2"], padx=32, pady=24)
        self._auth_form_frame.pack(fill="x", ipadx=8, ipady=8)

        def build():
            for w in self._auth_form_frame.winfo_children(): w.destroy()
            mode = self._auth_mode.get()
            tk.Label(self._auth_form_frame,
                     text=TR("create_account") if mode == "register" else TR("login_account"),
                     font=("Georgia", 15, "bold"), bg=t["bg2"], fg=t["fg"]).pack(pady=(0, 16))

            tk.Label(self._auth_form_frame, text=TR("email"),
                     font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w")
            email_entry = tk.Entry(self._auth_form_frame, font=("Courier", 12),
                                   bg=t["input_bg"], fg=t["fg"], insertbackground=t["fg"],
                                   bd=1, relief="solid", width=30)
            email_entry.pack(fill="x", pady=4, ipady=6)

            tk.Label(self._auth_form_frame, text=TR("password"),
                     font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w")
            pw_row = tk.Frame(self._auth_form_frame, bg=t["input_bg"], bd=1, relief="solid")
            pw_row.pack(fill="x", pady=4)
            pw_entry = tk.Entry(pw_row, font=("Courier", 12), show="•",
                                bg=t["input_bg"], fg=t["fg"], insertbackground=t["fg"],
                                bd=0, relief="flat")
            pw_entry.pack(side="left", fill="x", expand=True, ipady=6, padx=4)
            pw_visible = [False]
            eye_btn1 = tk.Button(pw_row, text="👁", font=("Courier", 11), bg=t["input_bg"],
                                  fg=t["fg2"], bd=0, cursor="hand2", padx=4)
            eye_btn1.pack(side="right", padx=2)
            def toggle_pw1():
                pw_visible[0] = not pw_visible[0]
                pw_entry.configure(show="" if pw_visible[0] else "•")
                eye_btn1.configure(text="🙈" if pw_visible[0] else "👁")
            eye_btn1.configure(command=toggle_pw1)

            pw2_entry = None
            if mode == "register":
                tk.Label(self._auth_form_frame, text=TR("repeat_password"),
                         font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w")
                pw2_row = tk.Frame(self._auth_form_frame, bg=t["input_bg"], bd=1, relief="solid")
                pw2_row.pack(fill="x", pady=4)
                pw2_entry = tk.Entry(pw2_row, font=("Courier", 12), show="•",
                                     bg=t["input_bg"], fg=t["fg"], insertbackground=t["fg"],
                                     bd=0, relief="flat")
                pw2_entry.pack(side="left", fill="x", expand=True, ipady=6, padx=4)
                pw2_visible = [False]
                eye_btn2 = tk.Button(pw2_row, text="👁", font=("Courier", 11), bg=t["input_bg"],
                                      fg=t["fg2"], bd=0, cursor="hand2", padx=4)
                eye_btn2.pack(side="right", padx=2)
                def toggle_pw2():
                    pw2_visible[0] = not pw2_visible[0]
                    pw2_entry.configure(show="" if pw2_visible[0] else "•")
                    eye_btn2.configure(text="🙈" if pw2_visible[0] else "👁")
                eye_btn2.configure(command=toggle_pw2)

            status = tk.Label(self._auth_form_frame, text="", font=("Courier", 10),
                              bg=t["bg2"], fg=t["red"])
            status.pack(pady=4)

            def submit():
                email = email_entry.get().strip().lower()
                pw    = pw_entry.get()
                if not email or not pw:
                    status.configure(text=TR("fill_all"))
                    return
                if not is_valid_email(email):
                    status.configure(text=TR("invalid_email"))
                    return
                if len(pw) < 6:
                    status.configure(text=TR("password_short"))
                    return
                if mode == "register":
                    pw2 = pw2_entry.get() if pw2_entry else ""
                    if pw != pw2:
                        status.configure(text=TR("passwords_mismatch"))
                        return
                    if email in self.data.get("users", {}):
                        status.configure(text=TR("user_exists"))
                        return
                    self.data.setdefault("users", {})[email] = {
                        "password": hash_pw(pw), "cars": [], "history": []
                    }
                    save_data(self.data)
                    self.current_user = email
                    self.data["current_user"] = email
                    save_data(self.data)
                else:
                    users = self.data.get("users", {})
                    if email not in users or users[email]["password"] != hash_pw(pw):
                        status.configure(text=TR("wrong_credentials"))
                        return
                    self.current_user = email
                    self.data["current_user"] = email
                    save_data(self.data)
                self._refresh_car_combo()
                self._refresh_profile()

            tk.Button(self._auth_form_frame,
                      text=TR("login") if mode == "login" else TR("register"),
                      font=("Georgia", 12, "bold"), bg=t["btn"], fg="white", bd=0,
                      pady=8, cursor="hand2", command=submit).pack(fill="x", pady=8)

        build()

    def _build_logged_in_profile(self, parent):
        t = T()
        hrow = tk.Frame(parent, bg=t["bg"])
        hrow.pack(fill="x", pady=(0, 16))
        tk.Label(hrow, text=f"👤  {self.current_user}", font=("Georgia", 14, "bold"),
                 bg=t["bg"], fg=t["fg"]).pack(side="left")

        btn_frame = tk.Frame(hrow, bg=t["bg"])
        btn_frame.pack(side="right")
        tk.Button(btn_frame, text=TR("logout"), font=("Courier", 10),
                  bg=t["btn2"], fg=t["fg"], bd=0, padx=12, pady=4,
                  cursor="hand2", command=self._logout).pack(side="left", padx=(0, 8))
        tk.Button(btn_frame, text=TR("delete_account"), font=("Courier", 10),
                  bg=t["red"], fg="white", bd=0, padx=12, pady=4,
                  cursor="hand2", command=self._delete_account).pack(side="left")

        pw_frame = tk.Frame(parent, bg=t["bg2"])
        pw_frame.pack(fill="x", pady=8, ipady=4)
        tk.Label(pw_frame, text=TR("change_password"), font=("Georgia", 13, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=16, pady=(12, 4))

        tk.Label(pw_frame, text=TR("new_password"),
                 font=("Courier", 9), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", padx=16)
        pw1_row = tk.Frame(pw_frame, bg=t["input_bg"], bd=1, relief="solid")
        pw1_row.pack(fill="x", padx=16, pady=(2, 4))
        new_pw = tk.Entry(pw1_row, font=("Courier", 11), show="•",
                          bg=t["input_bg"], fg=t["fg"], insertbackground=t["fg"], bd=0, relief="flat")
        new_pw.pack(side="left", fill="x", expand=True, ipady=5, padx=4)
        pw1_vis = [False]
        def toggle_pw1():
            pw1_vis[0] = not pw1_vis[0]
            new_pw.configure(show="" if pw1_vis[0] else "•")
            eye1.configure(text="🙈" if pw1_vis[0] else "👁")
        eye1 = tk.Button(pw1_row, text="👁", font=("Courier", 11), bg=t["input_bg"],
                          fg=t["fg2"], bd=0, cursor="hand2", padx=4, command=toggle_pw1)
        eye1.pack(side="right", padx=2)

        tk.Label(pw_frame, text=TR("repeat_new_password"),
                 font=("Courier", 9), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", padx=16)
        pw2_row = tk.Frame(pw_frame, bg=t["input_bg"], bd=1, relief="solid")
        pw2_row.pack(fill="x", padx=16, pady=(2, 4))
        new_pw2 = tk.Entry(pw2_row, font=("Courier", 11), show="•",
                           bg=t["input_bg"], fg=t["fg"], insertbackground=t["fg"], bd=0, relief="flat")
        new_pw2.pack(side="left", fill="x", expand=True, ipady=5, padx=4)
        pw2_vis = [False]
        def toggle_pw2():
            pw2_vis[0] = not pw2_vis[0]
            new_pw2.configure(show="" if pw2_vis[0] else "•")
            eye2.configure(text="🙈" if pw2_vis[0] else "👁")
        eye2 = tk.Button(pw2_row, text="👁", font=("Courier", 11), bg=t["input_bg"],
                          fg=t["fg2"], bd=0, cursor="hand2", padx=4, command=toggle_pw2)
        eye2.pack(side="right", padx=2)

        pw_status = tk.Label(pw_frame, text="", font=("Courier", 9), bg=t["bg2"], fg=t["green"])
        pw_status.pack(anchor="w", padx=16)

        def change_pw():
            np1 = new_pw.get()
            np2 = new_pw2.get()
            if len(np1) < 6:
                pw_status.configure(text=TR("password_short"), fg=T()["red"])
                return
            if np1 != np2:
                pw_status.configure(text=TR("passwords_mismatch"), fg=T()["red"])
                return
            self.data["users"][self.current_user]["password"] = hash_pw(np1)
            save_data(self.data)
            pw_status.configure(text=TR("password_changed"), fg=T()["green"])
            new_pw.delete(0, "end")
            new_pw2.delete(0, "end")

        tk.Button(pw_frame, text=TR("change_password_btn"), font=("Courier", 10),
                  bg=t["btn"], fg="white", bd=0, padx=10, pady=5,
                  cursor="hand2", command=change_pw).pack(anchor="w", padx=16, pady=(0, 12))

        tk.Frame(parent, bg=t["border"], height=1).pack(fill="x", pady=16)
        cars_header = tk.Frame(parent, bg=t["bg"])
        cars_header.pack(fill="x")
        tk.Label(cars_header, text=TR("my_cars"), font=("Georgia", 14, "bold"),
                 bg=t["bg"], fg=t["fg"]).pack(side="left")
        tk.Button(cars_header, text=TR("add_car"), font=("Courier", 10),
                  bg=t["btn"], fg="white", bd=0, padx=12, pady=4,
                  cursor="hand2", command=self._add_car_inline).pack(side="right")

        self.cars_container = tk.Frame(parent, bg=t["bg"])
        self.cars_container.pack(fill="x", pady=8)

        self._add_car_form_frame = tk.Frame(parent, bg=t["bg2"])
        self._add_car_form_visible = False

        self._render_cars()

    def _add_car_inline(self):
        t = T()
        if self._add_car_form_visible:
            self._add_car_form_frame.pack_forget()
            self._add_car_form_visible = False
            return

        for w in self._add_car_form_frame.winfo_children():
            w.destroy()

        self._add_car_form_frame.pack(fill="x", padx=0, pady=8, after=self.cars_container)
        self._add_car_form_visible = True

        tk.Label(self._add_car_form_frame, text=TR("add_car_title"),
                 font=("Georgia", 13, "bold"), bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=16, pady=(12, 4))

        tk.Label(self._add_car_form_frame, text=TR("car_name"),
                 font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", padx=16)
        name_var = tk.StringVar()
        name_entry = tk.Entry(self._add_car_form_frame, textvariable=name_var,
                              font=("Courier", 12), bg=t["input_bg"], fg=t["fg"],
                              insertbackground=t["fg"], bd=1, relief="solid")
        name_entry.pack(fill="x", padx=16, pady=(2, 8), ipady=6)

        photo_path_var = tk.StringVar(value="")
        photo_preview_frame = tk.Frame(self._add_car_form_frame, bg=t["bg2"])
        photo_preview_frame.pack(fill="x", padx=16, pady=4)

        preview_box = tk.Frame(photo_preview_frame, bg=t["bg3"], width=120, height=75)
        preview_box.pack_propagate(False)
        preview_box.pack(side="left", padx=(0, 12))
        preview_lbl = tk.Label(preview_box, bg=t["bg3"], text="📷", font=("Courier", 20), fg=t["fg2"])
        preview_lbl.place(relx=0.5, rely=0.5, anchor="center")

        right_col = tk.Frame(photo_preview_frame, bg=t["bg2"])
        right_col.pack(side="left", fill="x", expand=True)

        photo_name_lbl = tk.Label(right_col, text=TR("photo_not_chosen"),
                                   font=("Courier", 9), bg=t["bg2"], fg=t["fg2"])
        photo_name_lbl.pack(anchor="w", pady=(0, 6))

        def choose_photo():
            path = filedialog.askopenfilename(
                parent=self,
                filetypes=[("Images", "*.jpg *.jpeg *.png *.bmp *.webp")])
            if path:
                ext = os.path.splitext(path)[1]
                dest = os.path.join(PHOTO_DIR, f"car_{datetime.now().strftime('%Y%m%d%H%M%S')}{ext}")
                shutil.copy2(path, dest)
                photo_path_var.set(dest)
                photo_name_lbl.configure(text=os.path.basename(path))
                try:
                    prev_img = load_car_image(dest, 120, 75)
                    prev_photo = ImageTk.PhotoImage(prev_img)
                    self._photo_refs["add_car_preview"] = prev_photo
                    preview_lbl.configure(image=prev_photo, text="")
                except Exception:
                    pass

        tk.Button(right_col, text=TR("choose_photo"), font=("Courier", 10),
                  bg=t["btn2"], fg=t["fg"], bd=0, padx=12, pady=6,
                  cursor="hand2", command=choose_photo).pack(anchor="w")

        status = tk.Label(self._add_car_form_frame, text="",
                          font=("Courier", 9), bg=t["bg2"], fg=t["red"])
        status.pack(anchor="w", padx=16)

        btn_row = tk.Frame(self._add_car_form_frame, bg=t["bg2"])
        btn_row.pack(fill="x", padx=16, pady=12)

        def save_car():
            name = name_var.get().strip()
            if not name:
                status.configure(text=TR("enter_name"))
                return
            car = {"name": name, "photo": photo_path_var.get() or None}
            self.data["users"][self.current_user].setdefault("cars", []).append(car)
            save_data(self.data)
            self._add_car_form_frame.pack_forget()
            self._add_car_form_visible = False
            self._render_cars()
            self._refresh_car_combo()

        cancel_text = "✕  Отмена" if current_language == "ru" else "✕  Cancel"
        tk.Button(btn_row, text=TR("save"), font=("Georgia", 11, "bold"),
                  bg=t["btn"], fg="white", bd=0, pady=8,
                  cursor="hand2", command=save_car).pack(side="left", fill="x", expand=True, padx=(0, 8))
        tk.Button(btn_row, text=cancel_text,
                  font=("Courier", 10), bg=t["btn2"], fg=t["fg"], bd=0, pady=8,
                  cursor="hand2", command=lambda: [
                      self._add_car_form_frame.pack_forget(),
                      setattr(self, '_add_car_form_visible', False)
                  ]).pack(side="left")

    def _render_cars(self):
        t = T()
        for w in self.cars_container.winfo_children(): w.destroy()
        if not self.current_user:
            return
        cars = self.data["users"][self.current_user].get("cars", [])
        if not cars:
            tk.Label(self.cars_container, text=TR("no_cars"),
                     font=("Courier", 11), bg=t["bg"], fg=t["fg2"]).pack(pady=16)
            return
        row = tk.Frame(self.cars_container, bg=t["bg"])
        row.pack(fill="x")
        for i, car in enumerate(cars):
            card = tk.Frame(row, bg=t["bg2"], bd=0, relief="flat", padx=8, pady=8)
            card.pack(side="left", padx=8, pady=4)

            img_box = tk.Frame(card, bg=t["bg2"], width=140, height=90)
            img_box.pack_propagate(False)
            img_box.pack()
            ph = car.get("photo")
            img = load_car_image(ph, 140, 90) if ph and os.path.exists(ph) else make_placeholder(140, 90)
            photo = ImageTk.PhotoImage(img)
            self._photo_refs[f"car_{i}"] = photo
            lbl = tk.Label(img_box, image=photo, bg=t["bg2"])
            lbl.place(relx=0.5, rely=0.5, anchor="center")

            tk.Label(card, text=car["name"], font=("Courier", 10, "bold"),
                     bg=t["bg2"], fg=t["fg"]).pack(pady=2)

            avg = car.get("avg_consumption")
            if avg:
                tk.Label(card, text=f"💧 {avg} л/100", font=("Courier", 8),
                         bg=t["bg2"], fg=t["fg2"]).pack()

            tk.Button(card, text=TR("delete"), font=("Courier", 9),
                      bg=t["red"], fg="white", bd=0, padx=8, pady=3,
                      cursor="hand2", command=lambda idx=i: self._delete_car(idx)).pack(pady=(4,0))

    def _delete_car(self, idx):
        if messagebox.askyesno(TR("delete_confirm"), TR("delete_car_confirm")):
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
        if messagebox.askyesno(TR("delete_account_confirm"), TR("delete_account_msg")):
            for car in self.data["users"].get(self.current_user, {}).get("cars", []):
                if car.get("photo") and os.path.exists(car["photo"]):
                    try: os.remove(car["photo"])
                    except: pass
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

    # ── History ─────────────────────────────────────────────────────────────────
    def _build_history(self):
        t = T()
        f = self.sections["history"]
        for w in f.winfo_children(): w.destroy()

        tk.Label(f, text=TR("history_title"), font=("Georgia", 16, "bold"),
                 bg=t["bg"], fg=t["fg"]).pack(anchor="w", padx=24, pady=(20, 4))

        if not self.current_user:
            tk.Label(f, text=TR("login_for_history"),
                     font=("Courier", 12), bg=t["bg"], fg=t["fg2"]).pack(pady=40)
            return

        hdr = tk.Frame(f, bg=t["bg"])
        hdr.pack(fill="x", padx=24)
        tk.Button(hdr, text=TR("clear_all"), font=("Courier", 9),
                  bg=t["red"], fg="white", bd=0, padx=8, pady=4,
                  cursor="hand2", command=self._clear_all_history).pack(side="right")

        self.history_frame = tk.Frame(f, bg=t["bg"])
        self.history_frame.pack(fill="both", expand=True, padx=24, pady=8)

        canvas = tk.Canvas(self.history_frame, bg=t["bg"], highlightthickness=0)
        scroll = ttk.Scrollbar(self.history_frame, orient="vertical", command=canvas.yview)
        canvas.configure(yscrollcommand=scroll.set)
        scroll.pack(side="right", fill="y")
        canvas.pack(side="left", fill="both", expand=True)

        self._hist_inner = tk.Frame(canvas, bg=t["bg"])
        win = canvas.create_window((0, 0), window=self._hist_inner, anchor="nw")
        self._hist_inner.bind("<Configure>",
                               lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
        canvas.bind("<Configure>", lambda e: canvas.itemconfig(win, width=e.width))
        bind_mousewheel(self._hist_inner, canvas)

        self._render_history_entries()

    def _render_history_entries(self):
        t = T()
        for w in self._hist_inner.winfo_children(): w.destroy()
        if not self.current_user:
            return
        history = self.data["users"][self.current_user].get("history", [])
        if not history:
            tk.Label(self._hist_inner, text=TR("history_empty"), font=("Courier", 12),
                     bg=t["bg"], fg=t["fg2"]).pack(pady=40)
            return

        for i, entry in enumerate(history):
            card = tk.Frame(self._hist_inner, bg=t["bg2"], pady=8)
            card.pack(fill="x", pady=6, ipadx=8)

            ph = None
            car_name = entry.get("car", "—")
            if self.current_user and car_name not in (TR("no_car"), "— Без авто —", "— No car —"):
                for car in self.data["users"][self.current_user].get("cars", []):
                    if car["name"] == car_name and car.get("photo"):
                        ph = car["photo"]
                        break

            img_box = tk.Frame(card, bg=t["bg2"], width=200, height=120)
            img_box.pack_propagate(False)
            img_box.pack(side="left", padx=12)
            img = load_car_image(ph, 200, 120) if ph and os.path.exists(ph) else make_placeholder(200, 120)
            photo = ImageTk.PhotoImage(img)
            self._photo_refs[f"hist_car_{i}"] = photo
            car_lbl = tk.Label(img_box, image=photo, bg=t["bg2"])
            car_lbl.place(relx=0.5, rely=0.5, anchor="center")

            info = tk.Frame(card, bg=t["bg2"])
            info.pack(side="left", fill="x", expand=True, padx=8)

            sym = entry.get("currency", "₽")
            tk.Label(info, text=f"📅 {entry['date']}",
                     font=("Courier", 10, "bold"), bg=t["bg2"], fg=t["fg"]).pack(anchor="w")
            tk.Label(info, text=f"🚗 {car_name}",
                     font=("Courier", 10), bg=t["bg2"], fg=t["accent"]).pack(anchor="w")
            tk.Label(info,
                     text=f"📏 {entry['distance']} км   ⛽ {entry['fuel']} л   💰 {entry['price']} {sym}/л",
                     font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", pady=2)
            tk.Label(info,
                     text=f"💧 {entry['consumption']} л/100км   💳 {entry['cost']:,.2f} {sym}",
                     font=("Courier", 11, "bold"), bg=t["bg2"], fg=t["green"]).pack(anchor="w")

            # Symmetrical button column — fixed width, equal-sized buttons
            btn_col = tk.Frame(card, bg=t["bg2"], width=110)
            btn_col.pack_propagate(False)
            btn_col.pack(side="right", padx=12, pady=4)

            copy_text = (
                f"Date: {entry['date']}\n"
                f"Car: {car_name}\n"
                f"Distance: {entry['distance']} km\n"
                f"Fuel: {entry['fuel']} L\n"
                f"Price: {entry['price']} {sym}/L\n"
                f"Consumption: {entry['consumption']} L/100km\n"
                f"Cost: {entry['cost']:,.2f} {sym}"
            )

            def copy_entry(text=copy_text):
                self.clipboard_clear()
                self.clipboard_append(text)
                messagebox.showinfo(TR("copied"), TR("copied_msg"))

            tk.Button(btn_col, text=TR("copy"), font=("Courier", 9),
                      bg=t["green"], fg="white", bd=0,
                      cursor="hand2", command=copy_entry,
                      width=12, height=2).pack(fill="x", pady=(0, 4))
            tk.Button(btn_col, text=TR("delete_entry"), font=("Courier", 9),
                      bg=t["red"], fg="white", bd=0,
                      cursor="hand2", command=lambda idx=i: self._delete_history_entry(idx),
                      width=12, height=2).pack(fill="x")

    def _delete_history_entry(self, idx):
        del self.data["users"][self.current_user]["history"][idx]
        save_data(self.data)
        self._render_history_entries()

    def _clear_all_history(self):
        if messagebox.askyesno(TR("clear_history_confirm"), TR("clear_history_msg")):
            self.data["users"][self.current_user]["history"] = []
            save_data(self.data)
            self._render_history_entries()

    def _refresh_history(self):
        f = self.sections["history"]
        for w in f.winfo_children(): w.destroy()
        self._build_history()

    # ── Gas Stations (with embedded map) ────────────────────────────────────────
    def _build_gas_stations(self):
        t = T()
        f = self.sections["gas_stations"]
        for w in f.winfo_children(): w.destroy()

        # Split: left panel (controls) + right panel (map)
        main_pane = tk.Frame(f, bg=t["bg"])
        main_pane.pack(fill="both", expand=True)

        # ── Left control panel ──
        left_scroll_canvas = tk.Canvas(main_pane, bg=t["bg"], highlightthickness=0, width=320)
        left_scroll_canvas.pack_propagate(False)
        left_scroll = ttk.Scrollbar(main_pane, orient="vertical", command=left_scroll_canvas.yview)
        left_scroll_canvas.configure(yscrollcommand=left_scroll.set)
        left_scroll.pack(side="left", fill="y")
        left_scroll_canvas.pack(side="left", fill="y")

        left_inner = tk.Frame(left_scroll_canvas, bg=t["bg"])
        left_win = left_scroll_canvas.create_window((0, 0), window=left_inner, anchor="nw")
        left_inner.bind("<Configure>", lambda e: left_scroll_canvas.configure(scrollregion=left_scroll_canvas.bbox("all")))
        left_scroll_canvas.bind("<Configure>", lambda e: left_scroll_canvas.itemconfig(left_win, width=e.width))
        bind_mousewheel(left_inner, left_scroll_canvas)

        pad = tk.Frame(left_inner, bg=t["bg"])
        pad.pack(fill="both", expand=True, padx=12, pady=16)

        tk.Label(pad, text=TR("gs_title"), font=("Georgia", 15, "bold"),
                 bg=t["bg"], fg=t["fg"]).pack(anchor="w", pady=(0, 12))

        # Search card
        search_card = tk.Frame(pad, bg=t["bg2"])
        search_card.pack(fill="x", pady=(0, 8))
        search_inner = tk.Frame(search_card, bg=t["bg2"])
        search_inner.pack(fill="x", padx=14, pady=12)

        tk.Label(search_inner, text=TR("gs_city"), font=("Courier", 9),
                 bg=t["bg2"], fg=t["fg2"]).pack(anchor="w")
        self.gs_city_var = tk.StringVar()
        city_row = tk.Frame(search_inner, bg=t["input_bg"], bd=1, relief="solid")
        city_row.pack(fill="x", pady=(2, 8))
        tk.Label(city_row, text="🏙", font=("Courier", 12), bg=t["input_bg"], fg=t["fg2"], padx=6).pack(side="left")
        tk.Entry(city_row, textvariable=self.gs_city_var, font=("Courier", 11),
                 bg=t["input_bg"], fg=t["fg"], insertbackground=t["fg"], bd=0, relief="flat").pack(
                 side="left", fill="x", expand=True, pady=5)

        # Car selector
        tk.Label(search_inner, text=TR("car") + ":", font=("Courier", 9),
                 bg=t["bg2"], fg=t["fg2"]).pack(anchor="w")
        self.gs_car_var = tk.StringVar(value=TR("no_car"))
        self.gs_car_combo = ttk.Combobox(search_inner, textvariable=self.gs_car_var,
                                          state="readonly", font=("Courier", 10))
        self.gs_car_combo.pack(fill="x", pady=(2, 8))
        self.gs_car_combo.bind("<<ComboboxSelected>>", self._on_gs_car_selected)
        self._refresh_gs_car_combo()

        self.gs_avg_lbl = tk.Label(search_inner, text="",
                                    font=("Courier", 9), bg=t["bg2"], fg=t["accent"])
        self.gs_avg_lbl.pack(anchor="w", pady=(0, 8))

        tk.Button(search_inner, text=TR("gs_search"), font=("Georgia", 11, "bold"),
                  bg=t["btn"], fg="white", bd=0, pady=8,
                  cursor="hand2", command=self._gs_search_on_map).pack(fill="x")

        # Separator
        tk.Frame(pad, bg=t["border"], height=1).pack(fill="x", pady=10)

        # Selected station info card
        station_card = tk.Frame(pad, bg=t["bg2"])
        station_card.pack(fill="x", pady=(0, 8))
        st_inner = tk.Frame(station_card, bg=t["bg2"])
        st_inner.pack(fill="x", padx=14, pady=12)

        tk.Label(st_inner, text=TR("gs_selected_station"), font=("Courier", 9, "bold"),
                 bg=t["bg2"], fg=t["fg2"]).pack(anchor="w")
        self.gs_station_name_lbl = tk.Label(st_inner, text=TR("gs_click_hint"),
                                             font=("Courier", 9), bg=t["bg2"], fg=t["fg2"],
                                             wraplength=260, justify="left")
        self.gs_station_name_lbl.pack(anchor="w", pady=(2, 8))

        # Distance
        tk.Label(st_inner, text=TR("gs_enter_distance"), font=("Courier", 9),
                 bg=t["bg2"], fg=t["fg2"]).pack(anchor="w")
        dist_row = tk.Frame(st_inner, bg=t["input_bg"], bd=1, relief="solid")
        dist_row.pack(fill="x", pady=(2, 8))
        tk.Label(dist_row, text="📏", font=("Courier", 11), bg=t["input_bg"], fg=t["fg2"], padx=6).pack(side="left")
        self.gs_dist_var = tk.StringVar()
        tk.Entry(dist_row, textvariable=self.gs_dist_var, font=("Courier", 11),
                 bg=t["input_bg"], fg=t["fg"], insertbackground=t["fg"], bd=0, relief="flat").pack(
                 side="left", fill="x", expand=True, pady=5)
        tk.Label(dist_row, text="км", font=("Courier", 9), bg=t["input_bg"], fg=t["fg2"], padx=6).pack(side="right")

        self.gs_dist_calc_lbl = tk.Label(st_inner, text="",
                                          font=("Courier", 8), bg=t["bg2"], fg=t["green"])
        self.gs_dist_calc_lbl.pack(anchor="w", pady=(0, 4))

        # Fuel price
        tk.Label(st_inner, text=TR("gs_fuel_price_hint"), font=("Courier", 9),
                 bg=t["bg2"], fg=t["fg2"]).pack(anchor="w")
        price_row = tk.Frame(st_inner, bg=t["input_bg"], bd=1, relief="solid")
        price_row.pack(fill="x", pady=(2, 8))
        tk.Label(price_row, text="💰", font=("Courier", 11), bg=t["input_bg"], fg=t["fg2"], padx=6).pack(side="left")
        self.gs_price_var = tk.StringVar()
        tk.Entry(price_row, textvariable=self.gs_price_var, font=("Courier", 11),
                 bg=t["input_bg"], fg=t["fg"], insertbackground=t["fg"], bd=0, relief="flat").pack(
                 side="left", fill="x", expand=True, pady=5)
        tk.Label(price_row, text=f"{get_currency_symbol()}/л", font=("Courier", 9),
                 bg=t["input_bg"], fg=t["fg2"], padx=6).pack(side="right")

        # Avg from car
        self.gs_avg_info_lbl = tk.Label(st_inner, text="",
                                         font=("Courier", 9, "bold"), bg=t["bg2"], fg=t["accent"])
        self.gs_avg_info_lbl.pack(anchor="w", pady=(0, 8))

        tk.Button(st_inner, text=TR("gs_to_calc"), font=("Georgia", 11, "bold"),
                  bg=t["btn"], fg="white", bd=0, pady=8,
                  cursor="hand2", command=self._gs_to_calculator).pack(fill="x")

        # ── Right: Map ──
        map_frame = tk.Frame(main_pane, bg=t["bg3"], bd=0)
        map_frame.pack(side="left", fill="both", expand=True)

        if MAP_AVAILABLE:
            self._gs_map = tkintermapview.TkinterMapView(map_frame, corner_radius=0)
            self._gs_map.pack(fill="both", expand=True)
            # Default position: Moscow
            self._gs_map.set_position(55.7558, 37.6176)
            self._gs_map.set_zoom(12)
            self._map_markers = []
            self._map_user_marker = None
            self._map_selected_station = None
        else:
            tk.Label(map_frame, text=TR("gs_no_map"),
                     font=("Courier", 11), bg=t["bg3"], fg=t["fg2"],
                     justify="center").place(relx=0.5, rely=0.5, anchor="center")
            self._gs_map = None

        self._update_gs_avg_display()

    def _gs_search_on_map(self):
        if not MAP_AVAILABLE or not hasattr(self, '_gs_map') or self._gs_map is None:
            return

        city = self.gs_city_var.get().strip()
        if not city:
            if hasattr(self, 'gs_station_name_lbl'):
                self.gs_station_name_lbl.configure(
                    text="⚠️ " + ("Введите город или адрес" if current_language == "ru" else "Enter city or address"),
                    fg=T()["yellow"])
            return

        # Geocode the city via map widget
        try:
            self._gs_map.set_address(city)
        except Exception:
            pass

        # Clear old markers
        for m in self._map_markers:
            try:
                m.delete()
            except Exception:
                pass
        self._map_markers = []

        if self._map_user_marker:
            try:
                self._map_user_marker.delete()
            except Exception:
                pass
            self._map_user_marker = None

        # Get current map center after address set
        self.after(800, lambda: self._place_station_markers(city))

        if hasattr(self, 'gs_station_name_lbl'):
            self.gs_station_name_lbl.configure(
                text=TR("gs_map_search_hint"), fg=T()["accent"])

    # Known gas station brands with approximate relative offsets for demo placement
    GS_BRANDS = [
        "Лукойл", "Газпромнефть", "Роснефть", "Shell", "BP",
        "Total", "Neste", "Башнефть", "ТНК", "Сургутнефтегаз",
    ]

    def _place_station_markers(self, city):
        if not MAP_AVAILABLE or not hasattr(self, '_gs_map') or self._gs_map is None:
            return

        # Get the map center position
        try:
            center_lat = self._gs_map.get_position()[0]
            center_lon = self._gs_map.get_position()[1]
        except Exception:
            center_lat, center_lon = 55.7558, 37.6176

        # Store user/origin position
        self._map_user_lat = center_lat
        self._map_user_lon = center_lon

        # Mark city center as "My position"
        try:
            label = "📍 " + ("Мой маршрут" if current_language == "ru" else "My route")
            self._map_user_marker = self._gs_map.set_marker(
                center_lat, center_lon,
                text=label,
                marker_color_circle=T()["accent"],
                marker_color_outside=T()["bg2"]
            )
        except Exception:
            pass

        # Place 8 stations at varying offsets around center
        import random
        random.seed(hash(city) % 999999)
        offsets = [
            (0.008,  0.012),
            (-0.006, 0.018),
            (0.015, -0.010),
            (-0.012, -0.008),
            (0.020,  0.005),
            (-0.018, 0.015),
            (0.005, -0.020),
            (0.012,  0.022),
        ]
        brands = self.GS_BRANDS[:len(offsets)]
        random.shuffle(brands)

        for idx, ((dlat, dlon), brand) in enumerate(zip(offsets, brands)):
            slat = center_lat + dlat + random.uniform(-0.003, 0.003)
            slon = center_lon + dlon + random.uniform(-0.003, 0.003)
            try:
                marker = self._gs_map.set_marker(
                    slat, slon,
                    text=f"⛽ {brand}",
                    marker_color_circle="#f85149",
                    marker_color_outside="#21262d",
                    command=lambda m, lat=slat, lon=slon, b=brand: self._on_map_station_click(m, lat, lon, b)
                )
                self._map_markers.append(marker)
            except Exception:
                pass

        if hasattr(self, 'gs_station_name_lbl'):
            hint = TR("gs_click_hint")
            self.gs_station_name_lbl.configure(text=hint, fg=T()["fg2"])

    def _on_map_station_click(self, marker, lat, lon, brand_name):
        t = T()
        self._map_selected_station = brand_name
        self._map_selected_lat = lat
        self._map_selected_lon = lon

        # Calculate distance from user position
        dist_km = None
        if self._map_user_lat is not None and self._map_user_lon is not None:
            dist_km = haversine_km(self._map_user_lat, self._map_user_lon, lat, lon)
            self.gs_dist_var.set(f"{dist_km:.1f}")
            if hasattr(self, 'gs_dist_calc_lbl'):
                self.gs_dist_calc_lbl.configure(
                    text=TR("gs_calc_distance") + f": {dist_km:.1f} км",
                    fg=t["green"])

        if hasattr(self, 'gs_station_name_lbl'):
            self.gs_station_name_lbl.configure(
                text=f"⛽ {brand_name}\n📍 {lat:.4f}, {lon:.4f}",
                fg=t["fg"])

        self._update_gs_avg_display()

    def _refresh_gs_car_combo(self):
        cars = [TR("no_car")]
        if self.current_user and self.current_user in self.data.get("users", {}):
            user = self.data["users"][self.current_user]
            cars += [c["name"] for c in user.get("cars", [])]
        if hasattr(self, 'gs_car_combo'):
            self.gs_car_combo["values"] = cars
            self.gs_car_var.set(cars[0])

    def _on_gs_car_selected(self, event=None):
        self._update_gs_avg_display()

    def _update_gs_avg_display(self):
        if not hasattr(self, 'gs_avg_info_lbl'):
            return
        t = T()
        sel = self.gs_car_var.get() if hasattr(self, 'gs_car_var') else TR("no_car")
        if sel == TR("no_car") or not self.current_user:
            self.gs_avg_info_lbl.configure(text="")
            if hasattr(self, 'gs_avg_lbl'):
                self.gs_avg_lbl.configure(text="")
            return
        user = self.data["users"].get(self.current_user, {})
        for car in user.get("cars", []):
            if car["name"] == sel:
                avg = car.get("avg_consumption")
                if avg:
                    txt = f"💧 {TR('gs_avg_from_car')}: {avg} л/100 км"
                    self.gs_avg_info_lbl.configure(text=txt, fg=t["accent"])
                    if hasattr(self, 'gs_avg_lbl'):
                        self.gs_avg_lbl.configure(text=txt)
                    return
        self.gs_avg_info_lbl.configure(text=TR("gs_no_avg"), fg=t["fg2"])
        if hasattr(self, 'gs_avg_lbl'):
            self.gs_avg_lbl.configure(text="")

    def _gs_to_calculator(self):
        dist_str  = self.gs_dist_var.get().strip()
        price_str = self.gs_price_var.get().strip()

        if not dist_str:
            if hasattr(self, 'gs_station_name_lbl'):
                msg = "⚠️ Введите расстояние до заправки" if current_language == "ru" else "⚠️ Enter distance to station"
                self.gs_station_name_lbl.configure(text=msg, fg=T()["yellow"])
            return

        self._show_section("calculator")

        if dist_str:
            self.dist_var.set(dist_str)
        if price_str:
            self.price_var.set(price_str)

        sel_car = self.gs_car_var.get()
        if sel_car != TR("no_car") and self.current_user:
            user = self.data["users"].get(self.current_user, {})
            for car in user.get("cars", []):
                if car["name"] == sel_car:
                    avg = car.get("avg_consumption")
                    if avg:
                        self._calc_mode.set("cost")
                        self._build_calc_form()
                        self.avg_var.set(str(avg))
                    self.calc_car_var.set(sel_car)
                    self._on_car_selected()
                    break

    def _refresh_gas_stations(self):
        self._refresh_gs_car_combo()
        self._update_gs_avg_display()

    # ── About ───────────────────────────────────────────────────────────────────
    def _build_about(self):
        t = T()
        f = self.sections["about"]
        for w in f.winfo_children(): w.destroy()

        canvas = tk.Canvas(f, bg=t["bg"], highlightthickness=0)
        scroll = ttk.Scrollbar(f, orient="vertical", command=canvas.yview)
        canvas.configure(yscrollcommand=scroll.set)
        scroll.pack(side="right", fill="y")
        canvas.pack(side="left", fill="both", expand=True)

        inner = tk.Frame(canvas, bg=t["bg"])
        win = canvas.create_window((0, 0), window=inner, anchor="nw")
        inner.bind("<Configure>", lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
        canvas.bind("<Configure>", lambda e: canvas.itemconfig(win, width=e.width))
        bind_mousewheel(inner, canvas)

        outer_pad = tk.Frame(inner, bg=t["bg"])
        outer_pad.pack(fill="both", expand=True, pady=32)

        pad = tk.Frame(outer_pad, bg=t["bg"])
        pad.pack(anchor="center", padx=48)

        hero = tk.Frame(pad, bg=t["bg2"], pady=32)
        hero.pack(fill="x", pady=(0, 24))
        tk.Label(hero, text="⛽", font=("Courier", 48), bg=t["bg2"], fg=t["accent"]).pack()
        tk.Label(hero, text=TR("app_title"),
                 font=("Georgia", 22, "bold"), bg=t["bg2"], fg=t["fg"]).pack()
        tk.Label(hero, text=TR("version"),
                 font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(pady=4)

        sections_info = [
            (TR("about_section1_title"), TR("about_section1")),
            (TR("about_section2_title"), TR("about_section2")),
            (TR("about_section3_title"), TR("about_section3")),
            (TR("about_section4_title"), TR("about_section4")),
            (TR("about_section5_title"), TR("about_section5")),
        ]

        for title, body in sections_info:
            sec = tk.Frame(pad, bg=t["bg2"])
            sec.pack(fill="x", pady=8, ipadx=16, ipady=12)
            tk.Label(sec, text=title, font=("Georgia", 13, "bold"),
                     bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=16, pady=(8, 2))
            tk.Label(sec, text=body, font=("Courier", 10), bg=t["bg2"],
                     fg=t["fg2"], justify="left", wraplength=700,
                     anchor="w").pack(anchor="w", padx=16, pady=(2, 8))

        tk.Label(pad, text=TR("footer"),
                 font=("Courier", 9), bg=t["bg"], fg=t["fg2"]).pack(pady=16)

    # ── Settings ────────────────────────────────────────────────────────────────
    def _build_settings(self):
        t = T()
        f = self.sections["settings"]
        for w in f.winfo_children(): w.destroy()

        canvas = tk.Canvas(f, bg=t["bg"], highlightthickness=0)
        scroll = ttk.Scrollbar(f, orient="vertical", command=canvas.yview)
        canvas.configure(yscrollcommand=scroll.set)
        scroll.pack(side="right", fill="y")
        canvas.pack(side="left", fill="both", expand=True)

        inner = tk.Frame(canvas, bg=t["bg"])
        win = canvas.create_window((0, 0), window=inner, anchor="nw")
        inner.bind("<Configure>", lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
        canvas.bind("<Configure>", lambda e: canvas.itemconfig(win, width=e.width))
        bind_mousewheel(inner, canvas)

        pad = tk.Frame(inner, bg=t["bg"])
        pad.pack(fill="both", expand=True, padx=48, pady=32)

        tk.Label(pad, text=TR("settings_title"), font=("Georgia", 18, "bold"),
                 bg=t["bg"], fg=t["fg"]).pack(anchor="w", pady=(0, 24))

        # ── Theme ──
        theme_card = tk.Frame(pad, bg=t["bg2"], pady=16)
        theme_card.pack(fill="x", pady=8)
        tk.Label(theme_card, text=TR("theme"), font=("Georgia", 13, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=20, pady=(0, 4))
        tk.Label(theme_card, text=TR("theme_desc"),
                 font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", padx=20)

        btn_row = tk.Frame(theme_card, bg=t["bg2"])
        btn_row.pack(padx=20, pady=12, anchor="w")

        def apply_theme(name):
            global current_theme
            current_theme = name
            self._save_settings()
            self._rebuild_ui()
            self._show_section("settings")

        tk.Button(btn_row, text=TR("dark_theme"),
                  font=("Georgia", 11, "bold"), bg="#1f2937", fg="#e5e7eb",
                  bd=0, padx=20, pady=8, cursor="hand2",
                  command=lambda: apply_theme("dark")).pack(side="left", padx=(0, 12))
        tk.Button(btn_row, text=TR("light_theme"),
                  font=("Georgia", 11, "bold"), bg="#e2e8f0", fg="#1a202c",
                  bd=0, padx=20, pady=8, cursor="hand2",
                  command=lambda: apply_theme("light")).pack(side="left")

        tk.Label(theme_card,
                 text=f"{TR('current_theme')} {TR('dark') if current_theme == 'dark' else TR('light')}",
                 font=("Courier", 10), bg=t["bg2"], fg=t["accent"]).pack(anchor="w", padx=20, pady=4)

        # ── Currency ──
        cur_card = tk.Frame(pad, bg=t["bg2"], pady=16)
        cur_card.pack(fill="x", pady=8)
        tk.Label(cur_card, text=TR("currency_setting"), font=("Georgia", 13, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=20, pady=(0, 4))
        tk.Label(cur_card, text=TR("currency_desc"),
                 font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", padx=20)

        cur_var = tk.StringVar(value=current_currency)
        cur_combo = ttk.Combobox(cur_card, textvariable=cur_var,
                                  values=list(CURRENCIES.keys()),
                                  state="readonly", width=14, font=("Courier", 11))
        cur_combo.pack(anchor="w", padx=20, pady=8)

        cur_status = tk.Label(cur_card, text=f"{TR('current')} {get_currency_symbol()}",
                               font=("Courier", 10), bg=t["bg2"], fg=t["accent"])
        cur_status.pack(anchor="w", padx=20)

        def apply_currency(event=None):
            global current_currency
            current_currency = cur_var.get()
            self._save_settings()
            cur_status.configure(text=f"{TR('current')} {get_currency_symbol()}")
            if hasattr(self, 'calc_form_frame'):
                self._build_calc_form()

        cur_combo.bind("<<ComboboxSelected>>", apply_currency)

        # ── Language ──
        lang_card = tk.Frame(pad, bg=t["bg2"], pady=16)
        lang_card.pack(fill="x", pady=8)
        tk.Label(lang_card, text=TR("language_setting"), font=("Georgia", 13, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=20, pady=(0, 4))
        tk.Label(lang_card, text=TR("language_desc"),
                 font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", padx=20)

        lang_row = tk.Frame(lang_card, bg=t["bg2"])
        lang_row.pack(padx=20, pady=12, anchor="w")

        def apply_language(lang):
            global current_language
            current_language = lang
            self._save_settings()
            self._rebuild_ui()
            self._show_section("settings")

        tk.Button(lang_row, text="🇷🇺  Русский",
                  font=("Georgia", 11, "bold"),
                  bg=t["btn"] if current_language == "ru" else t["btn2"],
                  fg="white" if current_language == "ru" else t["fg"],
                  bd=0, padx=20, pady=8, cursor="hand2",
                  command=lambda: apply_language("ru")).pack(side="left", padx=(0, 12))
        tk.Button(lang_row, text="🇬🇧  English",
                  font=("Georgia", 11, "bold"),
                  bg=t["btn"] if current_language == "en" else t["btn2"],
                  fg="white" if current_language == "en" else t["fg"],
                  bd=0, padx=20, pady=8, cursor="hand2",
                  command=lambda: apply_language("en")).pack(side="left")

        tk.Label(lang_card,
                 text=f"{TR('current')} {'Русский 🇷🇺' if current_language == 'ru' else 'English 🇬🇧'}",
                 font=("Courier", 10), bg=t["bg2"], fg=t["accent"]).pack(anchor="w", padx=20, pady=4)


# ─── Entry point ───────────────────────────────────────────────────────────────
if __name__ == "__main__":
    app = FuelApp()
    app.mainloop()
