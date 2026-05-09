import tkinter as tk
from tkinter import ttk, messagebox, filedialog
import json
import os
import re
import hashlib
import shutil
from datetime import datetime, timedelta
import math
import psycopg2
from psycopg2 import sql, extras
from dataclasses import dataclass
from typing import Optional, List, Dict, Any
import threading

try:
    from PIL import Image, ImageTk, ImageDraw, ImageFont
except ImportError:
    import subprocess, sys
    subprocess.check_call([sys.executable, "-m", "pip", "install", "Pillow"])
    from PIL import Image, ImageTk, ImageDraw, ImageFont

# ============================================================================
# КРАСИВЫЕ ЭМОДЗИ (Unicode)
# ============================================================================
EMOJI = {
    "car": "🚗", "car_sport": "🏎️", "fuel": "⛽", "money": "💰", 
    "chart": "📊", "history": "📜", "settings": "⚙️", "profile": "👤",
    "logout": "🚪", "delete": "🗑️", "edit": "✏️", "save": "💾",
    "add": "➕", "close": "❌", "check": "✅", "warning": "⚠️",
    "info": "ℹ️", "calculator": "🧮", "stats": "📈", "calendar": "📅",
    "distance": "📏", "cost": "💳", "average": "📊", "success": "🎉",
    "error": "❌", "search": "🔍", "filter": "🎯", "star": "⭐",
    "heart": "❤️", "rocket": "🚀", "crown": "👑", "sparkles": "✨"
}

# ============================================================================
# КОНФИГУРАЦИЯ БАЗЫ ДАННЫХ POSTGRESQL
# ============================================================================
DB_CONFIG = {
    "host": "localhost",
    "port": 5432,
    "database": "fuel_calculator",
    "user": "postgres",
    "password": "postgres"
}

class Database:
    def __init__(self):
        self.conn = None
        self.connect()
        self.create_tables()
    
    def connect(self):
        try:
            self.conn = psycopg2.connect(**DB_CONFIG)
            self.conn.autocommit = False
        except Exception as e:
            print(f"Ошибка подключения к PostgreSQL: {e}")
            # Пробуем создать базу данных если её нет
            self.create_database()
    
    def create_database(self):
        try:
            conn = psycopg2.connect(
                host=DB_CONFIG["host"],
                port=DB_CONFIG["port"],
                user=DB_CONFIG["user"],
                password=DB_CONFIG["password"],
                database="postgres"
            )
            conn.autocommit = True
            cur = conn.cursor()
            cur.execute(sql.SQL("CREATE DATABASE {}").format(
                sql.Identifier(DB_CONFIG["database"])
            ))
            cur.close()
            conn.close()
            self.connect()
        except Exception as e:
            print(f"Не удалось создать базу данных: {e}")
    
    def create_tables(self):
        if not self.conn:
            return
        
        cur = self.conn.cursor()
        
        # Таблица пользователей
        cur.execute("""
            CREATE TABLE IF NOT EXISTS users (
                id SERIAL PRIMARY KEY,
                email VARCHAR(255) UNIQUE NOT NULL,
                password_hash VARCHAR(255) NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                last_login TIMESTAMP
            )
        """)
        
        # Таблица автомобилей
        cur.execute("""
            CREATE TABLE IF NOT EXISTS cars (
                id SERIAL PRIMARY KEY,
                user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
                name VARCHAR(100) NOT NULL,
                photo_path TEXT,
                avg_consumption DECIMAL(10,2),
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        
        # Таблица истории расчётов
        cur.execute("""
            CREATE TABLE IF NOT EXISTS history (
                id SERIAL PRIMARY KEY,
                user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
                car_id INTEGER REFERENCES cars(id) ON DELETE SET NULL,
                car_name VARCHAR(100),
                distance DECIMAL(10,2) NOT NULL,
                fuel_used DECIMAL(10,2) NOT NULL,
                consumption DECIMAL(10,2) NOT NULL,
                price_per_liter DECIMAL(10,2) NOT NULL,
                total_cost DECIMAL(10,2) NOT NULL,
                currency VARCHAR(10) DEFAULT '₽',
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        
        # Таблица настроек пользователя
        cur.execute("""
            CREATE TABLE IF NOT EXISTS user_settings (
                user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
                theme VARCHAR(20) DEFAULT 'dark',
                language VARCHAR(10) DEFAULT 'ru',
                currency VARCHAR(20) DEFAULT '₽ RUB',
                PRIMARY KEY (user_id)
            )
        """)
        
        self.conn.commit()
        cur.close()
    
    def execute_query(self, query, params=None, fetch=False):
        if not self.conn:
            return None if not fetch else []
        
        cur = self.conn.cursor(cursor_factory=psycopg2.extras.DictCursor)
        try:
            cur.execute(query, params or ())
            if fetch:
                result = cur.fetchall()
                cur.close()
                return [dict(row) for row in result]
            else:
                self.conn.commit()
                cur.close()
                return cur.rowcount if cur.rowcount else None
        except Exception as e:
            self.conn.rollback()
            print(f"Database error: {e}")
            cur.close()
            return None if not fetch else []
    
    def close(self):
        if self.conn:
            self.conn.close()

# Глобальный экземпляр БД
db = Database()

# ============================================================================
# КРАСИВЫЕ СТИЛИ И ШРИФТЫ
# ============================================================================
FONTS = {
    "title": ("Segoe UI", 24, "bold"),
    "subtitle": ("Segoe UI", 16, "bold"),
    "heading": ("Segoe UI", 14, "bold"),
    "body": ("Segoe UI", 11),
    "body_bold": ("Segoe UI", 11, "bold"),
    "small": ("Segoe UI", 9),
    "small_bold": ("Segoe UI", 9, "bold"),
    "button": ("Segoe UI", 11, "bold"),
    "stats": ("Segoe UI", 20, "bold"),
}

COLORS = {
    "dark": {
        "bg_primary": "#0F172A",
        "bg_secondary": "#1E293B",
        "bg_tertiary": "#334155",
        "text_primary": "#F1F5F9",
        "text_secondary": "#94A3B8",
        "accent_primary": "#3B82F6",
        "accent_secondary": "#8B5CF6",
        "success": "#10B981",
        "warning": "#F59E0B",
        "error": "#EF4444",
        "card_bg": "#1E293B",
        "input_bg": "#0F172A",
        "border": "#334155",
    },
    "light": {
        "bg_primary": "#F8FAFC",
        "bg_secondary": "#FFFFFF",
        "bg_tertiary": "#E2E8F0",
        "text_primary": "#0F172A",
        "text_secondary": "#64748B",
        "accent_primary": "#3B82F6",
        "accent_secondary": "#8B5CF6",
        "success": "#10B981",
        "warning": "#F59E0B",
        "error": "#EF4444",
        "card_bg": "#FFFFFF",
        "input_bg": "#F8FAFC",
        "border": "#E2E8F0",
    }
}

current_theme = "dark"
current_language = "ru"
current_currency = "₽ RUB"
current_user_id = None

def get_color(key):
    return COLORS[current_theme][key]

# ============================================================================
# ПЕРЕВОДЫ
# ============================================================================
TRANSLATIONS = {
    "ru": {
        "app_title": f"{EMOJI['car']} FuelMaster",
        "calculator": f"{EMOJI['calculator']} Калькулятор",
        "profile": f"{EMOJI['profile']} Профиль",
        "history": f"{EMOJI['history']} История",
        "about": f"{EMOJI['info']} О приложении",
        "settings": f"{EMOJI['settings']} Настройки",
        "login": f"{EMOJI['car']} Войти",
        "register": f"{EMOJI['add']} Регистрация",
        "logout": f"{EMOJI['logout']} Выйти",
        "email": "Email",
        "password": "Пароль",
        "repeat_password": "Повторите пароль",
        "remember_me": "Запомнить меня",
        "forgot_password": f"{EMOJI['warning']} Забыли пароль?",
        "my_cars": f"{EMOJI['car']} Мои автомобили",
        "add_car": f"{EMOJI['add']} Добавить авто",
        "car_name": "Название автомобиля",
        "no_cars": "У вас пока нет автомобилей",
        "delete_car": f"{EMOJI['delete']} Удалить",
        "select_car": "Выберите автомобиль",
        "distance_label": f"{EMOJI['distance']} Расстояние",
        "fuel_used": f"{EMOJI['fuel']} Израсходовано топлива",
        "fuel_price": f"{EMOJI['money']} Цена за литр",
        "avg_consumption": f"{EMOJI['average']} Средний расход",
        "calculate": f"{EMOJI['calculator']} Рассчитать",
        "clear": f"{EMOJI['close']} Очистить",
        "result_consumption": "Расход топлива",
        "result_cost": "Стоимость поездки",
        "quick_stats": f"{EMOJI['stats']} Быстрая статистика",
        "history_title": f"{EMOJI['history']} История расчётов",
        "no_history": "История расчётов пуста",
        "clear_history": f"{EMOJI['delete']} Очистить историю",
        "settings_title": f"{EMOJI['settings']} Настройки",
        "theme": "Тема оформления",
        "dark_theme": "🌙 Тёмная",
        "light_theme": "☀️ Светлая",
        "language": "Язык",
        "currency": "Валюта",
        "save_settings": f"{EMOJI['save']} Сохранить",
        "about_text": f"{EMOJI['rocket']} FuelMaster v6.0\n\n"
                      f"{EMOJI['sparkles']} Профессиональный расчёт расхода топлива\n"
                      f"{EMOJI['car']} Хранение автомобилей с фотографиями\n"
                      f"{EMOJI['history']} Детальная история расчётов\n"
                      f"{EMOJI['chart']} Анализ расхода топлива\n\n"
                      f"© 2024 FuelMaster",
        "fuel_unit": "л",
        "km_unit": "км",
        "per_100km": "л/100 км",
        "enter_numbers": f"{EMOJI['warning']} Введите корректные числа!",
        "distance_positive": f"{EMOJI['warning']} Расстояние должно быть больше 0!",
        "success_calc": f"{EMOJI['success']} Расчёт выполнен!",
        "car_added": f"{EMOJI['success']} Автомобиль добавлен!",
        "car_deleted": f"{EMOJI['delete']} Автомобиль удалён",
        "history_cleared": f"{EMOJI['delete']} История очищена",
    },
    "en": {
        "app_title": f"{EMOJI['car']} FuelMaster",
        "calculator": f"{EMOJI['calculator']} Calculator",
        "profile": f"{EMOJI['profile']} Profile",
        "history": f"{EMOJI['history']} History",
        "about": f"{EMOJI['info']} About",
        "settings": f"{EMOJI['settings']} Settings",
        "login": f"{EMOJI['car']} Login",
        "register": f"{EMOJI['add']} Register",
        "logout": f"{EMOJI['logout']} Logout",
        "email": "Email",
        "password": "Password",
        "repeat_password": "Repeat password",
        "remember_me": "Remember me",
        "forgot_password": f"{EMOJI['warning']} Forgot password?",
        "my_cars": f"{EMOJI['car']} My Cars",
        "add_car": f"{EMOJI['add']} Add Car",
        "car_name": "Car Name",
        "no_cars": "No cars yet",
        "delete_car": f"{EMOJI['delete']} Delete",
        "select_car": "Select a car",
        "distance_label": f"{EMOJI['distance']} Distance",
        "fuel_used": f"{EMOJI['fuel']} Fuel used",
        "fuel_price": f"{EMOJI['money']} Price per liter",
        "avg_consumption": f"{EMOJI['average']} Avg consumption",
        "calculate": f"{EMOJI['calculator']} Calculate",
        "clear": f"{EMOJI['close']} Clear",
        "result_consumption": "Fuel consumption",
        "result_cost": "Trip cost",
        "quick_stats": f"{EMOJI['stats']} Quick Stats",
        "history_title": f"{EMOJI['history']} Calculation History",
        "no_history": "No calculations yet",
        "clear_history": f"{EMOJI['delete']} Clear history",
        "settings_title": f"{EMOJI['settings']} Settings",
        "theme": "Theme",
        "dark_theme": "🌙 Dark",
        "light_theme": "☀️ Light",
        "language": "Language",
        "currency": "Currency",
        "save_settings": f"{EMOJI['save']} Save",
        "about_text": f"{EMOJI['rocket']} FuelMaster v6.0\n\n"
                      f"{EMOJI['sparkles']} Professional fuel consumption calculator\n"
                      f"{EMOJI['car']} Store cars with photos\n"
                      f"{EMOJI['history']} Detailed calculation history\n"
                      f"{EMOJI['chart']} Fuel consumption analysis\n\n"
                      f"© 2024 FuelMaster",
        "fuel_unit": "L",
        "km_unit": "km",
        "per_100km": "L/100 km",
        "enter_numbers": f"{EMOJI['warning']} Enter valid numbers!",
        "distance_positive": f"{EMOJI['warning']} Distance must be greater than 0!",
        "success_calc": f"{EMOJI['success']} Calculation completed!",
        "car_added": f"{EMOJI['success']} Car added!",
        "car_deleted": f"{EMOJI['delete']} Car deleted",
        "history_cleared": f"{EMOJI['delete']} History cleared",
    }
}

def t(key):
    return TRANSLATIONS.get(current_language, TRANSLATIONS["ru"]).get(key, key)

# ============================================================================
# ОСНОВНОЕ ПРИЛОЖЕНИЕ С КРАСИВЫМ ДИЗАЙНОМ
# ============================================================================
class FuelApp(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title(t("app_title"))
        self.geometry("1300x850")
        self.minsize(1100, 700)
        
        # Загрузка данных из БД
        self.load_user_data()
        
        # Переменные
        self.user_id = current_user_id
        self.current_car_id = None
        self.photo_refs = {}
        
        # Построение UI
        self.setup_styles()
        self.create_widgets()
        self.load_cars()
        self.load_history()
    
    def load_user_data(self):
        global current_user_id, current_theme, current_language, current_currency
        
        result = db.execute_query("SELECT * FROM user_settings WHERE user_id = %s", 
                                  (current_user_id,), fetch=True) if current_user_id else []
        if result:
            current_theme = result[0].get('theme', 'dark')
            current_language = result[0].get('language', 'ru')
            current_currency = result[0].get('currency', '₽ RUB')
    
    def setup_styles(self):
        self.configure(bg=get_color("bg_primary"))
        
        # Настройка стилей для ttk
        style = ttk.Style()
        style.theme_use('clam')
        
        style.configure("Custom.TFrame", background=get_color("bg_primary"))
        style.configure("Card.TFrame", background=get_color("card_bg"), relief="flat")
        style.configure("Title.TLabel", background=get_color("bg_primary"), 
                       foreground=get_color("text_primary"), font=FONTS["title"])
        style.configure("Heading.TLabel", background=get_color("bg_primary"),
                       foreground=get_color("text_primary"), font=FONTS["heading"])
        style.configure("Body.TLabel", background=get_color("bg_primary"),
                       foreground=get_color("text_secondary"), font=FONTS["body"])
    
    def create_widgets(self):
        # Sidebar
        self.sidebar = tk.Frame(self, bg=get_color("bg_secondary"), width=280)
        self.sidebar.pack(side="left", fill="y")
        self.sidebar.pack_propagate(False)
        
        # Logo
        logo_frame = tk.Frame(self.sidebar, bg=get_color("bg_secondary"), pady=30)
        logo_frame.pack(fill="x")
        tk.Label(logo_frame, text=f"{EMOJI['car']} FuelMaster", 
                font=FONTS["title"], bg=get_color("bg_secondary"), 
                fg=get_color("accent_primary")).pack()
        tk.Label(logo_frame, text="Professional Fuel Calculator",
                font=FONTS["small"], bg=get_color("bg_secondary"),
                fg=get_color("text_secondary")).pack()
        
        # Navigation
        nav_frame = tk.Frame(self.sidebar, bg=get_color("bg_secondary"))
        nav_frame.pack(fill="x", pady=20)
        
        nav_items = [
            ("calculator", t("calculator")),
            ("profile", t("profile")),
            ("history", t("history")),
            ("settings", t("settings")),
            ("about", t("about"))
        ]
        
        self.nav_buttons = {}
        for i, (key, text) in enumerate(nav_items):
            btn = tk.Button(nav_frame, text=text, font=FONTS["body_bold"],
                          bg=get_color("bg_secondary"), fg=get_color("text_primary"),
                          bd=0, pady=12, anchor="w", padx=24,
                          cursor="hand2", activebackground=get_color("bg_tertiary"),
                          command=lambda k=key: self.show_section(k))
            btn.pack(fill="x", pady=2)
            self.nav_buttons[key] = btn
        
        # User info
        if current_user_id:
            self.user_info_frame = tk.Frame(self.sidebar, bg=get_color("bg_secondary"), pady=20)
            self.user_info_frame.pack(side="bottom", fill="x")
            
            user_data = db.execute_query("SELECT email FROM users WHERE id = %s", 
                                        (current_user_id,), fetch=True)
            if user_data:
                tk.Label(self.user_info_frame, text=f"{EMOJI['profile']} {user_data[0]['email']}",
                        font=FONTS["small"], bg=get_color("bg_secondary"),
                        fg=get_color("text_secondary")).pack()
            
            tk.Button(self.user_info_frame, text=t("logout"), font=FONTS["small"],
                     bg=get_color("error"), fg="white", bd=0, pady=8,
                     cursor="hand2", command=self.logout).pack(fill="x", padx=20, pady=5)
        
        # Main content
        self.content_frame = tk.Frame(self, bg=get_color("bg_primary"))
        self.content_frame.pack(side="left", fill="both", expand=True)
        
        # Создание секций
        self.sections = {}
        for section in ["calculator", "profile", "history", "settings", "about"]:
            frame = tk.Frame(self.content_frame, bg=get_color("bg_primary"))
            self.sections[section] = frame
        
        self.build_calculator()
        self.build_profile()
        self.build_history()
        self.build_settings()
        self.build_about()
        
        self.show_section("calculator")
    
    def build_calculator(self):
        frame = self.sections["calculator"]
        
        # Scrollable container
        canvas = tk.Canvas(frame, bg=get_color("bg_primary"), highlightthickness=0)
        scrollbar = ttk.Scrollbar(frame, orient="vertical", command=canvas.yview)
        scrollable_frame = tk.Frame(canvas, bg=get_color("bg_primary"))
        
        scrollable_frame.bind("<Configure>", lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
        canvas.create_window((0, 0), window=scrollable_frame, anchor="nw")
        canvas.configure(yscrollcommand=scrollbar.set)
        
        canvas.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")
        
        # Main container
        main_container = tk.Frame(scrollable_frame, bg=get_color("bg_primary"))
        main_container.pack(fill="both", expand=True, padx=40, pady=30)
        
        # Header
        tk.Label(main_container, text=t("calculator"), font=FONTS["title"],
                bg=get_color("bg_primary"), fg=get_color("accent_primary")).pack(anchor="w", pady=(0, 20))
        
        # Two columns
        columns = tk.Frame(main_container, bg=get_color("bg_primary"))
        columns.pack(fill="both", expand=True)
        
        # Left column - Car selection and stats
        left_col = tk.Frame(columns, bg=get_color("card_bg"), padx=20, pady=20)
        left_col.pack(side="left", fill="both", expand=True, padx=(0, 20))
        
        tk.Label(left_col, text=f"{EMOJI['car']} {t('select_car')}", font=FONTS["heading"],
                bg=get_color("card_bg"), fg=get_color("text_primary")).pack(anchor="w", pady=(0, 10))
        
        self.car_combo = ttk.Combobox(left_col, font=FONTS["body"], state="readonly", width=30)
        self.car_combo.pack(fill="x", pady=(0, 15))
        self.car_combo.bind("<<ComboboxSelected>>", self.on_car_selected)
        
        # Car photo
        self.photo_frame = tk.Frame(left_col, bg=get_color("bg_tertiary"), width=250, height=150)
        self.photo_frame.pack(pady=10)
        self.photo_frame.pack_propagate(False)
        self.photo_label = tk.Label(self.photo_frame, bg=get_color("bg_tertiary"), text=f"{EMOJI['car']}")
        self.photo_label.place(relx=0.5, rely=0.5, anchor="center")
        
        # Stats card
        stats_frame = tk.Frame(left_col, bg=get_color("bg_tertiary"), padx=15, pady=15)
        stats_frame.pack(fill="x", pady=(20, 0))
        
        tk.Label(stats_frame, text=t("quick_stats"), font=FONTS["heading"],
                bg=get_color("bg_tertiary"), fg=get_color("text_primary")).pack(anchor="w", pady=(0, 10))
        
        self.stats_labels = {}
        stats_items = [
            ("avg", f"{EMOJI['average']} {t('avg_consumption')}", "--"),
            ("dist", f"{EMOJI['distance']} {t('distance_label')}", "--"),
            ("fuel", f"{EMOJI['fuel']} {t('fuel_used')}", "--"),
            ("cost", f"{EMOJI['money']} {t('result_cost')}", "--")
        ]
        
        for key, label, default in stats_items:
            row = tk.Frame(stats_frame, bg=get_color("bg_tertiary"))
            row.pack(fill="x", pady=5)
            tk.Label(row, text=label, font=FONTS["small"],
                    bg=get_color("bg_tertiary"), fg=get_color("text_secondary")).pack(side="left")
            lbl = tk.Label(row, text=default, font=FONTS["body_bold"],
                          bg=get_color("bg_tertiary"), fg=get_color("accent_primary"))
            lbl.pack(side="right")
            self.stats_labels[key] = lbl
        
        # Right column - Calculator form
        right_col = tk.Frame(columns, bg=get_color("card_bg"), padx=20, pady=20)
        right_col.pack(side="left", fill="both", expand=True)
        
        # Mode selection
        mode_frame = tk.Frame(right_col, bg=get_color("card_bg"))
        mode_frame.pack(fill="x", pady=(0, 15))
        
        self.calc_mode = tk.StringVar(value="consumption")
        tk.Radiobutton(mode_frame, text="📊 Расход на 100 км", variable=self.calc_mode,
                      value="consumption", font=FONTS["body"], bg=get_color("card_bg"),
                      fg=get_color("text_primary"), selectcolor=get_color("card_bg"),
                      command=self.toggle_calc_mode).pack(side="left", padx=(0, 20))
        tk.Radiobutton(mode_frame, text="💰 Стоимость поездки", variable=self.calc_mode,
                      value="cost", font=FONTS["body"], bg=get_color("card_bg"),
                      fg=get_color("text_primary"), selectcolor=get_color("card_bg"),
                      command=self.toggle_calc_mode).pack(side="left")
        
        # Form fields
        self.form_frame = tk.Frame(right_col, bg=get_color("card_bg"))
        self.form_frame.pack(fill="x", pady=10)
        
        # Input fields with beautiful styling
        self.distance_var = tk.StringVar()
        self.fuel_var = tk.StringVar()
        self.price_var = tk.StringVar()
        self.avg_var = tk.StringVar()
        
        self.create_input_field(self.form_frame, f"{EMOJI['distance']} {t('distance_label')}", 
                                self.distance_var, f"{EMOJI['km_unit']}")
        self.create_input_field(self.form_frame, f"{EMOJI['fuel']} {t('fuel_used')}", 
                                self.fuel_var, f"{EMOJI['fuel_unit']}")
        self.create_input_field(self.form_frame, f"{EMOJI['money']} {t('fuel_price')}", 
                                self.price_var, f"{get_currency_symbol()}/{EMOJI['fuel_unit']}")
        
        # Buttons
        btn_frame = tk.Frame(right_col, bg=get_color("card_bg"))
        btn_frame.pack(fill="x", pady=20)
        
        calc_btn = tk.Button(btn_frame, text=t("calculate"), font=FONTS["button"],
                            bg=get_color("accent_primary"), fg="white", bd=0, pady=10,
                            cursor="hand2", command=self.calculate)
        calc_btn.pack(side="left", fill="x", expand=True, padx=(0, 10))
        
        clear_btn = tk.Button(btn_frame, text=t("clear"), font=FONTS["button"],
                             bg=get_color("bg_tertiary"), fg=get_color("text_primary"),
                             bd=0, pady=10, cursor="hand2", command=self.clear_form)
        clear_btn.pack(side="left", fill="x", expand=True)
        
        # Results
        self.result_frame = tk.Frame(right_col, bg=get_color("bg_tertiary"), padx=15, pady=15)
        self.result_frame.pack(fill="x", pady=10)
        
        self.result_consumption_label = tk.Label(self.result_frame, text="--",
                                                 font=FONTS["stats"], bg=get_color("bg_tertiary"),
                                                 fg=get_color("success"))
        self.result_cost_label = tk.Label(self.result_frame, text="--",
                                         font=FONTS["stats"], bg=get_color("bg_tertiary"),
                                         fg=get_color("accent_primary"))
    
    def create_input_field(self, parent, label, variable, unit):
        frame = tk.Frame(parent, bg=get_color("card_bg"))
        frame.pack(fill="x", pady=8)
        
        tk.Label(frame, text=label, font=FONTS["small"],
                bg=get_color("card_bg"), fg=get_color("text_secondary")).pack(anchor="w")
        
        entry_frame = tk.Frame(frame, bg=get_color("input_bg"), bd=1, relief="solid",
                               highlightcolor=get_color("accent_primary"))
        entry_frame.pack(fill="x", pady=2)
        
        entry = tk.Entry(entry_frame, textvariable=variable, font=FONTS["body"],
                        bg=get_color("input_bg"), fg=get_color("text_primary"),
                        bd=0, insertbackground=get_color("accent_primary"))
        entry.pack(side="left", fill="x", expand=True, padx=10, pady=8)
        
        tk.Label(entry_frame, text=unit, font=FONTS["small"],
                bg=get_color("input_bg"), fg=get_color("text_secondary"),
                padx=10).pack(side="right")
        
        return entry
    
    def toggle_calc_mode(self):
        if self.calc_mode.get() == "consumption":
            # Show fuel_used field, hide avg field
            pass
        else:
            # Show avg field, hide fuel_used field
            pass
    
    def on_car_selected(self, event):
        car_name = self.car_combo.get()
        if car_name and car_name != t("select_car"):
            cars = db.execute_query("SELECT * FROM cars WHERE user_id = %s AND name = %s",
                                   (current_user_id, car_name), fetch=True)
            if cars:
                self.current_car_id = cars[0]['id']
                avg = cars[0].get('avg_consumption')
                if avg:
                    self.avg_var.set(str(avg))
                
                # Load photo
                photo_path = cars[0].get('photo_path')
                if photo_path and os.path.exists(photo_path):
                    img = load_car_image(photo_path, 230, 130)
                    photo = ImageTk.PhotoImage(img)
                    self.photo_refs['current_car'] = photo
                    self.photo_label.configure(image=photo, text="")
                else:
                    self.photo_label.configure(image="", text=f"{EMOJI['car']}")
    
    def calculate(self):
        try:
            distance = float(self.distance_var.get().replace(",", "."))
            price = float(self.price_var.get().replace(",", "."))
            
            if distance <= 0:
                messagebox.showerror("Ошибка", t("distance_positive"))
                return
            
            if self.calc_mode.get() == "consumption":
                fuel = float(self.fuel_var.get().replace(",", "."))
                consumption = (fuel / distance) * 100
                cost = fuel * price
            else:
                avg = float(self.avg_var.get().replace(",", "."))
                if avg <= 0:
                    messagebox.showerror("Ошибка", "Введите средний расход!")
                    return
                consumption = avg
                fuel = (avg / 100) * distance
                cost = fuel * price
            
            # Update stats
            self.stats_labels["avg"].configure(text=f"{consumption:.1f} {t('per_100km')}")
            self.stats_labels["dist"].configure(text=f"{distance:.0f} {t('km_unit')}")
            self.stats_labels["fuel"].configure(text=f"{fuel:.1f} {t('fuel_unit')}")
            self.stats_labels["cost"].configure(text=f"{cost:.2f} {get_currency_symbol()}")
            
            # Show result
            if self.calc_mode.get() == "consumption":
                self.result_consumption_label.pack()
                self.result_cost_label.pack_forget()
                self.result_consumption_label.configure(text=f"{consumption:.1f} {t('per_100km')}")
            else:
                self.result_consumption_label.pack_forget()
                self.result_cost_label.pack()
                self.result_cost_label.configure(text=f"{cost:.2f} {get_currency_symbol()}")
            
            # Save to history
            if current_user_id:
                car_id = self.current_car_id if self.current_car_id else None
                car_name = self.car_combo.get() if self.car_combo.get() != t("select_car") else None
                
                db.execute_query("""
                    INSERT INTO history (user_id, car_id, car_name, distance, fuel_used, 
                                       consumption, price_per_liter, total_cost, currency)
                    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
                """, (current_user_id, car_id, car_name, distance, fuel, consumption, 
                      price, cost, get_currency_symbol()))
                
                # Update car average consumption
                if self.current_car_id and self.calc_mode.get() == "consumption":
                    db.execute_query("UPDATE cars SET avg_consumption = %s WHERE id = %s",
                                    (round(consumption, 2), self.current_car_id))
            
            self.load_history()
            messagebox.showinfo("Успех", t("success_calc"))
            
        except ValueError:
            messagebox.showerror("Ошибка", t("enter_numbers"))
    
    def clear_form(self):
        self.distance_var.set("")
        self.fuel_var.set("")
        self.price_var.set("")
        self.avg_var.set("")
        self.result_consumption_label.configure(text="--")
        self.result_cost_label.configure(text="--")
        for key in self.stats_labels:
            self.stats_labels[key].configure(text="--")
    
    def build_profile(self):
        frame = self.sections["profile"]
        
        scrollable = tk.Frame(frame, bg=get_color("bg_primary"))
        scrollable.pack(fill="both", expand=True, padx=40, pady=30)
        
        if not current_user_id:
            self.build_login_form(scrollable)
        else:
            self.build_user_profile(scrollable)
    
    def build_login_form(self, parent):
        # Centered card
        card = tk.Frame(parent, bg=get_color("card_bg"), padx=30, pady=30)
        card.pack(expand=True)
        
        tk.Label(card, text=f"{EMOJI['car']} {t('login')}", font=FONTS["title"],
                bg=get_color("card_bg"), fg=get_color("accent_primary")).pack(pady=(0, 20))
        
        # Email
        tk.Label(card, text=t("email"), font=FONTS["small"],
                bg=get_color("card_bg"), fg=get_color("text_secondary")).pack(anchor="w")
        self.login_email = tk.Entry(card, font=FONTS["body"], bg=get_color("input_bg"),
                                   fg=get_color("text_primary"), bd=1, relief="solid",
                                   width=30)
        self.login_email.pack(fill="x", pady=(5, 10), ipady=8)
        
        # Password
        tk.Label(card, text=t("password"), font=FONTS["small"],
                bg=get_color("card_bg"), fg=get_color("text_secondary")).pack(anchor="w")
        self.login_password = tk.Entry(card, font=FONTS["body"], bg=get_color("input_bg"),
                                      fg=get_color("text_primary"), bd=1, relief="solid",
                                      show="•", width=30)
        self.login_password.pack(fill="x", pady=(5, 10), ipady=8)
        
        # Buttons
        btn_frame = tk.Frame(card, bg=get_color("card_bg"))
        btn_frame.pack(fill="x", pady=10)
        
        tk.Button(btn_frame, text=t("login"), font=FONTS["button"],
                 bg=get_color("accent_primary"), fg="white", bd=0, pady=8,
                 cursor="hand2", command=self.do_login).pack(side="left", fill="x", expand=True, padx=(0, 5))
        
        tk.Button(btn_frame, text=t("register"), font=FONTS["button"],
                 bg=get_color("bg_tertiary"), fg=get_color("text_primary"), bd=0, pady=8,
                 cursor="hand2", command=self.show_register).pack(side="left", fill="x", expand=True, padx=(5, 0))
    
    def show_register(self):
        dialog = tk.Toplevel(self)
        dialog.title(t("register"))
        dialog.geometry("400x500")
        dialog.configure(bg=get_color("bg_primary"))
        dialog.transient(self)
        dialog.grab_set()
        
        card = tk.Frame(dialog, bg=get_color("card_bg"), padx=30, pady=30)
        card.pack(expand=True, fill="both")
        
        tk.Label(card, text=f"{EMOJI['add']} {t('register')}", font=FONTS["title"],
                bg=get_color("card_bg"), fg=get_color("accent_primary")).pack(pady=(0, 20))
        
        # Email
        tk.Label(card, text=t("email"), font=FONTS["small"],
                bg=get_color("card_bg"), fg=get_color("text_secondary")).pack(anchor="w")
        reg_email = tk.Entry(card, font=FONTS["body"], bg=get_color("input_bg"),
                            fg=get_color("text_primary"), bd=1, relief="solid")
        reg_email.pack(fill="x", pady=(5, 10), ipady=8)
        
        # Password
        tk.Label(card, text=t("password"), font=FONTS["small"],
                bg=get_color("card_bg"), fg=get_color("text_secondary")).pack(anchor="w")
        reg_password = tk.Entry(card, font=FONTS["body"], bg=get_color("input_bg"),
                               fg=get_color("text_primary"), bd=1, relief="solid", show="•")
        reg_password.pack(fill="x", pady=(5, 10), ipady=8)
        
        # Repeat Password
        tk.Label(card, text=t("repeat_password"), font=FONTS["small"],
                bg=get_color("card_bg"), fg=get_color("text_secondary")).pack(anchor="w")
        reg_password2 = tk.Entry(card, font=FONTS["body"], bg=get_color("input_bg"),
                                fg=get_color("text_primary"), bd=1, relief="solid", show="•")
        reg_password2.pack(fill="x", pady=(5, 10), ipady=8)
        
        def do_register():
            email = reg_email.get().strip().lower()
            password = reg_password.get()
            password2 = reg_password2.get()
            
            if not email or not password:
                messagebox.showerror("Ошибка", t("fill_all"))
                return
            
            if not re.match(r"^[^@\s]+@[^@\s]+\.[^@\s]+$", email):
                messagebox.showerror("Ошибка", t("invalid_email"))
                return
            
            if len(password) < 6:
                messagebox.showerror("Ошибка", t("password_short"))
                return
            
            if password != password2:
                messagebox.showerror("Ошибка", t("passwords_mismatch"))
                return
            
            existing = db.execute_query("SELECT id FROM users WHERE email = %s", (email,), fetch=True)
            if existing:
                messagebox.showerror("Ошибка", t("user_exists"))
                return
            
            password_hash = hashlib.sha256(password.encode()).hexdigest()
            db.execute_query("INSERT INTO users (email, password_hash, created_at) VALUES (%s, %s, NOW())",
                           (email, password_hash))
            
            messagebox.showinfo("Успех", f"{EMOJI['success']} Аккаунт создан!")
            dialog.destroy()
            self.rebuild_ui()
        
        tk.Button(card, text=t("register"), font=FONTS["button"],
                 bg=get_color("accent_primary"), fg="white", bd=0, pady=10,
                 cursor="hand2", command=do_register).pack(fill="x", pady=10)
    
    def do_login(self):
        global current_user_id
        email = self.login_email.get().strip().lower()
        password = self.login_password.get()
        
        if not email or not password:
            messagebox.showerror("Ошибка", t("fill_all"))
            return
        
        password_hash = hashlib.sha256(password.encode()).hexdigest()
        user = db.execute_query("SELECT id FROM users WHERE email = %s AND password_hash = %s",
                               (email, password_hash), fetch=True)
        
        if user:
            current_user_id = user[0]['id']
            db.execute_query("UPDATE users SET last_login = NOW() WHERE id = %s", (current_user_id,))
            self.rebuild_ui()
        else:
            messagebox.showerror("Ошибка", t("wrong_credentials"))
    
    def build_user_profile(self, parent):
        # User info header
        header = tk.Frame(parent, bg=get_color("card_bg"), padx=20, pady=15)
        header.pack(fill="x", pady=(0, 20))
        
        user_data = db.execute_query("SELECT email, created_at FROM users WHERE id = %s",
                                    (current_user_id,), fetch=True)
        if user_data:
            tk.Label(header, text=f"{EMOJI['profile']} {user_data[0]['email']}", font=FONTS["heading"],
                    bg=get_color("card_bg"), fg=get_color("text_primary")).pack(anchor="w")
            tk.Label(header, text=f"{EMOJI['calendar']} Создан: {user_data[0]['created_at'].strftime('%d.%m.%Y')}",
                    font=FONTS["small"], bg=get_color("card_bg"), fg=get_color("text_secondary")).pack(anchor="w")
        
        # Cars section
        cars_section = tk.Frame(parent, bg=get_color("card_bg"), padx=20, pady=15)
        cars_section.pack(fill="x", pady=(0, 20))
        
        cars_header = tk.Frame(cars_section, bg=get_color("card_bg"))
        cars_header.pack(fill="x", pady=(0, 10))
        
        tk.Label(cars_header, text=f"{EMOJI['car']} {t('my_cars')}", font=FONTS["heading"],
                bg=get_color("card_bg"), fg=get_color("text_primary")).pack(side="left")
        
        tk.Button(cars_header, text=t("add_car"), font=FONTS["small_bold"],
                 bg=get_color("accent_primary"), fg="white", bd=0, padx=15, pady=5,
                 cursor="hand2", command=self.add_car_dialog).pack(side="right")
        
        self.cars_container = tk.Frame(cars_section, bg=get_color("card_bg"))
        self.cars_container.pack(fill="x")
        
        self.refresh_cars_list()
    
    def refresh_cars_list(self):
        for widget in self.cars_container.winfo_children():
            widget.destroy()
        
        cars = db.execute_query("SELECT * FROM cars WHERE user_id = %s ORDER BY created_at DESC",
                               (current_user_id,), fetch=True)
        
        if not cars:
            tk.Label(self.cars_container, text=f"{EMOJI['car']} {t('no_cars')}", 
                    font=FONTS["body"], bg=get_color("card_bg"), fg=get_color("text_secondary")).pack(pady=20)
            return
        
        # Grid of cars (3 per row)
        row_frame = None
        for i, car in enumerate(cars):
            if i % 3 == 0:
                row_frame = tk.Frame(self.cars_container, bg=get_color("card_bg"))
                row_frame.pack(fill="x", pady=5)
            
            card = tk.Frame(row_frame, bg=get_color("bg_tertiary"), padx=10, pady=10, width=200, height=180)
            card.pack(side="left", padx=5, expand=True, fill="both")
            card.pack_propagate(False)
            
            # Photo
            photo_path = car.get('photo_path')
            if photo_path and os.path.exists(photo_path):
                img = load_car_image(photo_path, 180, 100)
                photo = ImageTk.PhotoImage(img)
                self.photo_refs[f"car_{car['id']}"] = photo
                tk.Label(card, image=photo, bg=get_color("bg_tertiary")).pack(pady=5)
            else:
                tk.Label(card, text=f"{EMOJI['car']}", font=("Segoe UI", 40),
                        bg=get_color("bg_tertiary")).pack(pady=5)
            
            tk.Label(card, text=car['name'], font=FONTS["body_bold"],
                    bg=get_color("bg_tertiary"), fg=get_color("text_primary")).pack()
            
            avg = car.get('avg_consumption')
            if avg:
                tk.Label(card, text=f"{EMOJI['average']} {avg} {t('per_100km')}", font=FONTS["small"],
                        bg=get_color("bg_tertiary"), fg=get_color("accent_primary")).pack()
            
            tk.Button(card, text=t("delete_car"), font=FONTS["small"],
                     bg=get_color("error"), fg="white", bd=0, padx=10, pady=3,
                     cursor="hand2", command=lambda cid=car['id']: self.delete_car(cid)).pack(pady=5)
    
    def add_car_dialog(self):
        dialog = tk.Toplevel(self)
        dialog.title(t("add_car"))
        dialog.geometry("450x350")
        dialog.configure(bg=get_color("bg_primary"))
        dialog.transient(self)
        dialog.grab_set()
        
        card = tk.Frame(dialog, bg=get_color("card_bg"), padx=20, pady=20)
        card.pack(expand=True, fill="both")
        
        tk.Label(card, text=f"{EMOJI['add']} {t('add_car')}", font=FONTS["title"],
                bg=get_color("card_bg"), fg=get_color("accent_primary")).pack(pady=(0, 20))
        
        tk.Label(card, text=t("car_name"), font=FONTS["small"],
                bg=get_color("card_bg"), fg=get_color("text_secondary")).pack(anchor="w")
        name_entry = tk.Entry(card, font=FONTS["body"], bg=get_color("input_bg"),
                             fg=get_color("text_primary"), bd=1, relief="solid")
        name_entry.pack(fill="x", pady=(5, 10), ipady=8)
        
        photo_path_var = tk.StringVar()
        
        def choose_photo():
            path = filedialog.askopenfilename(filetypes=[("Images", "*.jpg *.jpeg *.png *.bmp")])
            if path:
                ext = os.path.splitext(path)[1]
                dest = os.path.join(PHOTO_DIR, f"car_{datetime.now().strftime('%Y%m%d%H%M%S')}{ext}")
                shutil.copy2(path, dest)
                photo_path_var.set(dest)
        
        tk.Button(card, text=f"{EMOJI['edit']} {t('choose_photo')}", font=FONTS["button"],
                 bg=get_color("bg_tertiary"), fg=get_color("text_primary"), bd=0, pady=8,
                 command=choose_photo).pack(fill="x", pady=5)
        
        def save_car():
            name = name_entry.get().strip()
            if not name:
                messagebox.showerror("Ошибка", t("enter_name"))
                return
            
            db.execute_query("""
                INSERT INTO cars (user_id, name, photo_path, created_at)
                VALUES (%s, %s, %s, NOW())
            """, (current_user_id, name, photo_path_var.get() or None))
            
            messagebox.showinfo("Успех", t("car_added"))
            dialog.destroy()
            self.refresh_cars_list()
            self.load_cars()
        
        tk.Button(card, text=t("save"), font=FONTS["button"],
                 bg=get_color("accent_primary"), fg="white", bd=0, pady=10,
                 command=save_car).pack(fill="x", pady=10)
    
    def delete_car(self, car_id):
        if messagebox.askyesno("Подтверждение", t("delete_car_confirm")):
            db.execute_query("DELETE FROM cars WHERE id = %s AND user_id = %s", (car_id, current_user_id))
            self.refresh_cars_list()
            self.load_cars()
            messagebox.showinfo("Успех", t("car_deleted"))
    
    def load_cars(self):
        cars = db.execute_query("SELECT name FROM cars WHERE user_id = %s", (current_user_id,), fetch=True) if current_user_id else []
        car_list = [t("select_car")] + [car['name'] for car in cars]
        self.car_combo['values'] = car_list
        self.car_combo.set(t("select_car"))
    
    def build_history(self):
        frame = self.sections["history"]
        
        header = tk.Frame(frame, bg=get_color("bg_primary"))
        header.pack(fill="x", padx=40, pady=20)
        
        tk.Label(header, text=t("history_title"), font=FONTS["title"],
                bg=get_color("bg_primary"), fg=get_color("accent_primary")).pack(side="left")
        
        if current_user_id:
            tk.Button(header, text=t("clear_history"), font=FONTS["small_bold"],
                     bg=get_color("error"), fg="white", bd=0, padx=15, pady=5,
                     cursor="hand2", command=self.clear_history).pack(side="right")
        
        self.history_container = tk.Frame(frame, bg=get_color("bg_primary"))
        self.history_container.pack(fill="both", expand=True, padx=40, pady=(0, 20))
        
        self.load_history()
    
    def load_history(self):
        for widget in self.history_container.winfo_children():
            widget.destroy()
        
        if not current_user_id:
            tk.Label(self.history_container, text=f"{EMOJI['warning']} {t('login_for_history')}", 
                    font=FONTS["body"], bg=get_color("bg_primary"), fg=get_color("text_secondary")).pack(pady=50)
            return
        
        history = db.execute_query("""
            SELECT * FROM history WHERE user_id = %s ORDER BY created_at DESC LIMIT 50
        """, (current_user_id,), fetch=True)
        
        if not history:
            tk.Label(self.history_container, text=f"{EMOJI['history']} {t('no_history')}",
                    font=FONTS["body"], bg=get_color("bg_primary"), fg=get_color("text_secondary")).pack(pady=50)
            return
        
        # Scrollable
        canvas = tk.Canvas(self.history_container, bg=get_color("bg_primary"), highlightthickness=0)
        scrollbar = ttk.Scrollbar(self.history_container, orient="vertical", command=canvas.yview)
        scrollable_frame = tk.Frame(canvas, bg=get_color("bg_primary"))
        
        scrollable_frame.bind("<Configure>", lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
        canvas.create_window((0, 0), window=scrollable_frame, anchor="nw")
        canvas.configure(yscrollcommand=scrollbar.set)
        
        canvas.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")
        
        for entry in history:
            card = tk.Frame(scrollable_frame, bg=get_color("card_bg"), padx=15, pady=10)
            card.pack(fill="x", pady=5)
            
            # Header
            header = tk.Frame(card, bg=get_color("card_bg"))
            header.pack(fill="x")
            
            tk.Label(header, text=f"{EMOJI['calendar']} {entry['created_at'].strftime('%d.%m.%Y %H:%M')}",
                    font=FONTS["small"], bg=get_color("card_bg"), fg=get_color("text_secondary")).pack(side="left")
            
            # Stats grid
            stats = tk.Frame(card, bg=get_color("card_bg"))
            stats.pack(fill="x", pady=10)
            
            row1 = tk.Frame(stats, bg=get_color("card_bg"))
            row1.pack(fill="x")
            
            tk.Label(row1, text=f"{EMOJI['car']} {entry['car_name'] or 'Без авто'}", font=FONTS["body"],
                    bg=get_color("card_bg"), fg=get_color("text_primary")).pack(side="left", padx=(0, 20))
            tk.Label(row1, text=f"{EMOJI['distance']} {entry['distance']} {t('km_unit')}", font=FONTS["body"],
                    bg=get_color("card_bg"), fg=get_color("text_primary")).pack(side="left", padx=(0, 20))
            tk.Label(row1, text=f"{EMOJI['fuel']} {entry['fuel_used']} {t('fuel_unit')}", font=FONTS["body"],
                    bg=get_color("card_bg"), fg=get_color("text_primary")).pack(side="left")
            
            row2 = tk.Frame(stats, bg=get_color("card_bg"))
            row2.pack(fill="x", pady=(5, 0))
            
            tk.Label(row2, text=f"{EMOJI['average']} {entry['consumption']} {t('per_100km')}", font=FONTS["body_bold"],
                    bg=get_color("card_bg"), fg=get_color("success")).pack(side="left", padx=(0, 20))
            tk.Label(row2, text=f"{EMOJI['money']} {entry['total_cost']:.2f} {entry['currency']}", font=FONTS["body_bold"],
                    bg=get_color("card_bg"), fg=get_color("accent_primary")).pack(side="left")
    
    def clear_history(self):
        if messagebox.askyesno("Подтверждение", t("clear_history_msg")):
            db.execute_query("DELETE FROM history WHERE user_id = %s", (current_user_id,))
            self.load_history()
            messagebox.showinfo("Успех", t("history_cleared"))
    
    def build_settings(self):
        frame = self.sections["settings"]
        
        container = tk.Frame(frame, bg=get_color("bg_primary"))
        container.pack(expand=True, padx=40, pady=30)
        
        tk.Label(container, text=t("settings_title"), font=FONTS["title"],
                bg=get_color("bg_primary"), fg=get_color("accent_primary")).pack(anchor="w", pady=(0, 20))
        
        # Theme
        theme_card = tk.Frame(container, bg=get_color("card_bg"), padx=20, pady=15)
        theme_card.pack(fill="x", pady=10)
        
        tk.Label(theme_card, text=f"{EMOJI['settings']} {t('theme')}", font=FONTS["heading"],
                bg=get_color("card_bg"), fg=get_color("text_primary")).pack(anchor="w")
        
        theme_frame = tk.Frame(theme_card, bg=get_color("card_bg"))
        theme_frame.pack(fill="x", pady=10)
        
        self.theme_var = tk.StringVar(value=current_theme)
        tk.Radiobutton(theme_frame, text=t("dark_theme"), variable=self.theme_var,
                      value="dark", font=FONTS["body"], bg=get_color("card_bg"),
                      fg=get_color("text_primary"), selectcolor=get_color("card_bg")).pack(side="left", padx=(0, 20))
        tk.Radiobutton(theme_frame, text=t("light_theme"), variable=self.theme_var,
                      value="light", font=FONTS["body"], bg=get_color("card_bg"),
                      fg=get_color("text_primary"), selectcolor=get_color("card_bg")).pack(side="left")
        
        # Language
        lang_card = tk.Frame(container, bg=get_color("card_bg"), padx=20, pady=15)
        lang_card.pack(fill="x", pady=10)
        
        tk.Label(lang_card, text=f"{EMOJI['settings']} {t('language')}", font=FONTS["heading"],
                bg=get_color("card_bg"), fg=get_color("text_primary")).pack(anchor="w")
        
        lang_frame = tk.Frame(lang_card, bg=get_color("card_bg"))
        lang_frame.pack(fill="x", pady=10)
        
        self.lang_var = tk.StringVar(value=current_language)
        tk.Radiobutton(lang_frame, text="🇷🇺 Русский", variable=self.lang_var,
                      value="ru", font=FONTS["body"], bg=get_color("card_bg"),
                      fg=get_color("text_primary"), selectcolor=get_color("card_bg")).pack(side="left", padx=(0, 20))
        tk.Radiobutton(lang_frame, text="🇬🇧 English", variable=self.lang_var,
                      value="en", font=FONTS["body"], bg=get_color("card_bg"),
                      fg=get_color("text_primary"), selectcolor=get_color("card_bg")).pack(side="left")
        
        # Currency
        curr_card = tk.Frame(container, bg=get_color("card_bg"), padx=20, pady=15)
        curr_card.pack(fill="x", pady=10)
        
        tk.Label(curr_card, text=f"{EMOJI['money']} {t('currency')}", font=FONTS["heading"],
                bg=get_color("card_bg"), fg=get_color("text_primary")).pack(anchor="w")
        
        self.curr_var = tk.StringVar(value=current_currency)
        curr_combo = ttk.Combobox(curr_card, textvariable=self.curr_var,
                                 values=["₽ RUB", "$ USD"], font=FONTS["body"], state="readonly")
        curr_combo.pack(anchor="w", pady=10)
        
        # Save button
        tk.Button(container, text=t("save_settings"), font=FONTS["button"],
                 bg=get_color("accent_primary"), fg="white", bd=0, pady=10,
                 cursor="hand2", command=self.save_settings).pack(fill="x", pady=20)
    
    def save_settings(self):
        global current_theme, current_language, current_currency
        current_theme = self.theme_var.get()
        current_language = self.lang_var.get()
        current_currency = self.curr_var.get()
        
        if current_user_id:
            db.execute_query("""
                INSERT INTO user_settings (user_id, theme, language, currency)
                VALUES (%s, %s, %s, %s)
                ON CONFLICT (user_id) DO UPDATE SET
                theme = EXCLUDED.theme, language = EXCLUDED.language, currency = EXCLUDED.currency
            """, (current_user_id, current_theme, current_language, current_currency))
        
        self.rebuild_ui()
        messagebox.showinfo("Успех", "Настройки сохранены!")
    
    def build_about(self):
        frame = self.sections["about"]
        
        container = tk.Frame(frame, bg=get_color("bg_primary"))
        container.pack(expand=True, padx=40, pady=30)
        
        card = tk.Frame(container, bg=get_color("card_bg"), padx=30, pady=30)
        card.pack(expand=True)
        
        tk.Label(card, text=f"{EMOJI['rocket']} FuelMaster", font=FONTS["title"],
                bg=get_color("card_bg"), fg=get_color("accent_primary")).pack()
        
        tk.Label(card, text=t("about_text"), font=FONTS["body"],
                bg=get_color("card_bg"), fg=get_color("text_secondary"),
                justify="center").pack(pady=20)
    
    def show_section(self, section):
        for s in self.sections.values():
            s.pack_forget()
        self.sections[section].pack(fill="both", expand=True)
        
        if section == "history":
            self.load_history()
        elif section == "profile" and current_user_id:
            self.refresh_cars_list()
    
    def logout(self):
        global current_user_id
        current_user_id = None
        self.rebuild_ui()
    
    def rebuild_ui(self):
        for widget in self.winfo_children():
            widget.destroy()
        self.create_widgets()
    
    def on_closing(self):
        db.close()
        self.destroy()

def load_car_image(path, width, height):
    try:
        img = Image.open(path)
        img = img.resize((width, height), Image.Resampling.LANCZOS)
        return img
    except:
        img = Image.new("RGB", (width, height), get_color("bg_tertiary"))
        draw = ImageDraw.Draw(img)
        draw.text((width//2-20, height//2-10), "🚗", fill=get_color("text_secondary"))
        return img

def get_currency_symbol():
    return "₽" if "RUB" in current_currency else "$"

if __name__ == "__main__":
    app = FuelApp()
    app.protocol("WM_DELETE_WINDOW", app.on_closing)
    app.mainloop()
