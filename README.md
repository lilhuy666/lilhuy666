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
    from PIL import Image, ImageTk, ImageDraw, UnidentifiedImageError
except ImportError:
    import subprocess, sys
    subprocess.check_call([sys.executable, "-m", "pip", "install", "Pillow"])
    from PIL import Image, ImageTk, ImageDraw, UnidentifiedImageError

# ─── Data storage ──────────────────────────────────────────────────────────────
DATA_FILE = os.path.join(os.path.expanduser("~"), ".fuel_calc_data.json")
PHOTO_DIR = os.path.join(os.path.expanduser("~"), ".fuel_calc_photos")
os.makedirs(PHOTO_DIR, exist_ok=True)

def load_data():
    if os.path.exists(DATA_FILE):
        try:
            with open(DATA_FILE, "r", encoding="utf-8") as f:
                return json.load(f)
        except (json.JSONDecodeError, IOError):
            return {"users": {}, "current_user": None, "settings": {}}
    return {"users": {}, "current_user": None, "settings": {}}

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

# ─── Currencies ────────────────────────────────────────────────────────────────
CURRENCIES = {
    "₽ RUB": "₽",
    "$ USD": "$",
}

# ─── Map data ──────────────────────────────────────────────────────────────────
GAS_STATIONS_RUSSIA = [
    {"name": "Лукойл", "city": "Москва", "address": "Ленинградское шоссе, 1", "lat": 55.7558, "lon": 37.6173, "brand": "lukoil", "price": 52.50},
    {"name": "Газпромнефть", "city": "Москва", "address": "Проспект Мира, 100", "lat": 55.7512, "lon": 37.6184, "brand": "gazprom", "price": 53.20},
    {"name": "Роснефть", "city": "Москва", "address": "Варшавское шоссе, 25", "lat": 55.7600, "lon": 37.6200, "brand": "rosneft", "price": 51.80},
    {"name": "Лукойл", "city": "Санкт-Петербург", "address": "Невский проспект, 50", "lat": 59.9343, "lon": 30.3351, "brand": "lukoil", "price": 53.00},
    {"name": "Газпромнефть", "city": "Санкт-Петербург", "address": "Московский проспект, 150", "lat": 59.9290, "lon": 30.3200, "brand": "gazprom", "price": 54.00},
    {"name": "Роснефть", "city": "Казань", "address": "Проспект Победы, 100", "lat": 55.7961, "lon": 49.1064, "brand": "rosneft", "price": 50.50},
    {"name": "Лукойл", "city": "Екатеринбург", "address": "Улица Ленина, 50", "lat": 56.8389, "lon": 60.6057, "brand": "lukoil", "price": 51.00},
    {"name": "Газпромнефть", "city": "Новосибирск", "address": "Красный проспект, 100", "lat": 55.0302, "lon": 82.9204, "brand": "gazprom", "price": 50.00},
    {"name": "Роснефть", "city": "Сочи", "address": "Курортный проспект, 50", "lat": 43.5855, "lon": 39.7231, "brand": "rosneft", "price": 55.00},
    {"name": "Лукойл", "city": "Краснодар", "address": "Улица Красная, 100", "lat": 45.0355, "lon": 38.9753, "brand": "lukoil", "price": 52.00},
    {"name": "Газпромнефть", "city": "Владивосток", "address": "Океанский проспект, 50", "lat": 43.1155, "lon": 131.8855, "brand": "gazprom", "price": 54.50},
    {"name": "Роснефть", "city": "Нижний Новгород", "address": "Улица Большая Покровская, 50", "lat": 56.3268, "lon": 44.0065, "brand": "rosneft", "price": 51.50},
]

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
        "avg_consumption": "Средний расход",
        "distance": "Расстояние",
        "fuel": "Топлива",
        "cost": "Стоимость",
        "calculate": "Рассчитать",
        "clear": "Очистить",
        "fuel_consumption": "Расход топлива",
        "trip_cost": "Стоимость поездки",
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
        "delete_account": "Удалить аккаунт",
        "change_password": "Сменить пароль",
        "new_password": "Новый пароль (мин. 6 символов)",
        "repeat_new_password": "Повторите новый пароль",
        "change_password_btn": "Сменить пароль",
        "password_changed": "Пароль изменён",
        "my_cars": "Мои автомобили",
        "add_car": "+ Добавить авто",
        "no_cars": "Автомобили не добавлены",
        "delete": "Удалить",
        "car_name": "Название автомобиля",
        "no_photo": "Нет фото",
        "choose_photo": "Выбрать фото",
        "photo_not_chosen": "Фото не выбрано",
        "save": "Сохранить",
        "enter_name": "Введите название",
        "add_car_title": "Добавить автомобиль",
        "history_title": "История расчётов",
        "login_for_history": "Войдите в аккаунт для просмотра истории.",
        "clear_all": "Очистить всё",
        "history_empty": "История пуста",
        "copy": "Копировать",
        "copied": "Скопировано",
        "copied_msg": "Расчёт скопирован в буфер обмена!",
        "delete_entry": "Удалить",
        "delete_confirm": "Удалить?",
        "delete_car_confirm": "Удалить этот автомобиль?",
        "clear_history_confirm": "Очистить?",
        "clear_history_msg": "Удалить всю историю?",
        "delete_account_confirm": "Удалить аккаунт?",
        "delete_account_msg": "Вы уверены? Все данные будут удалены безвозвратно.",
        "settings_title": "Настройки",
        "theme": "Тема оформления",
        "theme_desc": "Выберите цветовую схему приложения",
        "dark_theme": "Тёмная",
        "light_theme": "Светлая",
        "current_theme": "Текущая тема:",
        "dark": "Тёмная",
        "light": "Светлая",
        "currency_setting": "Валюта",
        "currency_desc": "Выберите валюту для расчётов",
        "language_setting": "Язык / Language",
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
        "version": "Версия 2.4  •  2024",
        "about_section1_title": "О приложении",
        "about_section1": "Это профессиональное приложение для расчёта расхода топлива вашего автомобиля.",
        "about_section2_title": "Возможности",
        "about_section2": "• Точный расчёт расхода топлива\n• Расчёт стоимости поездки\n• Хранение автомобилей с фото\n• История расчётов\n• Карта заправок России",
        "about_section3_title": "Как использовать",
        "about_section3": "Режим «Средний расход»:\n1. Введите расстояние\n2. Укажите израсходованное топливо\n3. Введите цену топлива\n4. Нажмите «Рассчитать»\n\nРежим «Стоимость поездки»:\n1. Введите расстояние\n2. Укажите средний расход\n3. Введите цену топлива\n4. Нажмите «Рассчитать»",
        "about_section4_title": "Безопасность",
        "about_section4": "Регистрация по email. Пароли зашифрованы (SHA-256). Все данные хранятся локально.",
        "about_section5_title": "Интерпретация результатов",
        "about_section5": "До 8 л/100 км — экономичный расход\n8–12 л/100 км — средний расход\nБолее 12 л/100 км — высокий расход",
        "footer": "© 2024 Калькулятор расхода топлива",
        "fuel_unit": "л",
        "price_unit": "за л",
        "gs_title": "Карта заправок России",
        "gs_city": "Город:",
        "gs_search": "Найти заправки",
        "gs_to_calc": "В калькулятор",
        "gs_no_car": "Выберите авто для автозаполнения",
        "gs_select_station": "Выберите город и нажмите «Найти заправки»",
        "gs_login_hint": "Войдите для доступа к данным авто",
        "gs_hint": "Выберите город для поиска заправок",
        "gs_fuel_price_hint": "Цена топлива на заправке:",
        "gs_distance_to": "Расстояние до заправки",
        "gs_enter_distance": "Ваше расстояние до заправки (км):",
        "gs_send_to_calc": "Отправить в калькулятор",
        "gs_avg_from_car": "Средний расход авто",
        "gs_no_avg": "Нет данных о расходе",
        "gs_selected_station": "Выбранная заправка:",
        "gs_calc_distance": "Расстояние:",
        "gs_click_hint": "Нажмите на метку заправки на карте",
        "gs_distance_note": "Введите ваше примерное расстояние до заправки",
        "gs_all_cities": "Все города",
        "brands": {
            "lukoil": "Лукойл",
            "gazprom": "Газпромнефть",
            "rosneft": "Роснефть"
        }
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
        "avg_consumption": "Avg. consumption",
        "distance": "Distance",
        "fuel": "Fuel",
        "cost": "Cost",
        "calculate": "Calculate",
        "clear": "Clear",
        "fuel_consumption": "Fuel consumption",
        "trip_cost": "Trip cost",
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
        "delete_account": "Delete account",
        "change_password": "Change password",
        "new_password": "New password (min. 6 chars)",
        "repeat_new_password": "Repeat new password",
        "change_password_btn": "Change password",
        "password_changed": "Password changed",
        "my_cars": "My vehicles",
        "add_car": "+ Add vehicle",
        "no_cars": "No vehicles added",
        "delete": "Delete",
        "car_name": "Vehicle name",
        "no_photo": "No photo",
        "choose_photo": "Choose photo",
        "photo_not_chosen": "No photo selected",
        "save": "Save",
        "enter_name": "Enter name",
        "add_car_title": "Add vehicle",
        "history_title": "Calculation history",
        "login_for_history": "Log in to view history.",
        "clear_all": "Clear all",
        "history_empty": "History is empty",
        "copy": "Copy",
        "copied": "Copied",
        "copied_msg": "Calculation copied to clipboard!",
        "delete_entry": "Delete",
        "delete_confirm": "Delete?",
        "delete_car_confirm": "Delete this vehicle?",
        "clear_history_confirm": "Clear?",
        "clear_history_msg": "Delete all history?",
        "delete_account_confirm": "Delete account?",
        "delete_account_msg": "Are you sure? All data will be permanently deleted.",
        "settings_title": "Settings",
        "theme": "Theme",
        "theme_desc": "Choose the color scheme of the app",
        "dark_theme": "Dark",
        "light_theme": "Light",
        "current_theme": "Current theme:",
        "dark": "Dark",
        "light": "Light",
        "currency_setting": "Currency",
        "currency_desc": "Choose currency for calculations",
        "language_setting": "Language / Язык",
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
        "version": "Version 2.4  •  2024",
        "about_section1_title": "About",
        "about_section1": "A professional application for calculating your vehicle's fuel consumption.",
        "about_section2_title": "Features",
        "about_section2": "• Accurate fuel consumption calculation\n• Trip cost calculation\n• Multiple vehicles with photos\n• Calculation history\n• Russian gas stations map",
        "about_section3_title": "How to use",
        "about_section3": "Consumption mode:\n1. Enter distance\n2. Enter fuel used\n3. Enter fuel price\n4. Press Calculate\n\nTrip cost mode:\n1. Enter distance\n2. Enter avg. consumption\n3. Enter fuel price\n4. Press Calculate",
        "about_section4_title": "Security",
        "about_section4": "Email registration. Passwords encrypted (SHA-256). All data stored locally.",
        "about_section5_title": "Result interpretation",
        "about_section5": "Up to 8 L/100 km — economical\n8–12 L/100 km — average\nOver 12 L/100 km — high consumption",
        "footer": "© 2024 Fuel Consumption Calculator",
        "fuel_unit": "L",
        "price_unit": "per L",
        "gs_title": "Russian Gas Stations Map",
        "gs_city": "City:",
        "gs_search": "Find Gas Stations",
        "gs_to_calc": "To Calculator",
        "gs_no_car": "Select a vehicle for autofill",
        "gs_select_station": "Select city and click Find Gas Stations",
        "gs_login_hint": "Log in to access vehicle data",
        "gs_hint": "Select city to search for gas stations",
        "gs_fuel_price_hint": "Fuel price at the station:",
        "gs_distance_to": "Distance to station",
        "gs_enter_distance": "Your distance to station (km):",
        "gs_send_to_calc": "Send to Calculator",
        "gs_avg_from_car": "Vehicle avg. consumption",
        "gs_no_avg": "No consumption data",
        "gs_selected_station": "Selected station:",
        "gs_calc_distance": "Distance:",
        "gs_click_hint": "Click on station marker on the map",
        "gs_distance_note": "Enter your approximate distance to station",
        "gs_all_cities": "All cities",
        "brands": {
            "lukoil": "Lukoil",
            "gazprom": "Gazpromneft",
            "rosneft": "Rosneft"
        }
    }
}

# ─── Theme ─────────────────────────────────────────────────────────────────────
THEMES = {
    "dark": {
        "bg": "#0d1117", "bg2": "#161b22", "bg3": "#21262d",
        "fg": "#e6edf3", "fg2": "#8b949e", "accent": "#58a6ff",
        "green": "#3fb950", "yellow": "#d29922", "red": "#f85149",
        "border": "#30363d", "btn": "#238636", "btn_hover": "#2ea043",
        "btn2": "#21262d", "btn2_hover": "#30363d", "input_bg": "#0d1117",
        "chart_bg": "#161b22", "chart_line": "#58a6ff",
        "chart_fill": "#1c2d3f", "chart_grid": "#21262d",
        "map_bg": "#1a2a3a", "map_land": "#2a4a2a", "map_water": "#1a3a5a",
        "map_road": "#3a3a3a", "map_marker": "#ff4444", "map_marker_selected": "#ffaa00",
    },
    "light": {
        "bg": "#f0f4f8", "bg2": "#ffffff", "bg3": "#e2e8f0",
        "fg": "#1a202c", "fg2": "#718096", "accent": "#3182ce",
        "green": "#38a169", "yellow": "#d69e2e", "red": "#e53e3e",
        "border": "#cbd5e0", "btn": "#3182ce", "btn_hover": "#2b6cb0",
        "btn2": "#e2e8f0", "btn2_hover": "#cbd5e0", "input_bg": "#ffffff",
        "chart_bg": "#ffffff", "chart_line": "#3182ce",
        "chart_fill": "#ebf4ff", "chart_grid": "#e2e8f0",
        "map_bg": "#e8f0e8", "map_land": "#c8e6c9", "map_water": "#bbdefb",
        "map_road": "#cfd8dc", "map_marker": "#e53935", "map_marker_selected": "#ff8f00",
    }
}

current_theme = "dark"
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
    canvas.bind("<Enter>", _bind_to_mousewheel)
    canvas.bind("<Leave>", _unbind_from_mousewheel)

# ─── Haversine distance ────────────────────────────────────────────────────────
def haversine_km(lat1, lon1, lat2, lon2):
    R = 6371.0
    dlat = math.radians(lat2 - lat1)
    dlon = math.radians(lon2 - lon1)
    a = math.sin(dlat / 2) ** 2 + math.cos(math.radians(lat1)) * math.cos(math.radians(lat2)) * math.sin(dlon / 2) ** 2
    return R * 2 * math.atan2(math.sqrt(a), math.sqrt(1 - a))

# ─── Helper: placeholder image ─────────────────────────────────────────────────
def make_placeholder(w=300, h=180, text=""):
    t = T()
    try:
        img = Image.new("RGB", (w, h), t["bg3"])
        draw = ImageDraw.Draw(img)
        draw.rectangle([0, 0, w - 1, h - 1], outline=t["border"], width=2)
        if text:
            draw.text((w // 2 - 25, h // 2 - 8), text, fill=t["fg2"])
        return img
    except Exception:
        img = Image.new("RGB", (w, h), "#333333")
        return img

def load_car_image(path, w=300, h=180):
    """
    Безопасная загрузка изображения с обработкой всех возможных ошибок.
    """
    if not path or not isinstance(path, str):
        return make_placeholder(w, h)

    if not os.path.exists(path):
        return make_placeholder(w, h)

    # Проверка размера файла (пропускаем пустые или слишком маленькие файлы)
    try:
        file_size = os.path.getsize(path)
        if file_size < 100:
            return make_placeholder(w, h)
    except OSError:
        return make_placeholder(w, h)

    try:
        # Открываем изображение
        img = Image.open(path)

        # Проверяем, что изображение валидно
        img.verify()

        # После verify() нужно переоткрыть
        img = Image.open(path)

        # Конвертируем в RGB
        if img.mode not in ('RGB', 'RGBA'):
            img = img.convert('RGB')
        elif img.mode == 'RGBA':
            background = Image.new('RGB', img.size, (0, 0, 0))
            background.paste(img, mask=img.split()[3])
            img = background

        # Ресайз с сохранением пропорций
        src_w, src_h = img.size
        if src_w <= 0 or src_h <= 0:
            return make_placeholder(w, h)

        src_ratio = src_w / src_h
        dst_ratio = w / h

        if src_ratio > dst_ratio:
            new_h = h
            new_w = int(src_w * h / src_h)
        else:
            new_w = w
            new_h = int(src_h * w / src_w)

        # Безопасный ресайз
        new_w = max(1, min(new_w, 5000))
        new_h = max(1, min(new_h, 5000))

        img = img.resize((new_w, new_h), Image.LANCZOS)

        # Кроп
        left = (new_w - w) // 2
        top = (new_h - h) // 2
        right = left + w
        bottom = top + h

        if left >= 0 and top >= 0 and right <= new_w and bottom <= new_h:
            img = img.crop((left, top, right, bottom))

        return img

    except (UnidentifiedImageError, OSError, IOError, ValueError, SyntaxError,
            TypeError, AttributeError, MemoryError, Exception) as e:
        # Любая ошибка — возвращаем заглушку
        print(f"Warning: Cannot load image {path}: {e}")
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
        current_theme = settings.get("theme", "dark")
        current_language = settings.get("language", "ru")
        current_currency = settings.get("currency", "₽ RUB")

        self._photo_refs = {}
        self._calc_mode = tk.StringVar(value="consumption")

        self._selected_station = None
        self._stations_list = []
        self._map_city_filter = tk.StringVar(value="")
        self._map_markers = []

        self._build_ui()

    def _save_settings(self):
        self.data.setdefault("settings", {})
        self.data["settings"]["theme"] = current_theme
        self.data["settings"]["language"] = current_language
        self.data["settings"]["currency"] = current_currency
        save_data(self.data)

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
        self._map_markers.clear()
        self._build_ui()

    def _build_sidebar(self):
        t = T()
        for w in self.sidebar.winfo_children():
            w.destroy()
        items = [
            (TR("calculator"), "calculator"),
            (TR("profile"), "profile"),
            (TR("history"), "history"),
            (TR("gas_stations"), "gas_stations"),
            (TR("about"), "about"),
        ]
        tk.Frame(self.sidebar, bg=t["bg2"], height=20).pack()
        icons = {"calculator": "🧮", "profile": "👤", "history": "📋", "gas_stations": "⛽", "about": "ℹ️"}
        for label, key in items:
            icon = icons.get(key, "")
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

    # ── Calculator ─────────────────────────────────────────────────────────────
    def _build_calculator(self):
        t = T()
        f = self.sections["calculator"]
        for w in f.winfo_children():
            w.destroy()

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
        left = tk.Frame(top, bg=t["bg2"])
        left.pack(side="left", fill="y", padx=(0, 16))

        img_container = tk.Frame(left, bg=t["bg2"], width=300, height=180)
        img_container.pack_propagate(False)
        img_container.pack()

        self.calc_car_label = tk.Label(img_container, bg=t["bg2"])
        self.calc_car_label.place(relx=0.5, rely=0.5, anchor="center")
        self._set_calc_car_image(None)

        tk.Label(left, text=TR("car") + ":", font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(pady=(8, 0))
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
                 font=("Georgia", 12, "bold"), bg=t["bg3"], fg=t["fg"]).pack(anchor="w", padx=12, pady=(8, 4))

        self.stat_labels = {}
        stats = [
            ("avg", TR("avg_consumption"), "— л/100 км", t["green"]),
            ("dist", f"📏  {TR('distance')}", "— км", t["accent"]),
            ("fuel", f"⛽  {TR('fuel')}", "— л", t["yellow"]),
            ("cost", f"💳  {TR('cost')}", "—", t["red"]),
        ]
        for key, label, default, color in stats:
            row = tk.Frame(stats_frame, bg=t["bg3"])
            row.pack(fill="x", padx=12, pady=3)
            tk.Label(row, text=label, font=("Courier", 10), bg=t["bg3"], fg=t["fg2"],
                     width=22, anchor="w").pack(side="left")
            lbl = tk.Label(row, text=default, font=("Courier", 10, "bold"), bg=t["bg3"], fg=color)
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
        tk.Radiobutton(mode_frame, text=TR("calc_mode_consumption"),
                       variable=self._calc_mode, value="consumption",
                       font=("Courier", 10), bg=t["bg2"], fg=t["fg"],
                       selectcolor=t["bg3"], activebackground=t["bg2"],
                       cursor="hand2", command=self._on_mode_changed).pack(side="left", padx=(0, 16))
        tk.Radiobutton(mode_frame, text=TR("calc_mode_cost"),
                       variable=self._calc_mode, value="cost",
                       font=("Courier", 10), bg=t["bg2"], fg=t["fg"],
                       selectcolor=t["bg3"], activebackground=t["bg2"],
                       cursor="hand2", command=self._on_mode_changed).pack(side="left")

        self.calc_form_frame = tk.Frame(right, bg=t["bg2"])
        self.calc_form_frame.pack(fill="x", padx=20)

        self.dist_var = tk.StringVar(value="")
        self.fuel_var = tk.StringVar(value="")
        self.price_var = tk.StringVar(value="")
        self.avg_var = tk.StringVar(value="")

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

        # Fuel chart
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

        self.chart_canvas = tk.Canvas(chart_outer, bg=t["chart_bg"], height=280,
                                      highlightthickness=1, highlightbackground=t["border"])
        self.chart_canvas.pack(fill="x", padx=16, pady=(0, 16))
        self.chart_canvas.bind("<Configure>", lambda e: self._draw_fuel_chart())
        self.chart_canvas.bind("<Motion>", self._on_chart_hover)
        self.chart_canvas.bind("<Leave>", self._on_chart_leave)
        self._chart_data_points = []

        self.after(100, self._draw_fuel_chart)

    def _build_calc_form(self):
        t = T()
        for w in self.calc_form_frame.winfo_children():
            w.destroy()
        mode = self._calc_mode.get() if hasattr(self, '_calc_mode') else "consumption"
        sym = get_currency_symbol()

        def make_entry_row(parent, label_text, var, unit_text, icon):
            tk.Label(parent, text=label_text, font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", pady=(8, 2))
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
            make_entry_row(self.calc_form_frame, TR("fuel_price"), self.price_var, sym + "/" + TR("fuel_unit"), "💰")
        else:
            make_entry_row(self.calc_form_frame, TR("avg_consumption_label"), self.avg_var, "л/100 км", "💧")
            make_entry_row(self.calc_form_frame, TR("fuel_price"), self.price_var, sym + "/" + TR("fuel_unit"), "💰")

    def _on_mode_changed(self):
        self._build_calc_form()

    def _set_calc_car_image(self, photo_path):
        t = T()
        img = load_car_image(photo_path, 300, 180)
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
            dist = float(self.dist_var.get().replace(",", "."))
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
                "date": datetime.now().strftime("%d.%m.%Y %H:%M"),
                "car": sel_car,
                "distance": dist,
                "fuel": round(fuel, 2),
                "price": price,
                "currency": sym,
                "consumption": round(consumption, 2),
                "cost": round(cost, 2),
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

    # ── Fuel Chart ─────────────────────────────────────────────────────────────
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
            c.create_oval(cx - r, cy - r, cx + r, cy + r,
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
            c.create_text(W // 2, H // 2, text=TR("login_for_chart"),
                          fill=t["fg2"], font=("Courier", 12))
            return

        selected_car = self.calc_car_var.get() if hasattr(self, 'calc_car_var') else TR("no_car")
        try:
            days_limit = int(self.chart_days_var.get())
        except Exception:
            days_limit = 30

        history = self.data["users"][self.current_user].get("history", [])
        cutoff = datetime.now() - timedelta(days=days_limit)

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
            c.create_text(W // 2, H // 2, text=TR("no_history_chart"),
                          fill=t["fg2"], font=("Courier", 12))
            return

        filtered.sort(key=lambda x: x[0])
        values = [v for _, v in filtered]
        min_v = max(0, min(values) - 1)
        max_v = max(values) + 1
        rng = max_v - min_v if max_v != min_v else 1

        chart_w = W - pad_l - pad_r
        chart_h = H - pad_t - pad_b

        def px(i):
            n = len(filtered)
            return pad_l + (i / max(n - 1, 1)) * chart_w

        def py(v):
            return pad_t + (1 - (v - min_v) / rng) * chart_h

        # Grid
        for gi in range(6):
            gv = min_v + gi * rng / 5
            gy = py(gv)
            c.create_line(pad_l, gy, W - pad_r, gy, fill=t["chart_grid"], dash=(4, 6), width=1)
            c.create_text(pad_l - 6, gy, text=f"{gv:.1f}", anchor="e", fill=t["fg2"], font=("Courier", 9))

        # Color zones
        y_green = py(8)
        y_yellow = py(12)
        y_bottom = H - pad_b
        if y_green > pad_t:
            c.create_rectangle(pad_l, pad_t, W - pad_r, min(y_green, y_bottom), fill="#1a2f1a", outline="")
        if y_yellow > y_green:
            c.create_rectangle(pad_l, max(y_green, pad_t), W - pad_r, min(y_yellow, y_bottom), fill="#2a2410", outline="")
        c.create_rectangle(pad_l, max(y_yellow, pad_t), W - pad_r, y_bottom, fill="#2a1212", outline="")

        # Fill under curve
        pts = []
        for i, (_, v) in enumerate(filtered):
            pts += [px(i), py(v)]
        if len(pts) >= 4:
            poly_pts = [pad_l, H - pad_b] + pts + [W - pad_r, H - pad_b]
            c.create_polygon(poly_pts, fill=t["chart_fill"], outline="")

        # Lines
        for i in range(len(filtered) - 1):
            v1, v2 = filtered[i][1], filtered[i + 1][1]
            avg_v = (v1 + v2) / 2
            lcolor = t["green"] if avg_v <= 8 else (t["yellow"] if avg_v <= 12 else t["red"])
            c.create_line(px(i), py(v1), px(i + 1), py(v2), fill=lcolor, width=2.5, smooth=True)

        # Dots
        for i, (dt, v) in enumerate(filtered):
            x, y = px(i), py(v)
            r = 5
            dot_color = t["green"] if v <= 8 else (t["yellow"] if v <= 12 else t["red"])
            c.create_oval(x - r, y - r, x + r, y + r, fill=dot_color, outline=t["bg2"], width=2)
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
            c.create_text(x, H - pad_b + 14, text=dt.strftime("%d.%m"), fill=t["fg2"], font=("Courier", 8))

        # Y-axis label
        c.create_text(pad_l - 40, pad_t + chart_h // 2, text="л/100", fill=t["fg2"], font=("Courier", 8), angle=90)

        # Legend
        legend_x = W - pad_r - 130
        legend_items = [("≤8", t["green"]), ("8-12", t["yellow"]), (">12", t["red"])]
        for li, (ltxt, lc) in enumerate(legend_items):
            lx = legend_x + li * 44
            c.create_rectangle(lx, pad_t + 2, lx + 10, pad_t + 12, fill=lc, outline="")
            c.create_text(lx + 13, pad_t + 7, text=ltxt, fill=t["fg2"], font=("Courier", 7), anchor="w")

        if values:
            avg_all = sum(values) / len(values)
            c.create_text(W - pad_r - 4, pad_t + 6,
                          text=f"avg: {avg_all:.1f} л/100",
                          anchor="ne", fill=t["fg2"], font=("Courier", 9))

    # ── Profile ────────────────────────────────────────────────────────────────
    def _build_profile(self):
        t = T()
        f = self.sections["profile"]
        for w in f.winfo_children():
            w.destroy()

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
            for w in self._auth_form_frame.winfo_children():
                w.destroy()
            mode = self._auth_mode.get()
            title = TR("create_account") if mode == "register" else TR("login_account")
            tk.Label(self._auth_form_frame, text=title,
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
                pw = pw_entry.get()
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

            btn_text = TR("login") if mode == "login" else TR("register")
            tk.Button(self._auth_form_frame, text=btn_text,
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

        # Change password
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

        # Cars
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
        tk.Entry(self._add_car_form_frame, textvariable=name_var,
                 font=("Courier", 12), bg=t["input_bg"], fg=t["fg"],
                 insertbackground=t["fg"], bd=1, relief="solid").pack(fill="x", padx=16, pady=(2, 8), ipady=6)

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
                try:
                    shutil.copy2(path, dest)
                    photo_path_var.set(dest)
                    photo_name_lbl.configure(text=os.path.basename(path))
                    prev_img = load_car_image(dest, 120, 75)
                    prev_photo = ImageTk.PhotoImage(prev_img)
                    self._photo_refs["add_car_preview"] = prev_photo
                    preview_lbl.configure(image=prev_photo, text="")
                except Exception as e:
                    messagebox.showwarning(TR("error"), f"Не удалось загрузить фото: {e}")

        tk.Button(right_col, text=TR("choose_photo"), font=("Courier", 10),
                  bg=t["btn2"], fg=t["fg"], bd=0, padx=12, pady=6,
                  cursor="hand2", command=choose_photo).pack(anchor="w")

        status = tk.Label(self._add_car_form_frame, text="", font=("Courier", 9), bg=t["bg2"], fg=t["red"])
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
        for w in self.cars_container.winfo_children():
            w.destroy()
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
            img = load_car_image(ph, 140, 90)
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
                      cursor="hand2", command=lambda idx=i: self._delete_car(idx)).pack(pady=(4, 0))

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
                    try:
                        os.remove(car["photo"])
                    except Exception:
                        pass
            del self.data["users"][self.current_user]
            self.current_user = None
            self.data["current_user"] = None
            save_data(self.data)
            self._refresh_car_combo()
            self._refresh_profile()

    def _refresh_profile(self):
        f = self.sections["profile"]
        for w in f.winfo_children():
            w.destroy()
        self._build_profile()

    # ── History ────────────────────────────────────────────────────────────────
    def _build_history(self):
        t = T()
        f = self.sections["history"]
        for w in f.winfo_children():
            w.destroy()

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
        self._hist_inner.bind("<Configure>", lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
        canvas.bind("<Configure>", lambda e: canvas.itemconfig(win, width=e.width))
        bind_mousewheel(self._hist_inner, canvas)

        self.after(50, self._render_history_entries)

    def _render_history_entries(self):
        t = T()
        if not hasattr(self, '_hist_inner'):
            return
        for w in self._hist_inner.winfo_children():
            w.destroy()
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
            img = load_car_image(ph, 200, 120)
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
        for w in f.winfo_children():
            w.destroy()
        self._build_history()

    # ── Gas Stations ───────────────────────────────────────────────────────────
    def _build_gas_stations(self):
        t = T()
        f = self.sections["gas_stations"]
        for w in f.winfo_children():
            w.destroy()

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

        tk.Label(pad, text=TR("gs_title"), font=("Georgia", 18, "bold"),
                 bg=t["bg"], fg=t["fg"]).pack(anchor="w", pady=(0, 8))

        # Search
        search_card = tk.Frame(pad, bg=t["bg2"], pady=16)
        search_card.pack(fill="x", pady=8)

        tk.Label(search_card, text=TR("gs_city"), font=("Georgia", 13, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=20, pady=(0, 8))

        search_row = tk.Frame(search_card, bg=t["bg2"])
        search_row.pack(fill="x", padx=20)

        cities = sorted(list(set([s["city"] for s in GAS_STATIONS_RUSSIA])))
        city_values = [TR("gs_all_cities")] + cities

        self._map_city_filter = tk.StringVar(value=city_values[0])
        city_combo = ttk.Combobox(search_row, textvariable=self._map_city_filter,
                                   values=city_values, state="readonly",
                                   font=("Courier", 11), width=20)
        city_combo.pack(side="left", padx=(0, 12))
        city_combo.bind("<<ComboboxSelected>>", lambda e: self._draw_map())

        tk.Button(search_row, text=TR("gs_search"), font=("Courier", 11),
                  bg=t["btn"], fg="white", bd=0, padx=16, pady=8,
                  cursor="hand2", command=self._draw_map).pack(side="left")

        # Map
        self.map_frame = tk.Frame(pad, bg=t["bg2"])
        self.map_frame.pack(fill="x", pady=8)

        self.map_canvas = tk.Canvas(self.map_frame, bg=t["map_bg"], height=450,
                                     highlightthickness=1, highlightbackground=t["border"])
        self.map_canvas.pack(fill="x", padx=8, pady=8)
        self.map_canvas.bind("<Button-1>", self._on_map_click)
        self.map_canvas.bind("<Motion>", self._on_map_hover)
        self.map_canvas.bind("<Leave>", self._on_map_leave)

        # Station info
        self.station_info_card = tk.Frame(pad, bg=t["bg2"], pady=16)
        self.station_info_card.pack(fill="x", pady=8)

        self.station_info_text = tk.Label(self.station_info_card,
                                           text=TR("gs_click_hint"),
                                           font=("Courier", 11), bg=t["bg2"], fg=t["fg2"],
                                           justify="left")
        self.station_info_text.pack(anchor="w", padx=20, pady=8)

        # Car
        car_card = tk.Frame(pad, bg=t["bg2"], pady=16)
        car_card.pack(fill="x", pady=8)

        tk.Label(car_card, text=TR("car"), font=("Georgia", 13, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=20, pady=(0, 8))

        self.gs_car_var = tk.StringVar(value=TR("no_car"))
        self.gs_car_combo = ttk.Combobox(car_card, textvariable=self.gs_car_var,
                                          state="readonly", font=("Courier", 11))
        self.gs_car_combo.pack(anchor="w", padx=20, pady=(0, 8))
        self.gs_car_combo.bind("<<ComboboxSelected>>", self._on_gs_car_selected)
        self._refresh_gs_car_combo()

        self.gs_avg_lbl = tk.Label(car_card, text="", font=("Courier", 10),
                                    bg=t["bg2"], fg=t["accent"])
        self.gs_avg_lbl.pack(anchor="w", padx=20, pady=(0, 8))

        # Distance
        dist_card = tk.Frame(pad, bg=t["bg2"], pady=16)
        dist_card.pack(fill="x", pady=8)

        tk.Label(dist_card, text=TR("gs_enter_distance"), font=("Georgia", 13, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=20, pady=(0, 8))
        tk.Label(dist_card, text=TR("gs_distance_note"),
                 font=("Courier", 9), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", padx=20, pady=(0, 4))

        dist_row = tk.Frame(dist_card, bg=t["input_bg"], bd=1, relief="solid")
        dist_row.pack(fill="x", padx=20, pady=(0, 12))
        tk.Label(dist_row, text="📏", font=("Courier", 13), bg=t["input_bg"], fg=t["fg2"], padx=8).pack(side="left")
        self.gs_dist_var = tk.StringVar()
        tk.Entry(dist_row, textvariable=self.gs_dist_var, font=("Courier", 13),
                 bg=t["input_bg"], fg=t["fg"], insertbackground=t["fg"],
                 bd=0, relief="flat").pack(side="left", fill="x", expand=True, pady=8)
        tk.Label(dist_row, text="км", font=("Courier", 10), bg=t["input_bg"], fg=t["fg2"], padx=8).pack(side="right")

        # Price
        price_card = tk.Frame(pad, bg=t["bg2"], pady=16)
        price_card.pack(fill="x", pady=8)

        tk.Label(price_card, text=TR("gs_fuel_price_hint"), font=("Georgia", 13, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=20, pady=(0, 8))

        price_row = tk.Frame(price_card, bg=t["input_bg"], bd=1, relief="solid")
        price_row.pack(fill="x", padx=20, pady=(0, 12))
        tk.Label(price_row, text="💰", font=("Courier", 13), bg=t["input_bg"], fg=t["fg2"], padx=8).pack(side="left")
        self.gs_price_var = tk.StringVar()
        tk.Entry(price_row, textvariable=self.gs_price_var, font=("Courier", 13),
                 bg=t["input_bg"], fg=t["fg"], insertbackground=t["fg"],
                 bd=0, relief="flat").pack(side="left", fill="x", expand=True, pady=8)
        tk.Label(price_row, text=f"{get_currency_symbol()}/л", font=("Courier", 10),
                 bg=t["input_bg"], fg=t["fg2"], padx=8).pack(side="right")

        # Send button
        tk.Button(pad, text=TR("gs_to_calc"), font=("Georgia", 12, "bold"),
                  bg=t["btn"], fg="white", bd=0, pady=12,
                  cursor="hand2", command=self._gs_to_calculator).pack(fill="x", pady=16)

        self.after(200, self._draw_map)

    def _get_filtered_stations(self):
        city = self._map_city_filter.get()
        if city == TR("gs_all_cities") or city == "Все города" or city == "All cities":
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

        W = c.winfo_width()
        H = c.winfo_height()

        if W < 10 or H < 10:
            W, H = 800, 450

        c.create_rectangle(0, 0, W, H, fill=t["map_bg"], outline="")

        # Russia outline
        russia_outline = [
            (41.0, 43.0), (41.0, 45.0), (37.0, 46.0), (36.0, 47.0),
            (32.0, 46.0), (28.0, 44.0), (22.0, 44.0), (20.0, 55.0),
            (19.0, 60.0), (20.0, 65.0), (28.0, 69.0), (32.0, 70.0),
            (40.0, 69.0), (45.0, 68.0), (50.0, 72.0), (60.0, 75.0),
            (70.0, 76.0), (80.0, 74.0), (90.0, 73.0), (100.0, 74.0),
            (110.0, 74.0), (120.0, 72.0), (130.0, 70.0), (140.0, 68.0),
            (150.0, 65.0), (160.0, 62.0), (170.0, 60.0), (170.0, 55.0),
            (160.0, 50.0), (150.0, 48.0), (140.0, 45.0), (135.0, 43.0),
            (130.0, 42.0), (120.0, 40.0), (110.0, 42.0), (100.0, 45.0),
            (90.0, 50.0), (80.0, 55.0), (70.0, 55.0), (60.0, 50.0),
            (50.0, 45.0), (45.0, 42.0), (41.0, 43.0)
        ]

        points = []
        for lat, lon in russia_outline:
            x, y = self._lat_lon_to_xy(lat, lon, W, H)
            points.extend([x, y])

        if len(points) >= 4:
            c.create_polygon(points, fill=t["map_land"], outline=t["border"], width=1.5)

        # Cities
        stations = self._get_filtered_stations()
        city_groups = {}
        for st in stations:
            city = st["city"]
            if city not in city_groups:
                city_groups[city] = {"lat": st["lat"], "lon": st["lon"], "stations": [], "count": 0}
            city_groups[city]["count"] += 1
            city_groups[city]["stations"].append(st)

        for city, data in city_groups.items():
            x, y = self._lat_lon_to_xy(data["lat"], data["lon"], W, H)
            radius = max(8, min(20, 6 + data["count"] * 3))

            city_id = c.create_oval(x - radius, y - radius, x + radius, y + radius,
                                     fill=t["accent"], outline=t["bg2"], width=2,
                                     tags=("city", city))
            c.create_text(x, y - radius - 10, text=city, fill=t["fg"],
                          font=("Courier", 8, "bold"), tags=("label", city))
            c.create_text(x, y, text=str(data["count"]), fill="white",
                          font=("Courier", 8, "bold"), tags=("count", city))

            self._map_markers.append({
                "city": city,
                "x": x,
                "y": y,
                "radius": radius,
                "stations": data["stations"],
                "canvas_id": city_id
            })

        if self._selected_station:
            self._highlight_selected_station()

        # Legend
        legend_y = 50
        c.create_rectangle(W - 180, legend_y - 10, W - 10, legend_y + 60,
                           fill=t["bg2"], outline=t["border"], width=1)
        c.create_text(W - 95, legend_y + 5, text="🏭 Бренды:", fill=t["fg"],
                      font=("Courier", 9, "bold"), anchor="n")

        brands = [("Лукойл", t["red"]), ("Газпромнефть", t["accent"]), ("Роснефть", t["yellow"])]
        for i, (name, color) in enumerate(brands):
            ly = legend_y + 20 + i * 14
            c.create_rectangle(W - 170, ly, W - 160, ly + 8, fill=color, outline="")
            c.create_text(W - 155, ly + 4, text=name, fill=t["fg"], font=("Courier", 8), anchor="w")

    def _on_map_click(self, event):
        closest = None
        closest_dist = 9999
        for marker in self._map_markers:
            dx = event.x - marker["x"]
            dy = event.y - marker["y"]
            dist = math.sqrt(dx * dx + dy * dy)
            if dist < marker["radius"] + 5 and dist < closest_dist:
                closest = marker
                closest_dist = dist

        if closest:
            self._select_marker(closest)
        else:
            self._selected_station = None
            self._draw_map()
            self.station_info_text.configure(text=TR("gs_click_hint"))

    def _on_map_hover(self, event):
        for marker in self._map_markers:
            dx = event.x - marker["x"]
            dy = event.y - marker["y"]
            if math.sqrt(dx * dx + dy * dy) < marker["radius"] + 3:
                self.map_canvas.config(cursor="hand2")
                return
        self.map_canvas.config(cursor="")

    def _on_map_leave(self, event):
        self.map_canvas.config(cursor="")

    def _select_marker(self, marker):
        self._selected_station = marker
        stations = marker["stations"]
        city = marker["city"]

        info_text = f"🏙 {TR('gs_selected_station')} {city}\n\n"
        for i, st in enumerate(stations[:5]):
            brand = TR("brands").get(st["brand"], st["brand"])
            info_text += f"⛽ {brand} — {st['address']}\n   💰 {st['price']} {get_currency_symbol()}/{TR('fuel_unit')}\n\n"

        if len(stations) > 5:
            info_text += f"... и ещё {len(stations) - 5} заправок\n"

        info_text += f"\n💡 {TR('gs_distance_note')}"
        self.station_info_text.configure(text=info_text, fg=T()["fg"])

        if stations:
            self.gs_price_var.set(str(stations[0]["price"]))

        self._draw_map()
        self._highlight_selected_station()

    def _highlight_selected_station(self):
        if not self._selected_station:
            return
        c = self.map_canvas
        t = T()
        marker = self._selected_station
        x, y, r = marker["x"], marker["y"], marker["radius"]

        c.create_oval(x - r - 4, y - r - 4, x + r + 4, y + r + 4,
                      outline=t["map_marker_selected"], width=3, tags="selection")
        c.create_oval(x - r - 2, y - r - 2, x + r + 2, y + r + 2,
                      outline="white", width=1, dash=(3, 3), tags="selection")

        stations = marker["stations"]
        if stations:
            info_y = y + r + 30
            c.create_text(x, info_y,
                          text=f"{marker['city']} — {len(stations)} заправок",
                          fill=t["fg"], font=("Courier", 9), bg=t["bg2"], tags="selection")
            c.create_text(x, info_y + 15,
                          text=f"от {stations[0]['price']} {get_currency_symbol()}/л",
                          fill=t["accent"], font=("Courier", 8), bg=t["bg2"], tags="selection")

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
        if not hasattr(self, 'gs_avg_lbl'):
            return
        t = T()
        sel = self.gs_car_var.get() if hasattr(self, 'gs_car_var') else TR("no_car")
        if sel == TR("no_car") or not self.current_user:
            self.gs_avg_lbl.configure(text="")
            return
        user = self.data["users"].get(self.current_user, {})
        for car in user.get("cars", []):
            if car["name"] == sel:
                avg = car.get("avg_consumption")
                if avg:
                    self.gs_avg_lbl.configure(text=f"💧 {TR('gs_avg_from_car')}: {avg} л/100 км", fg=t["accent"])
                    return
        self.gs_avg_lbl.configure(text=TR("gs_no_avg"), fg=t["fg2"])

    def _gs_to_calculator(self):
        dist_str = self.gs_dist_var.get().strip()
        price_str = self.gs_price_var.get().strip()

        if not dist_str:
            messagebox.showwarning(TR("error"), TR("gs_enter_distance"))
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
        if hasattr(self, 'map_canvas') and self.map_canvas.winfo_exists():
            self._draw_map()

    # ── About ──────────────────────────────────────────────────────────────────
    def _build_about(self):
        t = T()
        f = self.sections["about"]
        for w in f.winfo_children():
            w.destroy()

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
        tk.Label(hero, text=TR("app_title"), font=("Georgia", 22, "bold"), bg=t["bg2"], fg=t["fg"]).pack()
        tk.Label(hero, text=TR("version"), font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(pady=4)

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

        tk.Label(pad, text=TR("footer"), font=("Courier", 9), bg=t["bg"], fg=t["fg2"]).pack(pady=16)

    # ── Settings ───────────────────────────────────────────────────────────────
    def _build_settings(self):
        t = T()
        f = self.sections["settings"]
        for w in f.winfo_children():
            w.destroy()

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

        # Theme
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

        theme_name = TR("dark") if current_theme == "dark" else TR("light")
        tk.Label(theme_card, text=f"{TR('current_theme')} {theme_name}",
                 font=("Courier", 10), bg=t["bg2"], fg=t["accent"]).pack(anchor="w", padx=20, pady=4)

        # Currency
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

        # Language
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

        lang_name = "Русский 🇷🇺" if current_language == "ru" else "English 🇬🇧"
        tk.Label(lang_card, text=f"{TR('current')} {lang_name}",
                 font=("Courier", 10), bg=t["bg2"], fg=t["accent"]).pack(anchor="w", padx=20, pady=4)


# ─── Entry point ───────────────────────────────────────────────────────────────
if __name__ == "__main__":
    app = FuelApp()
    app.mainloop()
