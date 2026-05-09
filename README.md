import tkinter as tk
from tkinter import ttk, messagebox, filedialog
import json
import os
import re
import hashlib
import shutil
from datetime import datetime, timedelta
import math
import threading
from queue import Queue

try:
    from PIL import Image, ImageTk, ImageDraw, ImageFont
except ImportError:
    import subprocess, sys

    subprocess.check_call([sys.executable, "-m", "pip", "install", "Pillow"])
    from PIL import Image, ImageTk, ImageDraw, ImageFont

# PostgreSQL imports
try:
    import psycopg2
    from psycopg2 import sql
    from psycopg2 import pool

    PSYCOPG2_AVAILABLE = True
except ImportError:
    print("Устанавливаем psycopg2-binary...")
    import subprocess, sys

    subprocess.check_call([sys.executable, "-m", "pip", "install", "psycopg2-binary"])
    import psycopg2
    from psycopg2 import sql
    from psycopg2 import pool

    PSYCOPG2_AVAILABLE = True

# ─── Database configuration ────────────────────────────────────────────────────
DB_CONFIG = {
    'dbname': 'fuel_calc',
    'user': 'postgres',
    'password': '12345',  # ваш пароль
    'host': 'localhost',
    'port': 5432
}

# Connection pool
connection_pool = None


def init_db_pool():
    global connection_pool
    try:
        connection_pool = pool.SimpleConnectionPool(
            1, 10,
            dbname=DB_CONFIG['dbname'],
            user=DB_CONFIG['user'],
            password=DB_CONFIG['password'],
            host=DB_CONFIG['host'],
            port=DB_CONFIG['port']
        )
        return True
    except Exception as e:
        print(f"Database connection error: {e}")
        return False


def get_db_connection():
    return connection_pool.getconn()


def put_db_connection(conn):
    connection_pool.putconn(conn)


# ─── Database initialization ───────────────────────────────────────────────────
def create_tables():
    """Create all necessary tables if they don't exist"""
    conn = None
    try:
        conn = psycopg2.connect(**DB_CONFIG)
        cursor = conn.cursor()

        # Users table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS users (
                email TEXT PRIMARY KEY,
                password TEXT NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)

        # Cars table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS cars (
                id SERIAL PRIMARY KEY,
                email TEXT REFERENCES users(email) ON DELETE CASCADE,
                name TEXT NOT NULL,
                photo_path TEXT,
                avg_consumption FLOAT,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                UNIQUE(email, name)
            )
        """)

        # History table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS history (
                id SERIAL PRIMARY KEY,
                email TEXT REFERENCES users(email) ON DELETE CASCADE,
                car_name TEXT,
                date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                distance FLOAT NOT NULL,
                fuel FLOAT NOT NULL,
                price FLOAT NOT NULL,
                currency TEXT,
                consumption FLOAT NOT NULL,
                cost FLOAT NOT NULL
            )
        """)

        # Settings table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS settings (
                email TEXT PRIMARY KEY REFERENCES users(email) ON DELETE CASCADE,
                theme TEXT DEFAULT 'dark',
                language TEXT DEFAULT 'ru',
                currency TEXT DEFAULT '₽ RUB'
            )
        """)

        conn.commit()
        cursor.close()
        conn.close()
        return True
    except Exception as e:
        print(f"Error creating tables: {e}")
        return False


# ─── Database operations ───────────────────────────────────────────────────────
def db_create_user(email, password_hash):
    conn = get_db_connection()
    try:
        cursor = conn.cursor()
        cursor.execute(
            "INSERT INTO users (email, password) VALUES (%s, %s)",
            (email, password_hash)
        )
        cursor.execute(
            "INSERT INTO settings (email, theme, language, currency) VALUES (%s, %s, %s, %s)",
            (email, 'dark', 'ru', '₽ RUB')
        )
        conn.commit()
    finally:
        put_db_connection(conn)


def db_get_user(email):
    conn = get_db_connection()
    try:
        cursor = conn.cursor()
        cursor.execute("SELECT password FROM users WHERE email = %s", (email,))
        result = cursor.fetchone()
        return result[0] if result else None
    finally:
        put_db_connection(conn)


def db_user_exists(email):
    conn = get_db_connection()
    try:
        cursor = conn.cursor()
        cursor.execute("SELECT 1 FROM users WHERE email = %s", (email,))
        return cursor.fetchone() is not None
    finally:
        put_db_connection(conn)


def db_update_password(email, new_password_hash):
    conn = get_db_connection()
    try:
        cursor = conn.cursor()
        cursor.execute(
            "UPDATE users SET password = %s WHERE email = %s",
            (new_password_hash, email)
        )
        conn.commit()
    finally:
        put_db_connection(conn)


def db_delete_user(email):
    conn = get_db_connection()
    try:
        cursor = conn.cursor()
        # Delete photos from filesystem
        cursor.execute("SELECT photo_path FROM cars WHERE email = %s", (email,))
        for row in cursor.fetchall():
            if row[0] and os.path.exists(row[0]):
                try:
                    os.remove(row[0])
                except:
                    pass
        cursor.execute("DELETE FROM users WHERE email = %s", (email,))
        conn.commit()
    finally:
        put_db_connection(conn)


def db_get_cars(email):
    conn = get_db_connection()
    try:
        cursor = conn.cursor()
        cursor.execute(
            "SELECT name, photo_path, avg_consumption FROM cars WHERE email = %s ORDER BY id",
            (email,)
        )
        return [{"name": r[0], "photo": r[1], "avg_consumption": r[2]} for r in cursor.fetchall()]
    finally:
        put_db_connection(conn)


def db_add_car(email, name, photo_path=None):
    conn = get_db_connection()
    try:
        cursor = conn.cursor()
        cursor.execute(
            "INSERT INTO cars (email, name, photo_path) VALUES (%s, %s, %s)",
            (email, name, photo_path)
        )
        conn.commit()
    finally:
        put_db_connection(conn)


def db_delete_car(email, name):
    conn = get_db_connection()
    try:
        cursor = conn.cursor()
        cursor.execute("SELECT photo_path FROM cars WHERE email = %s AND name = %s", (email, name))
        row = cursor.fetchone()
        if row and row[0] and os.path.exists(row[0]):
            try:
                os.remove(row[0])
            except:
                pass
        cursor.execute("DELETE FROM cars WHERE email = %s AND name = %s", (email, name))
        conn.commit()
    finally:
        put_db_connection(conn)


def db_update_car_consumption(email, car_name, consumption):
    conn = get_db_connection()
    try:
        cursor = conn.cursor()
        cursor.execute(
            "UPDATE cars SET avg_consumption = %s WHERE email = %s AND name = %s",
            (consumption, email, car_name)
        )
        conn.commit()
    finally:
        put_db_connection(conn)


def db_add_history(email, entry):
    conn = get_db_connection()
    try:
        cursor = conn.cursor()
        cursor.execute("""
            INSERT INTO history (email, car_name, distance, fuel, price, currency, consumption, cost, date)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
        """, (
            email, entry.get('car'), entry['distance'], entry['fuel'],
            entry['price'], entry.get('currency'), entry['consumption'],
            entry['cost'], datetime.strptime(entry['date'], "%d.%m.%Y %H:%M")
        ))
        conn.commit()
    finally:
        put_db_connection(conn)


def db_get_history(email, car_name=None, days=None):
    conn = get_db_connection()
    try:
        cursor = conn.cursor()
        query = """
            SELECT car_name, distance, fuel, price, currency, consumption, cost, 
                   TO_CHAR(date, 'DD.MM.YYYY HH24:MI') as date_str,
                   date as raw_date
            FROM history 
            WHERE email = %s
        """
        params = [email]

        if car_name and car_name != "— Без авто —" and car_name != "— No car —":
            query += " AND car_name = %s"
            params.append(car_name)

        if days:
            query += " AND date >= CURRENT_DATE - INTERVAL '%s days'"
            params.append(days)

        query += " ORDER BY date DESC"

        cursor.execute(query, params)
        results = []
        for r in cursor.fetchall():
            results.append({
                'car': r[0], 'distance': r[1], 'fuel': r[2], 'price': r[3],
                'currency': r[4], 'consumption': r[5], 'cost': r[6],
                'date': r[7], 'raw_date': r[8]
            })
        return results
    finally:
        put_db_connection(conn)


def db_clear_history(email):
    conn = get_db_connection()
    try:
        cursor = conn.cursor()
        cursor.execute("DELETE FROM history WHERE email = %s", (email,))
        conn.commit()
    finally:
        put_db_connection(conn)


def db_get_settings(email):
    conn = get_db_connection()
    try:
        cursor = conn.cursor()
        cursor.execute(
            "SELECT theme, language, currency FROM settings WHERE email = %s",
            (email,)
        )
        result = cursor.fetchone()
        if result:
            return {'theme': result[0], 'language': result[1], 'currency': result[2]}
        return {'theme': 'dark', 'language': 'ru', 'currency': '₽ RUB'}
    finally:
        put_db_connection(conn)


def db_update_settings(email, theme, language, currency):
    conn = get_db_connection()
    try:
        cursor = conn.cursor()
        cursor.execute("""
            UPDATE settings 
            SET theme = %s, language = %s, currency = %s 
            WHERE email = %s
        """, (theme, language, currency, email))
        conn.commit()
    finally:
        put_db_connection(conn)


def db_count_cars(email):
    conn = get_db_connection()
    try:
        cursor = conn.cursor()
        cursor.execute("SELECT COUNT(*) FROM cars WHERE email = %s", (email,))
        return cursor.fetchone()[0]
    finally:
        put_db_connection(conn)


def db_get_history_for_chart(email, car_name, days):
    conn = get_db_connection()
    try:
        cursor = conn.cursor()
        query = """
            SELECT DATE(date) as day, AVG(consumption) as avg_consumption
            FROM history 
            WHERE email = %s 
              AND date >= CURRENT_DATE - INTERVAL '%s days'
        """
        params = [email, days]

        if car_name and car_name != "— Без авто —" and car_name != "— No car —":
            query += " AND car_name = %s"
            params.append(car_name)

        query += " GROUP BY DATE(date) ORDER BY day"

        cursor.execute(query, params)
        return [(r[0], r[1]) for r in cursor.fetchall()]
    finally:
        put_db_connection(conn)


# ─── Data storage (fallback if PostgreSQL fails) ──────────────────────────────
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


# ─── Translations ──────────────────────────────────────────────────────────────
TRANSLATIONS = {
    "ru": {
        "app_title": "Калькулятор расхода топлива",
        "app_subtitle": "Расчёт расхода топлива",
        "calculator": "Калькулятор",
        "profile": "Профиль",
        "history": "История",
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
        "password_short": "Пароль (мин. 6 символов)",
        "passwords_mismatch": "Пароли не совпадают",
        "user_exists": "Пользователь уже существует",
        "wrong_credentials": "Неверный email или пароль",
        "forgot_password": "Забыли пароль?",
        "forgot_password_title": "Восстановление пароля",
        "new_password": "Новый пароль",
        "new_password_set": "Новый пароль установлен",
        "password_changed": "Пароль изменён",
        "logout": "Выйти",
        "delete_account": "Удалить аккаунт",
        "change_password": "Сменить пароль",
        "repeat_new_password": "Повторите новый пароль",
        "change_password_btn": "Сменить пароль",
        "my_cars": "Мои автомобили (макс. 25)",
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
        "copied_msg": "Расчёт скопирован!",
        "delete_entry": "Удалить",
        "delete_confirm": "Удалить?",
        "delete_car_confirm": "Удалить этот автомобиль?",
        "clear_history_confirm": "Очистить?",
        "clear_history_msg": "Удалить всю историю?",
        "delete_account_confirm": "Удалить аккаунт?",
        "delete_account_msg": "Вы уверены? Все данные будут удалены.",
        "settings_title": "Настройки",
        "theme": "Тема оформления",
        "theme_desc": "Выберите цветовую схему",
        "dark_theme": "Тёмная",
        "light_theme": "Светлая",
        "current_theme": "Текущая тема:",
        "currency_setting": "Валюта",
        "currency_desc": "Выберите валюту",
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
        "fuel_unit": "л",
        "price_unit": "за л",
        "about_title": "О приложении",
        "version": "Версия 5.0  •  2024",
        "about_section1_title": "О приложении",
        "about_section1": "Профессиональное приложение для расчёта расхода топлива.",
        "about_section2_title": "Возможности",
        "about_section2": "• Точный расчёт расхода\n• Расчёт стоимости поездки\n• Хранение до 25 автомобилей\n• История расчётов\n• PostgreSQL база данных",
        "about_section3_title": "Как использовать",
        "about_section3": "1. Зарегистрируйтесь\n2. Добавьте автомобиль\n3. Выберите режим расчёта\n4. Введите данные\n5. Нажмите «Рассчитать»",
        "about_section4_title": "Безопасность",
        "about_section4": "Пароли зашифрованы (SHA-256). Данные хранятся в PostgreSQL.",
        "about_section5_title": "Интерпретация",
        "about_section5": "До 8 л/100 км — экономично\n8-12 л/100 км — средне\nБолее 12 л/100 км — высокий расход",
        "footer": "© 2024 Калькулятор расхода топлива",
        "no_cars_limit": "Максимум 25 автомобилей",
        "database_error": "Ошибка подключения к базе данных",
        "database_retry": "Попробуйте позже или проверьте настройки PostgreSQL",
    },
    "en": {
        "app_title": "Fuel Calculator",
        "app_subtitle": "Fuel consumption calculation",
        "calculator": "Calculator",
        "profile": "Profile",
        "history": "History",
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
        "password_short": "Password (min. 6 chars)",
        "passwords_mismatch": "Passwords do not match",
        "user_exists": "User already exists",
        "wrong_credentials": "Invalid email or password",
        "forgot_password": "Forgot password?",
        "forgot_password_title": "Password recovery",
        "new_password": "New password",
        "new_password_set": "New password set",
        "password_changed": "Password changed",
        "logout": "Logout",
        "delete_account": "Delete account",
        "change_password": "Change password",
        "repeat_new_password": "Repeat new password",
        "change_password_btn": "Change password",
        "my_cars": "My vehicles (max. 25)",
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
        "copied_msg": "Calculation copied!",
        "delete_entry": "Delete",
        "delete_confirm": "Delete?",
        "delete_car_confirm": "Delete this vehicle?",
        "clear_history_confirm": "Clear?",
        "clear_history_msg": "Delete all history?",
        "delete_account_confirm": "Delete account?",
        "delete_account_msg": "Are you sure? All data will be deleted.",
        "settings_title": "Settings",
        "theme": "Theme",
        "theme_desc": "Choose color scheme",
        "dark_theme": "Dark",
        "light_theme": "Light",
        "current_theme": "Current theme:",
        "currency_setting": "Currency",
        "currency_desc": "Choose currency",
        "language_setting": "Language / Язык",
        "language_desc": "Choose language",
        "current": "Current:",
        "error": "Error",
        "enter_numbers": "Enter valid numbers!",
        "distance_positive": "Distance must be greater than 0!",
        "distance_positive2": "Distance and Avg must be greater than 0!",
        "fuel_chart_title": "Fuel consumption chart",
        "chart_days": "Days:",
        "no_history_chart": "No data for chart",
        "login_for_chart": "Log in to view chart",
        "fuel_unit": "L",
        "price_unit": "per L",
        "about_title": "About",
        "version": "Version 5.0  •  2024",
        "about_section1_title": "About",
        "about_section1": "Professional fuel calculator with PostgreSQL database.",
        "about_section2_title": "Features",
        "about_section2": "• Accurate consumption calculation\n• Trip cost calculation\n• Up to 25 vehicles\n• Calculation history\n• PostgreSQL database storage",
        "about_section3_title": "How to use",
        "about_section3": "1. Register\n2. Add a vehicle\n3. Select calculation mode\n4. Enter data\n5. Press Calculate",
        "about_section4_title": "Security",
        "about_section4": "Passwords encrypted (SHA-256). Data stored in PostgreSQL.",
        "about_section5_title": "Interpretation",
        "about_section5": "Up to 8 L/100 km — economical\n8-12 L/100 km — average\nOver 12 L/100 km — high",
        "footer": "© 2024 Fuel Consumption Calculator",
        "no_cars_limit": "Maximum 25 vehicles",
        "database_error": "Database connection error",
        "database_retry": "Please try again later or check PostgreSQL settings",
    }
}

# ─── Currencies ────────────────────────────────────────────────────────────────
CURRENCIES = {
    "₽ RUB": "₽",
    "$ USD": "$",
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
        "chart_bar1": "#3fb950", "chart_bar2": "#d29922", "chart_bar3": "#f85149",
    },
    "light": {
        "bg": "#f0f4f8", "bg2": "#ffffff", "bg3": "#e2e8f0",
        "fg": "#1a202c", "fg2": "#718096", "accent": "#3182ce",
        "green": "#38a169", "yellow": "#d69e2e", "red": "#e53e3e",
        "border": "#cbd5e0", "btn": "#3182ce", "btn_hover": "#2b6cb0",
        "btn2": "#e2e8f0", "btn2_hover": "#cbd5e0", "input_bg": "#ffffff",
        "chart_bg": "#ffffff", "chart_line": "#3182ce",
        "chart_fill": "#ebf4ff", "chart_grid": "#e2e8f0",
        "chart_bar1": "#38a169", "chart_bar2": "#d69e2e", "chart_bar3": "#e53e3e",
    }
}

current_theme = "dark"
current_language = "ru"
current_currency = "₽ RUB"


def T():
    return THEMES[current_theme]


def TR(key):
    return TRANSLATIONS.get(current_language, TRANSLATIONS["ru"]).get(key, key)


def get_currency_symbol():
    return CURRENCIES.get(current_currency, "₽")


def hash_pw(pw):
    return hashlib.sha256(pw.encode()).hexdigest()


def is_valid_email(email):
    return re.match(r"^[^@\s]+@[^@\s]+\.[^@\s]+$", email) is not None


# ─── Scroll helper ─────────────────────────────────────────────────────────────
def make_scrollable(parent, bg=None):
    if bg is None:
        bg = T()["bg"]
    outer = tk.Frame(parent, bg=bg)
    canvas = tk.Canvas(outer, bg=bg, highlightthickness=0)
    scroll = ttk.Scrollbar(outer, orient="vertical", command=canvas.yview)
    canvas.configure(yscrollcommand=scroll.set)
    scroll.pack(side="right", fill="y")
    canvas.pack(side="left", fill="both", expand=True)

    inner = tk.Frame(canvas, bg=bg)
    win = canvas.create_window((0, 0), window=inner, anchor="nw")

    def configure_inner(event):
        canvas.configure(scrollregion=canvas.bbox("all"))

    inner.bind("<Configure>", configure_inner)

    def configure_canvas(event):
        canvas.itemconfig(win, width=event.width)

    canvas.bind("<Configure>", configure_canvas)

    def _on_mousewheel(event):
        if event.num == 4:
            canvas.yview_scroll(-1, "units")
        elif event.num == 5:
            canvas.yview_scroll(1, "units")
        else:
            canvas.yview_scroll(int(-1 * (event.delta / 120)), "units")

    def _bind_mw(e):
        canvas.bind_all("<MouseWheel>", _on_mousewheel)
        canvas.bind_all("<Button-4>", _on_mousewheel)
        canvas.bind_all("<Button-5>", _on_mousewheel)

    def _unbind_mw(e):
        canvas.unbind_all("<MouseWheel>")
        canvas.unbind_all("<Button-4>")
        canvas.unbind_all("<Button-5>")

    def bind_recursive(widget):
        widget.bind("<Enter>", _bind_mw)
        widget.bind("<Leave>", _unbind_mw)
        for child in widget.winfo_children():
            bind_recursive(child)

    bind_recursive(inner)
    canvas.bind("<Enter>", _bind_mw)
    canvas.bind("<Leave>", _unbind_mw)

    return outer, inner


# ─── Image helpers ─────────────────────────────────────────────────────────────
def make_placeholder(w=300, h=180, text=""):
    t = T()
    try:
        img = Image.new("RGB", (w, h), t["bg3"])
        draw = ImageDraw.Draw(img)
        draw.rectangle([0, 0, w - 1, h - 1], outline=t["border"], width=2)
        if text:
            draw.text((w // 2 - 30, h // 2 - 8), text, fill=t["fg2"])
        return img
    except Exception:
        return Image.new("RGB", (w, h), "#333333")


def load_car_image(path, w=300, h=180):
    if not path or not isinstance(path, str) or not os.path.exists(path):
        return make_placeholder(w, h)
    try:
        file_size = os.path.getsize(path)
        if file_size < 100:
            return make_placeholder(w, h)
    except OSError:
        return make_placeholder(w, h)

    try:
        img = Image.open(path)
        img.verify()
        img = Image.open(path)
        if img.mode not in ('RGB', 'RGBA'):
            img = img.convert('RGB')
        elif img.mode == 'RGBA':
            bg = Image.new('RGB', img.size, (0, 0, 0))
            bg.paste(img, mask=img.split()[3])
            img = bg

        src_w, src_h = img.size
        if src_w <= 0 or src_h <= 0:
            return make_placeholder(w, h)
        src_r = src_w / src_h
        dst_r = w / h
        if src_r > dst_r:
            new_h = h
            new_w = int(src_w * h / src_h)
        else:
            new_w = w
            new_h = int(src_h * w / src_w)
        new_w = max(1, min(new_w, 5000))
        new_h = max(1, min(new_h, 5000))
        img = img.resize((new_w, new_h), Image.LANCZOS)
        left = (new_w - w) // 2
        top = (new_h - h) // 2
        img = img.crop((left, top, left + w, top + h))
        return img
    except Exception:
        return make_placeholder(w, h)


# ─── Main Application ──────────────────────────────────────────────────────────
class FuelApp(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title(TR("app_title"))
        self.geometry("1200x850")
        self.minsize(1000, 700)
        self.resizable(True, True)

        self.current_user = None
        self._photo_refs = {}
        self._calc_mode = tk.StringVar(value="consumption")
        self._db_connected = False

        # Initialize database
        self._init_database()

        if self._db_connected:
            self._load_user_settings()

        self._build_ui()

    def _init_database(self):
        """Initialize database connection and tables"""
        global connection_pool
        try:
            # First, try to create database if it doesn't exist
            conn = psycopg2.connect(
                dbname='postgres',
                user=DB_CONFIG['user'],
                password=DB_CONFIG['password'],
                host=DB_CONFIG['host'],
                port=DB_CONFIG['port']
            )
            conn.autocommit = True
            cursor = conn.cursor()
            cursor.execute("SELECT 1 FROM pg_database WHERE datname = 'fuel_calc'")
            if not cursor.fetchone():
                cursor.execute("CREATE DATABASE fuel_calc")
            cursor.close()
            conn.close()

            # Now connect to the fuel_calc database
            if init_db_pool():
                if create_tables():
                    self._db_connected = True
                    print("Database initialized successfully")
                else:
                    print("Failed to create tables")
                    self._db_connected = False
            else:
                print("Failed to initialize connection pool")
                self._db_connected = False
        except Exception as e:
            print(f"Database initialization error: {e}")
            print("Using local file storage instead...")
            self._db_connected = False

    def _load_user_settings(self):
        """Load settings for the current user if logged in"""
        if self.current_user and self._db_connected:
            try:
                settings = db_get_settings(self.current_user)
                global current_theme, current_language, current_currency
                current_theme = settings.get('theme', 'dark')
                current_language = settings.get('language', 'ru')
                current_currency = settings.get('currency', '₽ RUB')
            except Exception as e:
                print(f"Error loading settings: {e}")

    def _save_settings(self):
        """Save settings to database"""
        if self.current_user and self._db_connected:
            try:
                db_update_settings(self.current_user, current_theme, current_language, current_currency)
            except Exception as e:
                print(f"Error saving settings: {e}")

    def _rebuild_ui(self):
        self.header.destroy()
        self.body.destroy()
        self._photo_refs.clear()
        self._build_ui()

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
        for name in ["calculator", "profile", "history", "about", "settings"]:
            self.sections[name] = tk.Frame(self.content, bg=t["bg"])

        self._build_calculator()
        self._build_profile()
        self._build_history()
        self._build_about()
        self._build_settings()

        self._show_section("calculator")

    def _build_sidebar(self):
        t = T()
        for w in self.sidebar.winfo_children():
            w.destroy()
        items = [
            ("🧮  " + TR("calculator"), "calculator"),
            ("👤  " + TR("profile"), "profile"),
            ("📋  " + TR("history"), "history"),
            ("ℹ️  " + TR("about"), "about"),
        ]
        tk.Frame(self.sidebar, bg=t["bg2"], height=20).pack()
        for label, key in items:
            btn = tk.Button(self.sidebar, text=label, font=("Courier", 12),
                            bg=t["bg2"], fg=t["fg"], bd=0, padx=20, pady=12,
                            anchor="w", cursor="hand2",
                            activebackground=t["bg3"], activeforeground=t["fg"],
                            command=lambda k=key: self._show_section(k))
            btn.pack(fill="x")

    def _toggle_sidebar(self):
        if self._sidebar_visible:
            self.sidebar.pack_forget()
        else:
            self.sidebar.pack(side="left", fill="y", before=self.content)
        self._sidebar_visible = not self._sidebar_visible

    def _show_section(self, name):
        for f in self.sections.values():
            f.pack_forget()
        self.sections[name].pack(fill="both", expand=True)
        if name == "history":
            self._refresh_history()
        if name == "profile":
            self._refresh_profile()

    # ── Calculator ─────────────────────────────────────────────────
    def _build_calculator(self):
        t = T()
        f = self.sections["calculator"]
        for w in f.winfo_children():
            w.destroy()

        outer, inner = make_scrollable(f, t["bg"])
        outer.pack(fill="both", expand=True)
        pad = tk.Frame(inner, bg=t["bg"])
        pad.pack(fill="both", expand=True, padx=24, pady=16)

        top = tk.Frame(pad, bg=t["bg"])
        top.pack(fill="x")

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

        sf = tk.Frame(left, bg=t["bg3"])
        sf.pack(fill="x", padx=8, pady=8, ipady=8)
        tk.Label(sf, text=TR("quick_stats"), font=("Georgia", 12, "bold"),
                 bg=t["bg3"], fg=t["fg"]).pack(anchor="w", padx=12, pady=(8, 4))
        self.stat_labels = {}
        for key, label, default, color in [
            ("avg", TR("avg_consumption"), "— л/100 км", t["green"]),
            ("dist", "📏 " + TR("distance"), "— км", t["accent"]),
            ("fuel", "⛽ " + TR("fuel"), "— л", t["yellow"]),
            ("cost", "💳 " + TR("cost"), "—", t["red"]),
        ]:
            row = tk.Frame(sf, bg=t["bg3"])
            row.pack(fill="x", padx=12, pady=3)
            tk.Label(row, text=label, font=("Courier", 10), bg=t["bg3"], fg=t["fg2"],
                     width=22, anchor="w").pack(side="left")
            lbl = tk.Label(row, text=default, font=("Courier", 10, "bold"), bg=t["bg3"], fg=color)
            lbl.pack(side="right")
            self.stat_labels[key] = lbl

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
        self.dist_var = tk.StringVar()
        self.fuel_var = tk.StringVar()
        self.price_var = tk.StringVar()
        self.avg_var = tk.StringVar()
        self._build_calc_form()

        btn_row = tk.Frame(right, bg=t["bg2"])
        btn_row.pack(fill="x", padx=20, pady=8)
        tk.Button(btn_row, text=TR("calculate"),
                  font=("Georgia", 12, "bold"), bg=t["btn"], fg="white",
                  bd=0, pady=10, cursor="hand2",
                  command=self._calculate).pack(side="left", fill="x", expand=True, padx=(0, 8))
        tk.Button(btn_row, text=TR("clear"),
                  font=("Courier", 11), bg=t["btn2"], fg=t["fg"],
                  bd=0, pady=10, cursor="hand2",
                  command=self._clear_calc).pack(side="left", fill="x", expand=True)

        self.result_frame = tk.Frame(right, bg=t["bg3"])
        self.result_frame.pack(fill="x", padx=20, pady=(4, 8))
        self.result_container = tk.Frame(self.result_frame, bg=t["bg3"])
        self.result_container.pack(fill="x", padx=16, pady=12)
        self._update_result_widgets("consumption")

        co = tk.Frame(pad, bg=t["bg2"])
        co.pack(fill="x", pady=(12, 0))
        ch = tk.Frame(co, bg=t["bg2"])
        ch.pack(fill="x", padx=16, pady=(10, 4))
        tk.Label(ch, text=TR("fuel_chart_title"), font=("Georgia", 13, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack(side="left")
        df = tk.Frame(ch, bg=t["bg2"])
        df.pack(side="right")
        tk.Label(df, text=TR("chart_days"), font=("Courier", 10),
                 bg=t["bg2"], fg=t["fg2"]).pack(side="left", padx=(0, 6))
        self.chart_days_var = tk.StringVar(value="30")
        dc = ttk.Combobox(df, textvariable=self.chart_days_var,
                          values=["7", "30", "90", "180", "360"],
                          state="readonly", width=5, font=("Courier", 10))
        dc.pack(side="left")
        dc.bind("<<ComboboxSelected>>", lambda e: self._draw_fuel_chart())

        self.chart_canvas = tk.Canvas(co, bg=t["chart_bg"], height=300,
                                      highlightthickness=1, highlightbackground=t["border"])
        self.chart_canvas.pack(fill="x", padx=16, pady=(0, 16))
        self.chart_canvas.bind("<Configure>", lambda e: self._draw_fuel_chart())
        self.chart_canvas.bind("<Motion>", self._on_chart_hover)
        self.chart_canvas.bind("<Leave>", self._on_chart_leave)
        self._chart_data_points = []
        self.after(100, self._draw_fuel_chart)

    def _update_result_widgets(self, mode):
        for w in self.result_container.winfo_children():
            w.destroy()
        t = T()
        if mode == "consumption":
            tk.Label(self.result_container, text=TR("fuel_consumption"),
                     font=("Courier", 10), bg=t["bg3"], fg=t["fg2"]).pack(anchor="w")
            self.result_consumption_label = tk.Label(self.result_container, text="— л/100 км",
                                                     font=("Georgia", 20, "bold"), bg=t["bg3"], fg=t["green"])
            self.result_consumption_label.pack(anchor="w")
            self.result_cost_label = None
        else:
            tk.Label(self.result_container, text=TR("trip_cost"),
                     font=("Courier", 10), bg=t["bg3"], fg=t["fg2"]).pack(anchor="w")
            self.result_cost_label = tk.Label(self.result_container, text="—",
                                              font=("Georgia", 20, "bold"), bg=t["bg3"], fg=t["fg"])
            self.result_cost_label.pack(anchor="w")
            self.result_consumption_label = None

    def _build_calc_form(self):
        t = T()
        for w in self.calc_form_frame.winfo_children():
            w.destroy()
        mode = self._calc_mode.get()
        sym = get_currency_symbol()

        def make_row(parent, label_text, var, unit, icon):
            tk.Label(parent, text=label_text, font=("Courier", 10),
                     bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", pady=(8, 2))
            row = tk.Frame(parent, bg=t["input_bg"], bd=1, relief="solid")
            row.pack(fill="x", pady=2)
            tk.Label(row, text=icon, font=("Courier", 13), bg=t["input_bg"],
                     fg=t["fg2"], padx=8).pack(side="left")
            tk.Entry(row, textvariable=var, font=("Courier", 13),
                     bg=t["input_bg"], fg=t["fg"], insertbackground=t["fg"],
                     bd=0, relief="flat").pack(side="left", fill="x", expand=True, pady=8)
            tk.Label(row, text=unit, font=("Courier", 10), bg=t["input_bg"],
                     fg=t["fg2"], padx=8).pack(side="right")

        make_row(self.calc_form_frame, TR("distance_label"), self.dist_var, "км", "📏")
        if mode == "consumption":
            make_row(self.calc_form_frame, TR("fuel_used"), self.fuel_var, TR("fuel_unit"), "⛽")
            make_row(self.calc_form_frame, TR("fuel_price"), self.price_var,
                     sym + "/" + TR("fuel_unit"), "💰")
        else:
            make_row(self.calc_form_frame, TR("avg_consumption_label"), self.avg_var, "л/100 км", "💧")
            make_row(self.calc_form_frame, TR("fuel_price"), self.price_var,
                     sym + "/" + TR("fuel_unit"), "💰")

    def _on_mode_changed(self):
        self._build_calc_form()
        mode = self._calc_mode.get()
        self._update_result_widgets(mode)
        self._clear_calc()

    def _set_calc_car_image(self, photo_path):
        img = load_car_image(photo_path, 300, 180)
        photo = ImageTk.PhotoImage(img)
        self._photo_refs["calc_car"] = photo
        self.calc_car_label.configure(image=photo, bg=T()["bg2"])

    def _refresh_car_combo(self):
        cars = [TR("no_car")]
        if self.current_user and self._db_connected:
            try:
                user_cars = db_get_cars(self.current_user)
                cars += [c["name"] for c in user_cars]
            except Exception as e:
                print(f"Error loading cars: {e}")
        self.calc_car_combo["values"] = cars
        self.calc_car_var.set(cars[0])

    def _on_car_selected(self, event=None):
        sel = self.calc_car_var.get()
        if sel == TR("no_car") or not self.current_user or not self._db_connected:
            self._set_calc_car_image(None)
            self._draw_fuel_chart()
            return
        try:
            for car in db_get_cars(self.current_user):
                if car["name"] == sel:
                    self._set_calc_car_image(car.get("photo"))
                    avg = car.get("avg_consumption")
                    if avg and hasattr(self, 'avg_var'):
                        self.avg_var.set(str(avg))
                    self._draw_fuel_chart()
                    return
        except Exception:
            pass
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

        if mode == "consumption":
            self.result_consumption_label.configure(text=f"{consumption:.2f} л/100 км")
        else:
            self.result_cost_label.configure(text=f"{cost:,.2f} {sym}")

        self.stat_labels["avg"].configure(text=f"{consumption:.2f} л/100 км")
        self.stat_labels["dist"].configure(text=f"{dist:.0f} км")
        self.stat_labels["fuel"].configure(text=f"{fuel:.1f} л")
        self.stat_labels["cost"].configure(text=f"{cost:,.2f} {sym}")

        if self.current_user and self._db_connected:
            try:
                entry = {
                    "date": datetime.now().strftime("%d.%m.%Y %H:%M"),
                    "car": self.calc_car_var.get(),
                    "distance": dist, "fuel": round(fuel, 2),
                    "price": price, "currency": sym,
                    "consumption": round(consumption, 2), "cost": round(cost, 2),
                }
                db_add_history(self.current_user, entry)

                # Update car's average consumption
                if self.calc_car_var.get() != TR("no_car"):
                    db_update_car_consumption(self.current_user, self.calc_car_var.get(), round(consumption, 2))

                self._draw_fuel_chart()
            except Exception as e:
                print(f"Error saving to database: {e}")
                messagebox.showerror(TR("error"), TR("database_error"))

    def _clear_calc(self):
        self.dist_var.set("")
        self.fuel_var.set("")
        self.price_var.set("")
        self.avg_var.set("")
        if self.result_consumption_label:
            self.result_consumption_label.configure(text="— л/100 км")
        if self.result_cost_label:
            self.result_cost_label.configure(text="—")
        for k, v in {"avg": "— л/100 км", "dist": "— км", "fuel": "— л", "cost": "—"}.items():
            self.stat_labels[k].configure(text=v)

    # ── Chart ──────────────────────────────────────────────────────────────────
    def _on_chart_hover(self, event):
        if not self._chart_data_points:
            return
        c = self.chart_canvas
        best = None
        best_d = 9999
        for (cx, cy, dt, val) in self._chart_data_points:
            d = math.sqrt((event.x - cx) ** 2 + (event.y - cy) ** 2)
            if d < best_d:
                best_d = d
                best = (cx, cy, dt, val)
        if best and best_d < 30:
            cx, cy, dt, val = best
            c.delete("tooltip")
            txt = f"{dt.strftime('%d.%m.%Y')}: {val:.1f} л/100"
            pad = 6
            tw = len(txt) * 7 + pad * 2
            th = 22
            tx = min(cx - tw // 2, c.winfo_width() - tw - 4)
            tx = max(tx, 4)
            ty = cy - th - 8
            if ty < 4:
                ty = cy + 12
            c.create_rectangle(tx, ty, tx + tw, ty + th,
                               fill=T()["bg3"], outline=T()["accent"], tags="tooltip")
            c.create_text(tx + tw // 2, ty + th // 2, text=txt,
                          fill=T()["fg"], font=("Courier", 9), tags="tooltip")
            c.create_oval(cx - 5, cy - 5, cx + 5, cy + 5,
                          fill=T()["accent"], outline=T()["bg2"], width=2, tags="tooltip")

    def _on_chart_leave(self, event):
        if hasattr(self, 'chart_canvas'):
            self.chart_canvas.delete("tooltip")

    def _draw_fuel_chart(self):
        if not hasattr(self, 'chart_canvas') or not self._db_connected:
            return
        c = self.chart_canvas
        t = T()
        c.delete("all")
        self._chart_data_points = []
        W, H = c.winfo_width(), c.winfo_height()
        if W < 10 or H < 10:
            return

        pl, pr, pt, pb = 80, 30, 40, 80
        if not self.current_user:
            c.create_text(W // 2, H // 2, text=TR("login_for_chart"),
                          fill=t["fg2"], font=("Courier", 12))
            return

        try:
            days_limit = int(self.chart_days_var.get())
        except Exception:
            days_limit = 30

        sel = self.calc_car_var.get() if hasattr(self, 'calc_car_var') else TR("no_car")

        try:
            daily_avg = db_get_history_for_chart(self.current_user, sel, days_limit)
        except Exception as e:
            print(f"Error getting chart data: {e}")
            c.create_text(W // 2, H // 2, text=TR("database_error"),
                          fill=t["red"], font=("Courier", 12))
            return

        if not daily_avg:
            c.create_text(W // 2, H // 2, text=TR("no_history_chart"),
                          fill=t["fg2"], font=("Courier", 12))
            return

        values = [v for _, v in daily_avg]
        min_v = max(0, min(values) - 2)
        max_v = max(values) + 2
        rng = max_v - min_v if max_v != min_v else 1
        cw, ch = W - pl - pr, H - pt - pb

        c.create_line(pl, H - pb, W - pr, H - pb, fill=t["fg2"], width=2)
        c.create_line(pl, pt, pl, H - pb, fill=t["fg2"], width=2)

        for gi in range(6):
            gv = min_v + gi * rng / 5
            gy = pt + (1 - (gv - min_v) / rng) * ch
            c.create_line(pl, gy, W - pr, gy, fill=t["chart_grid"], dash=(4, 6))
            c.create_text(pl - 10, gy, text=f"{gv:.1f}", anchor="e", fill=t["fg2"], font=("Courier", 9))

        yg = pt + (1 - (8 - min_v) / rng) * ch if 8 > min_v and 8 < max_v else None
        yy = pt + (1 - (12 - min_v) / rng) * ch if 12 > min_v and 12 < max_v else None

        if yg:
            c.create_rectangle(pl, pt, W - pr, min(yg, H - pb), fill="#1a2f1a", outline="")
        if yy and yg:
            c.create_rectangle(pl, max(yg, pt), W - pr, min(yy, H - pb), fill="#2a2410", outline="")
        if yy:
            c.create_rectangle(pl, max(yy, pt), W - pr, H - pb, fill="#2a1212", outline="")

        bar_width = max(10, min(40, cw / len(daily_avg) - 5))
        for i, (dt, val) in enumerate(daily_avg):
            x = pl + (i + 0.5) * cw / len(daily_avg)
            y = pt + (1 - (val - min_v) / rng) * ch

            if val <= 8:
                color = t["chart_bar1"]
            elif val <= 12:
                color = t["chart_bar2"]
            else:
                color = t["chart_bar3"]

            bar_height = H - pb - y
            c.create_rectangle(x - bar_width / 2, y, x + bar_width / 2, H - pb,
                               fill=color, outline=t["bg2"], width=1)
            c.create_text(x, y - 10, text=f"{val:.1f}",
                          fill=t["fg"], font=("Courier", 8, "bold"))
            self._chart_data_points.append((x, y, dt, val))

        step = max(1, len(daily_avg) // 10)
        for i in range(0, len(daily_avg), step):
            dt, _ = daily_avg[i]
            x = pl + (i + 0.5) * cw / len(daily_avg)
            c.create_text(x, H - pb + 15, text=dt.strftime("%d.%m"),
                          fill=t["fg2"], font=("Courier", 8), angle=45)

    # ── Profile ────────────────────────────────────────────────────────────────
    def _build_profile(self):
        t = T()
        f = self.sections["profile"]
        for w in f.winfo_children():
            w.destroy()
        outer, inner = make_scrollable(f, t["bg"])
        outer.pack(fill="both", expand=True)
        pad = tk.Frame(inner, bg=t["bg"])
        pad.pack(fill="both", expand=True, padx=32, pady=24)

        if not self._db_connected:
            tk.Label(pad, text=TR("database_error"), font=("Georgia", 14, "bold"),
                     bg=t["bg"], fg=t["red"]).pack(pady=40)
            tk.Label(pad, text=TR("database_retry"), font=("Courier", 10),
                     bg=t["bg"], fg=t["fg2"]).pack()
            return

        if not self.current_user:
            self._build_login_form(pad)
        else:
            self._build_logged_in_profile(pad)

    def _build_login_form(self, parent):
        t = T()
        self._auth_mode = tk.StringVar(value="login")
        tf = tk.Frame(parent, bg=t["bg"])
        tf.pack(pady=(0, 16))
        tk.Button(tf, text=TR("login"), font=("Georgia", 12, "bold"),
                  bg=t["btn"], fg="white", bd=0, padx=20, pady=6,
                  command=lambda: self._auth_mode.set("login") or build()).pack(side="left", padx=4)
        tk.Button(tf, text=TR("register"), font=("Georgia", 12),
                  bg=t["btn2"], fg=t["fg"], bd=0, padx=20, pady=6,
                  command=lambda: self._auth_mode.set("register") or build()).pack(side="left", padx=4)

        self._auth_form_frame = tk.Frame(parent, bg=t["bg2"], padx=32, pady=24)
        self._auth_form_frame.pack(fill="x")

        def build():
            for w in self._auth_form_frame.winfo_children():
                w.destroy()
            mode = self._auth_mode.get()
            tk.Label(self._auth_form_frame,
                     text=TR("create_account") if mode == "register" else TR("login_account"),
                     font=("Georgia", 15, "bold"), bg=t["bg2"], fg=t["fg"]).pack(pady=(0, 16))

            tk.Label(self._auth_form_frame, text=TR("email"),
                     font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w")
            email_entry = tk.Entry(self._auth_form_frame, font=("Courier", 12),
                                   bg=t["input_bg"], fg=t["fg"], bd=1, relief="solid")
            email_entry.pack(fill="x", pady=4, ipady=6)

            tk.Label(self._auth_form_frame, text=TR("password"),
                     font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w")
            pw_row = tk.Frame(self._auth_form_frame, bg=t["input_bg"], bd=1, relief="solid")
            pw_row.pack(fill="x", pady=4)
            pw_entry = tk.Entry(pw_row, font=("Courier", 12), show="•",
                                bg=t["input_bg"], fg=t["fg"], bd=0)
            pw_entry.pack(side="left", fill="x", expand=True, ipady=6, padx=4)
            pw_vis = [False]

            def toggle_pw():
                pw_vis[0] = not pw_vis[0]
                pw_entry.configure(show="" if pw_vis[0] else "•")
                eye.configure(text="🙈" if pw_vis[0] else "👁")

            eye = tk.Button(pw_row, text="👁", font=("Courier", 11), bg=t["input_bg"],
                            fg=t["fg2"], bd=0, cursor="hand2", command=toggle_pw)
            eye.pack(side="right", padx=2)

            if mode == "login":
                forgot_btn = tk.Button(self._auth_form_frame, text=TR("forgot_password"),
                                       font=("Courier", 9, "underline"), bg=t["bg2"], fg=t["accent"],
                                       bd=0, cursor="hand2",
                                       command=self._show_forgot_password_dialog)
                forgot_btn.pack(anchor="e", pady=(2, 0))

            pw2_entry = None
            if mode == "register":
                tk.Label(self._auth_form_frame, text=TR("repeat_password"),
                         font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w")
                pw2_row = tk.Frame(self._auth_form_frame, bg=t["input_bg"], bd=1, relief="solid")
                pw2_row.pack(fill="x", pady=4)
                pw2_entry = tk.Entry(pw2_row, font=("Courier", 12), show="•",
                                     bg=t["input_bg"], fg=t["fg"], bd=0)
                pw2_entry.pack(side="left", fill="x", expand=True, ipady=6, padx=4)

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
                    try:
                        if db_user_exists(email):
                            status.configure(text=TR("user_exists"))
                            return
                        db_create_user(email, hash_pw(pw))
                        self.current_user = email
                        self._save_settings()
                        self._refresh_car_combo()
                        self._refresh_profile()
                    except Exception as e:
                        print(f"Registration error: {e}")
                        status.configure(text=TR("database_error"))
                else:
                    try:
                        stored_pw = db_get_user(email)
                        if not stored_pw or stored_pw != hash_pw(pw):
                            status.configure(text=TR("wrong_credentials"))
                            return
                        self.current_user = email
                        self._load_user_settings()
                        self._refresh_car_combo()
                        self._refresh_profile()
                    except Exception as e:
                        print(f"Login error: {e}")
                        status.configure(text=TR("database_error"))

            btn_text = TR("login") if mode == "login" else TR("register")
            tk.Button(self._auth_form_frame, text=btn_text,
                      font=("Georgia", 12, "bold"), bg=t["btn"], fg="white",
                      bd=0, pady=8, cursor="hand2", command=submit).pack(fill="x", pady=8)

        build()

    def _show_forgot_password_dialog(self):
        dialog = tk.Toplevel(self)
        dialog.title(TR("forgot_password_title"))
        dialog.geometry("400x300")
        dialog.configure(bg=T()["bg2"])
        dialog.transient(self)
        dialog.grab_set()

        tk.Label(dialog, text=TR("forgot_password_title"),
                 font=("Georgia", 14, "bold"), bg=T()["bg2"], fg=T()["fg"]).pack(pady=16)
        tk.Label(dialog, text=TR("email"),
                 font=("Courier", 10), bg=T()["bg2"], fg=T()["fg2"]).pack(anchor="w", padx=20)
        email_entry = tk.Entry(dialog, font=("Courier", 12), bg=T()["input_bg"],
                               fg=T()["fg"], bd=1, relief="solid")
        email_entry.pack(fill="x", padx=20, pady=4, ipady=6)

        tk.Label(dialog, text=TR("new_password"),
                 font=("Courier", 10), bg=T()["bg2"], fg=T()["fg2"]).pack(anchor="w", padx=20, pady=(8, 0))
        pw_entry = tk.Entry(dialog, font=("Courier", 12), show="•", bg=T()["input_bg"],
                            fg=T()["fg"], bd=1, relief="solid")
        pw_entry.pack(fill="x", padx=20, pady=4, ipady=6)

        status = tk.Label(dialog, text="", font=("Courier", 10), bg=T()["bg2"], fg=T()["red"])
        status.pack(pady=4)

        def reset_pw():
            email = email_entry.get().strip().lower()
            pw = pw_entry.get().strip()
            if not is_valid_email(email):
                status.configure(text=TR("invalid_email"))
                return
            try:
                if not db_user_exists(email):
                    status.configure(text="Email не найден")
                    return
                if len(pw) < 6:
                    status.configure(text=TR("password_short"))
                    return
                db_update_password(email, hash_pw(pw))
                messagebox.showinfo(TR("password_changed"), TR("new_password_set"))
                dialog.destroy()
            except Exception as e:
                print(f"Password reset error: {e}")
                status.configure(text=TR("database_error"))

        tk.Button(dialog, text=TR("change_password_btn"),
                  font=("Georgia", 11, "bold"), bg=T()["btn"], fg="white",
                  bd=0, pady=8, command=reset_pw).pack(fill="x", padx=20, pady=12)

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
                  command=self._logout).pack(side="left", padx=(0, 8))
        tk.Button(btn_frame, text=TR("delete_account"), font=("Courier", 10),
                  bg=t["red"], fg="white", bd=0, padx=12, pady=4,
                  command=self._delete_account).pack(side="left")

        pw_frame = tk.Frame(parent, bg=t["bg2"])
        pw_frame.pack(fill="x", pady=8, ipady=4)
        tk.Label(pw_frame, text=TR("change_password"), font=("Georgia", 13, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=16, pady=(12, 4))

        for label_text, var_name in [(TR("new_password"), "new_pw"), (TR("repeat_new_password"), "new_pw2")]:
            tk.Label(pw_frame, text=label_text, font=("Courier", 9), bg=t["bg2"],
                     fg=t["fg2"]).pack(anchor="w", padx=16)
            row = tk.Frame(pw_frame, bg=t["input_bg"], bd=1, relief="solid")
            row.pack(fill="x", padx=16, pady=(2, 4))
            entry = tk.Entry(row, font=("Courier", 11), show="•", bg=t["input_bg"],
                             fg=t["fg"], bd=0, relief="flat")
            entry.pack(side="left", fill="x", expand=True, ipady=5, padx=4)
            setattr(self, f"_{var_name}_entry", entry)

        pw_status = tk.Label(pw_frame, text="", font=("Courier", 9), bg=t["bg2"], fg=t["green"])
        pw_status.pack(anchor="w", padx=16)

        def change_pw():
            np1 = self._new_pw_entry.get()
            np2 = self._new_pw2_entry.get()
            if len(np1) < 6:
                pw_status.configure(text=TR("password_short"), fg=t["red"])
                return
            if np1 != np2:
                pw_status.configure(text=TR("passwords_mismatch"), fg=t["red"])
                return
            try:
                db_update_password(self.current_user, hash_pw(np1))
                pw_status.configure(text=TR("password_changed"), fg=t["green"])
                self._new_pw_entry.delete(0, "end")
                self._new_pw2_entry.delete(0, "end")
            except Exception as e:
                print(f"Password change error: {e}")
                pw_status.configure(text=TR("database_error"), fg=t["red"])

        tk.Button(pw_frame, text=TR("change_password_btn"), font=("Courier", 10),
                  bg=t["btn"], fg="white", bd=0, padx=10, pady=5,
                  command=change_pw).pack(anchor="w", padx=16, pady=(0, 12))
        tk.Frame(parent, bg=t["border"], height=1).pack(fill="x", pady=16)

        cars_header = tk.Frame(parent, bg=t["bg"])
        cars_header.pack(fill="x")
        tk.Label(cars_header, text=TR("my_cars"), font=("Georgia", 14, "bold"),
                 bg=t["bg"], fg=t["fg"]).pack(side="left")
        tk.Button(cars_header, text=TR("add_car"), font=("Courier", 10),
                  bg=t["btn"], fg="white", bd=0, padx=12, pady=4,
                  command=self._add_car_inline).pack(side="right")

        self.cars_container = tk.Frame(parent, bg=t["bg"])
        self.cars_container.pack(fill="x", pady=8)
        self._add_car_form_frame = tk.Frame(parent, bg=t["bg2"])
        self._add_car_form_visible = False
        self._render_cars()

    def _add_car_inline(self):
        t = T()
        try:
            car_count = db_count_cars(self.current_user)
            if car_count >= 25:
                messagebox.showwarning(TR("error"), TR("no_cars_limit"))
                return
        except Exception as e:
            print(f"Error checking car count: {e}")

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
                 bd=1, relief="solid").pack(fill="x", padx=16, pady=(2, 8), ipady=6)

        photo_path_var = tk.StringVar()
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
        photo_name_lbl.pack(anchor="w")

        def choose_photo():
            path = filedialog.askopenfilename(filetypes=[("Images", "*.jpg *.jpeg *.png *.bmp *.webp")])
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
                except Exception:
                    pass

        tk.Button(right_col, text=TR("choose_photo"), font=("Courier", 10),
                  bg=t["btn2"], fg=t["fg"], bd=0, padx=12, pady=6,
                  command=choose_photo).pack(anchor="w", pady=(0, 6))

        status = tk.Label(self._add_car_form_frame, text="", font=("Courier", 9),
                          bg=t["bg2"], fg=t["red"])
        status.pack(anchor="w", padx=16)

        btn_row = tk.Frame(self._add_car_form_frame, bg=t["bg2"])
        btn_row.pack(fill="x", padx=16, pady=12)

        def save_car():
            name = name_var.get().strip()
            if not name:
                status.configure(text=TR("enter_name"))
                return
            try:
                db_add_car(self.current_user, name, photo_path_var.get() or None)
                self._add_car_form_frame.pack_forget()
                self._add_car_form_visible = False
                self._render_cars()
                self._refresh_car_combo()
            except Exception as e:
                print(f"Error adding car: {e}")
                status.configure(text=TR("database_error"))

        cancel_text = "✕ Отмена" if current_language == "ru" else "✕ Cancel"
        tk.Button(btn_row, text=TR("save"), font=("Georgia", 11, "bold"),
                  bg=t["btn"], fg="white", bd=0, pady=8,
                  command=save_car).pack(side="left", fill="x", expand=True, padx=(0, 8))
        tk.Button(btn_row, text=cancel_text, font=("Courier", 10),
                  bg=t["btn2"], fg=t["fg"], bd=0, pady=8,
                  command=lambda: [self._add_car_form_frame.pack_forget(),
                                   setattr(self, '_add_car_form_visible', False)]).pack(side="left")

    def _render_cars(self):
        t = T()
        for w in self.cars_container.winfo_children():
            w.destroy()
        if not self.current_user or not self._db_connected:
            return

        try:
            cars = db_get_cars(self.current_user)
        except Exception as e:
            print(f"Error loading cars: {e}")
            tk.Label(self.cars_container, text=TR("database_error"),
                     font=("Courier", 11), bg=t["bg"], fg=t["red"]).pack(pady=16)
            return

        if not cars:
            tk.Label(self.cars_container, text=TR("no_cars"),
                     font=("Courier", 11), bg=t["bg"], fg=t["fg2"]).pack(pady=16)
            return

        for row_idx in range(5):
            row = tk.Frame(self.cars_container, bg=t["bg"])
            row.pack(fill="x", pady=4)
            for col_idx in range(5):
                idx = row_idx * 5 + col_idx
                if idx >= len(cars):
                    break
                car = cars[idx]

                card = tk.Frame(row, bg=t["bg2"], bd=0, relief="flat", padx=6, pady=6)
                card.pack(side="left", padx=6, pady=4)

                img_box = tk.Frame(card, bg=t["bg2"], width=120, height=75)
                img_box.pack_propagate(False)
                img_box.pack()
                ph = car.get("photo")
                img = load_car_image(ph, 120, 75)
                photo = ImageTk.PhotoImage(img)
                self._photo_refs[f"car_{idx}"] = photo
                tk.Label(img_box, image=photo, bg=t["bg2"]).place(relx=0.5, rely=0.5, anchor="center")

                tk.Label(card, text=car["name"], font=("Courier", 9, "bold"),
                         bg=t["bg2"], fg=t["fg"]).pack(pady=2)
                avg = car.get("avg_consumption")
                if avg:
                    tk.Label(card, text=f"💧 {avg} л/100", font=("Courier", 7),
                             bg=t["bg2"], fg=t["fg2"]).pack()
                tk.Button(card, text=TR("delete"), font=("Courier", 8),
                          bg=t["red"], fg="white", bd=0, padx=6, pady=2,
                          command=lambda i=idx, name=car["name"]: self._delete_car(name)).pack(pady=(3, 0))

    def _delete_car(self, car_name):
        if messagebox.askyesno(TR("delete_confirm"), TR("delete_car_confirm")):
            try:
                db_delete_car(self.current_user, car_name)
                self._render_cars()
                self._refresh_car_combo()
            except Exception as e:
                print(f"Error deleting car: {e}")
                messagebox.showerror(TR("error"), TR("database_error"))

    def _logout(self):
        self.current_user = None
        self._refresh_car_combo()
        self._refresh_profile()
        self._draw_fuel_chart()

    def _delete_account(self):
        if messagebox.askyesno(TR("delete_account_confirm"), TR("delete_account_msg")):
            try:
                db_delete_user(self.current_user)
                self.current_user = None
                self._refresh_car_combo()
                self._refresh_profile()
                self._draw_fuel_chart()
            except Exception as e:
                print(f"Error deleting account: {e}")
                messagebox.showerror(TR("error"), TR("database_error"))

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

        outer, inner = make_scrollable(f, t["bg"])
        outer.pack(fill="both", expand=True)
        pad = tk.Frame(inner, bg=t["bg"])
        pad.pack(fill="both", expand=True, padx=24, pady=20)

        tk.Label(pad, text=TR("history_title"), font=("Georgia", 16, "bold"),
                 bg=t["bg"], fg=t["fg"]).pack(anchor="w", pady=(0, 4))

        if not self.current_user or not self._db_connected:
            tk.Label(pad, text=TR("login_for_history"),
                     font=("Courier", 12), bg=t["bg"], fg=t["fg2"]).pack(pady=40)
            return

        hdr = tk.Frame(pad, bg=t["bg"])
        hdr.pack(fill="x")
        tk.Button(hdr, text=TR("clear_all"), font=("Courier", 9),
                  bg=t["red"], fg="white", bd=0, padx=8, pady=4,
                  command=self._clear_all_history).pack(side="right")

        self._hist_inner = tk.Frame(pad, bg=t["bg"])
        self._hist_inner.pack(fill="x", pady=8)
        self._render_history_entries()

    def _render_history_entries(self):
        t = T()
        if not hasattr(self, '_hist_inner'):
            return
        for w in self._hist_inner.winfo_children():
            w.destroy()
        if not self.current_user or not self._db_connected:
            return

        try:
            history = db_get_history(self.current_user)
        except Exception as e:
            print(f"Error loading history: {e}")
            tk.Label(self._hist_inner, text=TR("database_error"),
                     font=("Courier", 12), bg=t["bg"], fg=t["red"]).pack(pady=40)
            return

        if not history:
            tk.Label(self._hist_inner, text=TR("history_empty"),
                     font=("Courier", 12), bg=t["bg"], fg=t["fg2"]).pack(pady=40)
            return

        for i, entry in enumerate(history):
            card = tk.Frame(self._hist_inner, bg=t["bg2"], pady=8, padx=8)
            card.pack(fill="x", pady=6)

            top_row = tk.Frame(card, bg=t["bg2"])
            top_row.pack(fill="x", pady=(0, 8))

            tk.Label(top_row, text=f"📅 {entry['date']}",
                     font=("Courier", 10, "bold"), bg=t["bg2"], fg=t["fg"]).pack(side="left")

            btn_frame = tk.Frame(top_row, bg=t["bg2"])
            btn_frame.pack(side="right")

            sym = entry.get("currency", "₽")
            copy_text = (
                f"Date: {entry['date']}\n"
                f"Car: {entry.get('car', '—')}\n"
                f"Distance: {entry['distance']} km\n"
                f"Fuel: {entry['fuel']} L\n"
                f"Price: {entry['price']} {sym}/L\n"
                f"Consumption: {entry['consumption']} L/100km\n"
                f"Cost: {entry['cost']:,.2f} {sym}"
            )

            def copy_entry(txt=copy_text):
                self.clipboard_clear()
                self.clipboard_append(txt)
                messagebox.showinfo(TR("copied"), TR("copied_msg"))

            copy_btn = tk.Button(btn_frame, text="📋 " + TR("copy"),
                                 font=("Courier", 9),
                                 bg=t["green"], fg="white", bd=0,
                                 padx=8, pady=4, cursor="hand2",
                                 command=copy_entry)
            copy_btn.pack(side="left", padx=(0, 8))

            content = tk.Frame(card, bg=t["bg2"])
            content.pack(fill="x")

            info = tk.Frame(content, bg=t["bg2"])
            info.pack(side="left", fill="x", expand=True)

            tk.Label(info, text=f"🚗 {entry.get('car', '—')}",
                     font=("Courier", 11, "bold"), bg=t["bg2"], fg=t["accent"]).pack(anchor="w")

            data_text = f"📏 {TR('distance')}: {entry['distance']} км"
            tk.Label(info, text=data_text,
                     font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", pady=(4, 0))

            data_text = f"⛽ {TR('fuel')}: {entry['fuel']} л"
            tk.Label(info, text=data_text,
                     font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w")

            data_text = f"💰 {TR('fuel_price')}: {entry['price']} {sym}/{TR('fuel_unit')}"
            tk.Label(info, text=data_text,
                     font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w")

            results_frame = tk.Frame(info, bg=t["bg3"], padx=8, pady=4)
            results_frame.pack(fill="x", pady=(8, 0))

            result_text = f"💧 {TR('avg_consumption')}: {entry['consumption']} л/100км"
            tk.Label(results_frame, text=result_text,
                     font=("Courier", 11, "bold"), bg=t["bg3"], fg=t["green"]).pack(anchor="w")

            result_text = f"💳 {TR('trip_cost')}: {entry['cost']:,.2f} {sym}"
            tk.Label(results_frame, text=result_text,
                     font=("Courier", 11, "bold"), bg=t["bg3"], fg=t["yellow"]).pack(anchor="w")

            if i < len(history) - 1:
                separator = tk.Frame(self._hist_inner, bg=t["border"], height=1)
                separator.pack(fill="x", pady=4)

    def _clear_all_history(self):
        if messagebox.askyesno(TR("clear_history_confirm"), TR("clear_history_msg")):
            try:
                db_clear_history(self.current_user)
                self._render_history_entries()
            except Exception as e:
                print(f"Error clearing history: {e}")
                messagebox.showerror(TR("error"), TR("database_error"))

    def _refresh_history(self):
        if hasattr(self, '_hist_inner'):
            self._render_history_entries()

    # ── About ──────────────────────────────────────────────────────────────────
    def _build_about(self):
        t = T()
        f = self.sections["about"]
        for w in f.winfo_children():
            w.destroy()
        outer, inner = make_scrollable(f, t["bg"])
        outer.pack(fill="both", expand=True)
        pad = tk.Frame(inner, bg=t["bg"])
        pad.pack(anchor="center", padx=48, pady=32)

        hero = tk.Frame(pad, bg=t["bg2"], pady=32)
        hero.pack(fill="x", pady=(0, 24))
        tk.Label(hero, text="⛽", font=("Courier", 48), bg=t["bg2"], fg=t["accent"]).pack()
        tk.Label(hero, text=TR("app_title"), font=("Georgia", 22, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack()
        tk.Label(hero, text=TR("version"), font=("Courier", 10),
                 bg=t["bg2"], fg=t["fg2"]).pack(pady=4)

        for title_key, body_key in [
            ("about_section1_title", "about_section1"),
            ("about_section2_title", "about_section2"),
            ("about_section3_title", "about_section3"),
            ("about_section4_title", "about_section4"),
            ("about_section5_title", "about_section5"),
        ]:
            sec = tk.Frame(pad, bg=t["bg2"])
            sec.pack(fill="x", pady=8, ipadx=16, ipady=12)
            tk.Label(sec, text=TR(title_key), font=("Georgia", 13, "bold"),
                     bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=16, pady=(8, 2))
            tk.Label(sec, text=TR(body_key), font=("Courier", 10),
                     bg=t["bg2"], fg=t["fg2"], justify="left",
                     wraplength=700).pack(anchor="w", padx=16, pady=(2, 8))

        tk.Label(pad, text=TR("footer"), font=("Courier", 9),
                 bg=t["bg"], fg=t["fg2"]).pack(pady=16)

    # ── Settings ───────────────────────────────────────────────────────────────
    def _build_settings(self):
        t = T()
        f = self.sections["settings"]
        for w in f.winfo_children():
            w.destroy()
        outer, inner = make_scrollable(f, t["bg"])
        outer.pack(fill="both", expand=True)
        pad = tk.Frame(inner, bg=t["bg"])
        pad.pack(fill="both", expand=True, padx=48, pady=32)

        tk.Label(pad, text=TR("settings_title"), font=("Georgia", 18, "bold"),
                 bg=t["bg"], fg=t["fg"]).pack(anchor="w", pady=(0, 24))

        tc = tk.Frame(pad, bg=t["bg2"], pady=16)
        tc.pack(fill="x", pady=8)
        tk.Label(tc, text=TR("theme"), font=("Georgia", 13, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=20)
        tk.Label(tc, text=TR("theme_desc"), font=("Courier", 10),
                 bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", padx=20)
        tr = tk.Frame(tc, bg=t["bg2"])
        tr.pack(padx=20, pady=12, anchor="w")

        def apply_theme(name):
            global current_theme
            current_theme = name
            if self.current_user:
                self._save_settings()
            self._rebuild_ui()
            self._show_section("settings")

        tk.Button(tr, text=TR("dark_theme"), font=("Georgia", 11, "bold"),
                  bg="#1f2937", fg="#e5e7eb", bd=0, padx=20, pady=8,
                  command=lambda: apply_theme("dark")).pack(side="left", padx=(0, 12))
        tk.Button(tr, text=TR("light_theme"), font=("Georgia", 11, "bold"),
                  bg="#e2e8f0", fg="#1a202c", bd=0, padx=20, pady=8,
                  command=lambda: apply_theme("light")).pack(side="left")

        cc = tk.Frame(pad, bg=t["bg2"], pady=16)
        cc.pack(fill="x", pady=8)
        tk.Label(cc, text=TR("currency_setting"), font=("Georgia", 13, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=20)
        tk.Label(cc, text=TR("currency_desc"), font=("Courier", 10),
                 bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", padx=20)
        cv = tk.StringVar(value=current_currency)
        ccombo = ttk.Combobox(cc, textvariable=cv, values=list(CURRENCIES.keys()),
                              state="readonly", width=14, font=("Courier", 11))
        ccombo.pack(anchor="w", padx=20, pady=8)
        cs = tk.Label(cc, text=f"{TR('current')} {get_currency_symbol()}",
                      font=("Courier", 10), bg=t["bg2"], fg=t["accent"])
        cs.pack(anchor="w", padx=20)

        def apply_currency(event=None):
            global current_currency
            current_currency = cv.get()
            if self.current_user:
                self._save_settings()
            cs.configure(text=f"{TR('current')} {get_currency_symbol()}")
            if hasattr(self, 'calc_form_frame'):
                self._build_calc_form()

        ccombo.bind("<<ComboboxSelected>>", apply_currency)

        lc = tk.Frame(pad, bg=t["bg2"], pady=16)
        lc.pack(fill="x", pady=8)
        tk.Label(lc, text=TR("language_setting"), font=("Georgia", 13, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=20)
        tk.Label(lc, text=TR("language_desc"), font=("Courier", 10),
                 bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", padx=20)
        lr = tk.Frame(lc, bg=t["bg2"])
        lr.pack(padx=20, pady=12, anchor="w")

        def apply_language(lang):
            global current_language
            current_language = lang
            if self.current_user:
                self._save_settings()
            self._rebuild_ui()
            self._show_section("settings")

        tk.Button(lr, text="🇷🇺  Русский", font=("Georgia", 11, "bold"),
                  bg=t["btn"] if current_language == "ru" else t["btn2"],
                  fg="white" if current_language == "ru" else t["fg"],
                  bd=0, padx=20, pady=8,
                  command=lambda: apply_language("ru")).pack(side="left", padx=(0, 12))
        tk.Button(lr, text="🇬🇧  English", font=("Georgia", 11, "bold"),
                  bg=t["btn"] if current_language == "en" else t["btn2"],
                  fg="white" if current_language == "en" else t["fg"],
                  bd=0, padx=20, pady=8,
                  command=lambda: apply_language("en")).pack(side="left")


if __name__ == "__main__":
    app = FuelApp()
    app.mainloop()
