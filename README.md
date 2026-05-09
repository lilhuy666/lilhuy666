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

# ─── Constants ──────────────────────────────────────────────────────────────────
MAX_CARS_PER_USER = 25
CAR_PHOTO_WIDTH = 300
CAR_PHOTO_HEIGHT = 180
CAR_PREVIEW_WIDTH = 120
CAR_PREVIEW_HEIGHT = 75

# ─── Database configuration ────────────────────────────────────────────────────
DB_CONFIG = {
    'dbname': 'fuel_calc',
    'user': 'postgres',
    'password': '12345',
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
    except Exception as e:
        print(f"Error creating user: {e}")
        raise
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
            if row[0] and isinstance(row[0], str) and os.path.exists(row[0]):
                try:
                    os.remove(row[0])
                except Exception as e:
                    print(f"Error deleting photo: {e}")
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
    except Exception as e:
        print(f"Error adding car: {e}")
        raise
    finally:
        put_db_connection(conn)


def db_delete_car(email, name):
    conn = get_db_connection()
    try:
        cursor = conn.cursor()
        cursor.execute("SELECT photo_path FROM cars WHERE email = %s AND name = %s", (email, name))
        row = cursor.fetchone()
        if row and row[0] and isinstance(row[0], str) and os.path.exists(row[0]):
            try:
                os.remove(row[0])
            except Exception as e:
                print(f"Error deleting car photo: {e}")
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
    except Exception as e:
        print(f"Error updating car consumption: {e}")
    finally:
        put_db_connection(conn)


def db_add_history(email, entry):
    """Добавление записи в историю"""
    conn = get_db_connection()
    try:
        cursor = conn.cursor()
        cursor.execute("""
            INSERT INTO history (email, car_name, distance, fuel, price, currency, consumption, cost, date)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
        """, (
            email, 
            entry.get('car'), 
            entry['distance'], 
            entry['fuel'],
            entry['price'], 
            entry.get('currency'), 
            entry['consumption'],
            entry['cost'], 
            datetime.strptime(entry['date'], "%d.%m.%Y %H:%M")
        ))
        conn.commit()
        print(f"History added successfully for {email}")
        return True
    except Exception as e:
        print(f"Error adding history: {e}")
        conn.rollback()
        return False
    finally:
        put_db_connection(conn)


def db_get_history(email, car_name=None, days=None):
    """Получение истории с фильтрацией"""
    conn = get_db_connection()
    try:
        cursor = conn.cursor()
        
        # Базовый запрос
        query = """
            SELECT car_name, distance, fuel, price, currency, consumption, cost, 
                   TO_CHAR(date, 'DD.MM.YYYY HH24:MI') as date_str,
                   date as raw_date,
                   id
            FROM history 
            WHERE email = %s
        """
        params = [email]

        # Фильтр по автомобилю
        if car_name and car_name not in ["— Без авто —", "— No car —"]:
            query += " AND car_name = %s"
            params.append(car_name)

        # Фильтр по дням
        if days:
            query += " AND date >= CURRENT_DATE - %s * INTERVAL '1 day'"
            params.append(days)

        # Сортировка по дате (сначала новые)
        query += " ORDER BY date DESC"

        cursor.execute(query, params)
        results = []
        for r in cursor.fetchall():
            results.append({
                'id': r[9],
                'car': r[0], 
                'distance': float(r[1]), 
                'fuel': float(r[2]), 
                'price': float(r[3]),
                'currency': r[4], 
                'consumption': float(r[5]), 
                'cost': float(r[6]),
                'date': r[7], 
                'raw_date': r[8]
            })
        return results
    except Exception as e:
        print(f"Error getting history: {e}")
        return []
    finally:
        put_db_connection(conn)


def db_delete_history_entry(email, entry_id):
    """Удаление конкретной записи из истории"""
    conn = get_db_connection()
    try:
        cursor = conn.cursor()
        cursor.execute(
            "DELETE FROM history WHERE email = %s AND id = %s",
            (email, entry_id)
        )
        conn.commit()
        return cursor.rowcount > 0
    except Exception as e:
        print(f"Error deleting history entry: {e}")
        conn.rollback()
        return False
    finally:
        put_db_connection(conn)


def db_clear_history(email):
    """Очистка всей истории пользователя"""
    conn = get_db_connection()
    try:
        cursor = conn.cursor()
        cursor.execute("DELETE FROM history WHERE email = %s", (email,))
        conn.commit()
        return True
    except Exception as e:
        print(f"Error clearing history: {e}")
        conn.rollback()
        return False
    finally:
        put_db_connection(conn)


def db_get_history_stats(email, car_name=None):
    """Получение статистики по истории"""
    conn = get_db_connection()
    try:
        cursor = conn.cursor()
        query = """
            SELECT 
                COUNT(*) as total_trips,
                SUM(distance) as total_distance,
                SUM(fuel) as total_fuel,
                AVG(consumption) as avg_consumption,
                SUM(cost) as total_cost,
                MIN(date) as first_trip,
                MAX(date) as last_trip
            FROM history 
            WHERE email = %s
        """
        params = [email]
        
        if car_name and car_name not in ["— Без авто —", "— No car —"]:
            query += " AND car_name = %s"
            params.append(car_name)
            
        cursor.execute(query, params)
        result = cursor.fetchone()
        
        if result and result[0] > 0:
            return {
                'total_trips': result[0],
                'total_distance': float(result[1]) if result[1] else 0,
                'total_fuel': float(result[2]) if result[2] else 0,
                'avg_consumption': float(result[3]) if result[3] else 0,
                'total_cost': float(result[4]) if result[4] else 0,
                'first_trip': result[5],
                'last_trip': result[6]
            }
        return None
    except Exception as e:
        print(f"Error getting history stats: {e}")
        return None
    finally:
        put_db_connection(conn)


def db_get_history_for_chart(email, car_name, days):
    """Получение данных для графика"""
    conn = get_db_connection()
    try:
        cursor = conn.cursor()
        days = int(days)
        
        query = """
            SELECT DATE(date) as day, AVG(consumption) as avg_consumption
            FROM history 
            WHERE email = %s 
              AND date >= CURRENT_DATE - %s * INTERVAL '1 day'
        """
        params = [email, days]

        if car_name and car_name not in ["— Без авто —", "— No car —"]:
            query += " AND car_name = %s"
            params.append(car_name)

        query += " GROUP BY DATE(date) ORDER BY day"

        cursor.execute(query, params)
        results = []
        for r in cursor.fetchall():
            if r[0] and r[1] is not None:
                results.append((r[0], float(r[1])))
        return results
    except Exception as e:
        print(f"Error getting chart data: {e}")
        return []
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


# ─── Data storage (fallback) ──────────────────────────────────────────────
DATA_FILE = os.path.join(os.path.expanduser("~"), ".fuel_calc_data.json")
PHOTO_DIR = os.path.join(os.path.expanduser("~"), ".fuel_calc_photos")
os.makedirs(PHOTO_DIR, exist_ok=True)


# ─── Translations ──────────────────────────────────────────────────────────────
TRANSLATIONS = {
    "ru": {
        "app_title": "Калькулятор расхода топлива",
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
        "password_short": "Пароль должен быть не менее 6 символов",
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
        "my_cars": f"Мои автомобили (макс. {MAX_CARS_PER_USER})",
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
        "about_section2": f"• Точный расчёт расхода\n• Расчёт стоимости поездки\n• Хранение до {MAX_CARS_PER_USER} автомобилей\n• История расчётов\n• PostgreSQL база данных",
        "about_section3_title": "Как использовать",
        "about_section3": "1. Зарегистрируйтесь\n2. Добавьте автомобиль\n3. Выберите режим расчёта\n4. Введите данные\n5. Нажмите «Рассчитать»",
        "about_section4_title": "Безопасность",
        "about_section4": "Пароли зашифрованы (SHA-256). Данные хранятся в PostgreSQL.",
        "about_section5_title": "Интерпретация",
        "about_section5": "До 8 л/100 км — экономично\n8-12 л/100 км — средне\nБолее 12 л/100 км — высокий расход",
        "footer": "© 2024 Калькулятор расхода топлива",
        "no_cars_limit": f"Максимум {MAX_CARS_PER_USER} автомобилей",
        "database_error": "Ошибка подключения к базе данных",
        "database_retry": "Попробуйте позже или проверьте настройки PostgreSQL",
        "cancel": "Отмена",
        "all_vehicles": "Все автомобили",
        "all_time": "Все время",
        "days_7": "7 дней",
        "days_30": "30 дней",
        "days_90": "90 дней",
        "days_180": "180 дней",
        "days_365": "365 дней",
    },
    "en": {
        "app_title": "Fuel Calculator",
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
        "password_short": "Password must be at least 6 characters",
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
        "my_cars": f"My vehicles (max. {MAX_CARS_PER_USER})",
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
        "about_section2": f"• Accurate consumption calculation\n• Trip cost calculation\n• Up to {MAX_CARS_PER_USER} vehicles\n• Calculation history\n• PostgreSQL database storage",
        "about_section3_title": "How to use",
        "about_section3": "1. Register\n2. Add a vehicle\n3. Select calculation mode\n4. Enter data\n5. Press Calculate",
        "about_section4_title": "Security",
        "about_section4": "Passwords encrypted (SHA-256). Data stored in PostgreSQL.",
        "about_section5_title": "Interpretation",
        "about_section5": "Up to 8 L/100 km — economical\n8-12 L/100 km — average\nOver 12 L/100 km — high",
        "footer": "© 2024 Fuel Consumption Calculator",
        "no_cars_limit": f"Maximum {MAX_CARS_PER_USER} vehicles",
        "database_error": "Database connection error",
        "database_retry": "Please try again later or check PostgreSQL settings",
        "cancel": "Cancel",
        "all_vehicles": "All vehicles",
        "all_time": "All time",
        "days_7": "7 days",
        "days_30": "30 days",
        "days_90": "90 days",
        "days_180": "180 days",
        "days_365": "365 days",
    }
}

# ─── Currencies ────────────────────────────────────────────────────────────────
CURRENCIES = {
    "₽ RUB": "₽",
    "$ USD": "$",
    "€ EUR": "€",
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
            try:
                draw.text((w // 2 - 30, h // 2 - 8), text, fill=t["fg2"])
            except:
                pass
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


# ─── History Panel Class ────────────────────────────────────────────────────────
class HistoryPanel:
    """Класс для управления панелью истории"""
    
    def __init__(self, parent, app):
        self.parent = parent
        self.app = app
        self.current_page = 1
        self.items_per_page = 10
        self.total_pages = 1
        self.current_history = []
        self.filter_car = None
        self.filter_days = None
        
    def build(self):
        """Построение интерфейса истории"""
        t = T()
        
        # Заголовок
        header_frame = tk.Frame(self.parent, bg=t["bg"])
        header_frame.pack(fill="x", pady=(0, 16))
        
        tk.Label(header_frame, text=TR("history_title"), 
                font=("Georgia", 18, "bold"),
                bg=t["bg"], fg=t["fg"]).pack(side="left")
        
        # Кнопка очистки
        clear_btn = tk.Button(header_frame, text=TR("clear_all"), 
                            font=("Courier", 10),
                            bg=t["red"], fg="white", bd=0, padx=12, pady=6,
                            command=self.clear_all_history)
        clear_btn.pack(side="right")
        
        # Фильтры
        self.filter_frame = tk.Frame(self.parent, bg=t["bg2"], pady=10)
        self.filter_frame.pack(fill="x", pady=(0, 16))
        
        tk.Label(self.filter_frame, text="Фильтр:", 
                font=("Courier", 10, "bold"), bg=t["bg2"], fg=t["fg"]).pack(side="left", padx=10)
        
        tk.Label(self.filter_frame, text="Автомобиль:", 
                font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(side="left", padx=(20, 5))
        
        self.car_filter_var = tk.StringVar(value=TR("all_vehicles"))
        self.car_combo = ttk.Combobox(self.filter_frame, textvariable=self.car_filter_var,
                                     state="readonly", width=20, font=("Courier", 10))
        self.car_combo.pack(side="left", padx=5)
        
        tk.Label(self.filter_frame, text="Период:", 
                font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(side="left", padx=10)
        
        self.days_filter_var = tk.StringVar(value=TR("all_time"))
        days_combo = ttk.Combobox(self.filter_frame, textvariable=self.days_filter_var,
                                 values=[TR("all_time"), TR("days_7"), TR("days_30"), 
                                        TR("days_90"), TR("days_180"), TR("days_365")],
                                 state="readonly", width=12, font=("Courier", 10))
        days_combo.pack(side="left", padx=5)
        
        apply_btn = tk.Button(self.filter_frame, text="Применить", 
                            font=("Courier", 10), bg=t["btn"], fg="white",
                            bd=0, padx=12, pady=4, command=self.apply_filters)
        apply_btn.pack(side="left", padx=10)
        
        # Статистика
        self.stats_frame = tk.Frame(self.parent, bg=t["bg3"])
        self.stats_frame.pack(fill="x", pady=(0, 16))
        
        # Контейнер для записей истории
        self.history_container = tk.Frame(self.parent, bg=t["bg"])
        self.history_container.pack(fill="both", expand=True)
        
        # Пагинация
        self.pagination_frame = tk.Frame(self.parent, bg=t["bg"])
        self.pagination_frame.pack(fill="x", pady=16)
        
        # Загрузка данных
        self.load_cars_for_filter()
        self.refresh()
    
    def load_cars_for_filter(self):
        """Загрузка списка автомобилей для фильтра"""
        cars = [TR("all_vehicles")]
        if self.app.current_user and self.app._db_connected:
            try:
                user_cars = db_get_cars(self.app.current_user)
                cars += [c["name"] for c in user_cars]
            except Exception as e:
                print(f"Error loading cars for filter: {e}")
        
        # Обновляем комбобокс
        self.car_combo['values'] = cars
    
    def apply_filters(self):
        """Применение фильтров"""
        car_filter = self.car_filter_var.get()
        self.filter_car = None if car_filter == TR("all_vehicles") else car_filter
        
        days_filter = self.days_filter_var.get()
        if days_filter != TR("all_time"):
            days_map = {TR("days_7"): 7, TR("days_30"): 30, TR("days_90"): 90, 
                       TR("days_180"): 180, TR("days_365"): 365}
            self.filter_days = days_map.get(days_filter)
        else:
            self.filter_days = None
        
        self.current_page = 1
        self.refresh()
    
    def refresh(self):
        """Обновление истории"""
        if not self.app.current_user or not self.app._db_connected:
            self.show_login_message()
            return
        
        # Загрузка истории
        self.current_history = db_get_history(
            self.app.current_user, 
            self.filter_car, 
            self.filter_days
        )
        
        # Расчет пагинации
        total_items = len(self.current_history)
        self.total_pages = max(1, (total_items + self.items_per_page - 1) // self.items_per_page)
        
        # Отображение статистики
        self.display_stats()
        
        # Отображение записей
        self.display_history_entries()
        
        # Отображение пагинации
        self.display_pagination()
    
    def display_stats(self):
        """Отображение статистики"""
        for w in self.stats_frame.winfo_children():
            w.destroy()
        
        t = T()
        stats = db_get_history_stats(self.app.current_user, self.filter_car)
        
        if stats and stats['total_trips'] > 0:
            # Создаем сетку статистики
            stats_grid = tk.Frame(self.stats_frame, bg=t["bg3"])
            stats_grid.pack(padx=16, pady=12, fill="x")
            
            # Статистические карточки
            stats_items = [
                ("🚗 Всего поездок:", f"{stats['total_trips']}"),
                ("📏 Общий пробег:", f"{stats['total_distance']:.0f} км"),
                ("⛽ Всего топлива:", f"{stats['total_fuel']:.1f} л"),
                ("💧 Средний расход:", f"{stats['avg_consumption']:.1f} л/100км"),
                ("💰 Общая стоимость:", f"{stats['total_cost']:,.2f} {get_currency_symbol()}")
            ]
            
            for i, (label, value) in enumerate(stats_items):
                card = tk.Frame(stats_grid, bg=t["bg2"], padx=12, pady=8)
                card.grid(row=i//3, column=i%3, padx=8, pady=4, sticky="nsew")
                
                tk.Label(card, text=label, font=("Courier", 9), 
                        bg=t["bg2"], fg=t["fg2"]).pack()
                tk.Label(card, text=value, font=("Georgia", 14, "bold"), 
                        bg=t["bg2"], fg=t["accent"]).pack()
        else:
            tk.Label(self.stats_frame, text=TR("history_empty"), 
                    font=("Courier", 12), bg=t["bg3"], fg=t["fg2"]).pack(pady=20)
    
    def display_history_entries(self):
        """Отображение записей истории"""
        for w in self.history_container.winfo_children():
            w.destroy()
        
        if not self.current_history:
            t = T()
            tk.Label(self.history_container, text=TR("history_empty"), 
                    font=("Courier", 12), bg=t["bg"], fg=t["fg2"]).pack(pady=40)
            return
        
        # Расчет отображаемых записей для текущей страницы
        start_idx = (self.current_page - 1) * self.items_per_page
        end_idx = min(start_idx + self.items_per_page, len(self.current_history))
        page_entries = self.current_history[start_idx:end_idx]
        
        # Создаем canvas для скроллинга
        canvas = tk.Canvas(self.history_container, bg=T()["bg"], highlightthickness=0)
        scrollbar = ttk.Scrollbar(self.history_container, orient="vertical", command=canvas.yview)
        scrollable_frame = tk.Frame(canvas, bg=T()["bg"])
        
        scrollable_frame.bind(
            "<Configure>",
            lambda e: canvas.configure(scrollregion=canvas.bbox("all"))
        )
        
        canvas.create_window((0, 0), window=scrollable_frame, anchor="nw")
        canvas.configure(yscrollcommand=scrollbar.set)
        
        # Отображение каждой записи
        for i, entry in enumerate(page_entries):
            self.create_history_card(scrollable_frame, entry, i)
            if i < len(page_entries) - 1:
                tk.Frame(scrollable_frame, bg=T()["border"], height=1).pack(fill="x", pady=8)
        
        canvas.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")
        
        # Привязка скроллинга
        def _on_mousewheel(event):
            canvas.yview_scroll(int(-1*(event.delta/120)), "units")
        
        canvas.bind("<Enter>", lambda e: canvas.bind_all("<MouseWheel>", _on_mousewheel))
        canvas.bind("<Leave>", lambda e: canvas.unbind_all("<MouseWheel>"))
    
    def create_history_card(self, parent, entry, index):
        """Создание карточки записи истории"""
        t = T()
        sym = entry.get('currency', '₽')
        
        card = tk.Frame(parent, bg=t["bg2"], padx=16, pady=12)
        card.pack(fill="x", pady=4)
        
        # Верхняя строка с датой и кнопками
        top_row = tk.Frame(card, bg=t["bg2"])
        top_row.pack(fill="x", pady=(0, 10))
        
        tk.Label(top_row, text=f"📅 {entry['date']}", 
                font=("Courier", 11, "bold"), bg=t["bg2"], fg=t["fg"]).pack(side="left")
        
        # Кнопки действий
        btn_frame = tk.Frame(top_row, bg=t["bg2"])
        btn_frame.pack(side="right")
        
        # Кнопка копирования
        copy_text = self.format_entry_for_copy(entry, sym)
        copy_btn = tk.Button(btn_frame, text="📋 " + TR("copy"), 
                            font=("Courier", 9), bg=t["green"], fg="white",
                            bd=0, padx=8, pady=4, cursor="hand2",
                            command=lambda: self.copy_to_clipboard(copy_text))
        copy_btn.pack(side="left", padx=(0, 5))
        
        # Кнопка удаления
        delete_btn = tk.Button(btn_frame, text="🗑 " + TR("delete"), 
                              font=("Courier", 9), bg=t["red"], fg="white",
                              bd=0, padx=8, pady=4, cursor="hand2",
                              command=lambda: self.delete_entry(entry['id']))
        delete_btn.pack(side="left")
        
        # Основная информация
        content = tk.Frame(card, bg=t["bg2"])
        content.pack(fill="x")
        
        # Левая колонка - информация об автомобиле
        left_col = tk.Frame(content, bg=t["bg2"])
        left_col.pack(side="left", fill="x", expand=True)
        
        tk.Label(left_col, text=f"🚗 {entry.get('car', '—')}", 
                font=("Courier", 11, "bold"), bg=t["bg2"], fg=t["accent"]).pack(anchor="w")
        
        info_text = f"📏 {TR('distance')}: {entry['distance']:.0f} км  |  ⛽ {TR('fuel')}: {entry['fuel']:.1f} л"
        tk.Label(left_col, text=info_text, font=("Courier", 10), 
                bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", pady=(4, 2))
        
        price_text = f"💰 {TR('fuel_price')}: {entry['price']:.2f} {sym}/{TR('fuel_unit')}"
        tk.Label(left_col, text=price_text, font=("Courier", 10), 
                bg=t["bg2"], fg=t["fg2"]).pack(anchor="w")
        
        # Правая колонка - результаты
        right_col = tk.Frame(content, bg=t["bg3"], padx=12, pady=8)
        right_col.pack(side="right")
        
        consumption_text = f"💧 {TR('avg_consumption')}: {entry['consumption']:.1f} л/100км"
        tk.Label(right_col, text=consumption_text, font=("Courier", 11, "bold"), 
                bg=t["bg3"], fg=t["green"]).pack(anchor="w")
        
        cost_text = f"💰 {TR('trip_cost')}: {entry['cost']:,.2f} {sym}"
        tk.Label(right_col, text=cost_text, font=("Courier", 11, "bold"), 
                bg=t["bg3"], fg=t["yellow"]).pack(anchor="w")
    
    def format_entry_for_copy(self, entry, sym):
        """Форматирование записи для копирования"""
        return (
            f"📅 Дата: {entry['date']}\n"
            f"🚗 Автомобиль: {entry.get('car', '—')}\n"
            f"📏 Расстояние: {entry['distance']:.0f} км\n"
            f"⛽ Топливо: {entry['fuel']:.1f} л\n"
            f"💰 Цена: {entry['price']:.2f} {sym}/л\n"
            f"💧 Расход: {entry['consumption']:.1f} л/100км\n"
            f"💰 Стоимость: {entry['cost']:,.2f} {sym}"
        )
    
    def copy_to_clipboard(self, text):
        """Копирование в буфер обмена"""
        self.app.clipboard_clear()
        self.app.clipboard_append(text)
        messagebox.showinfo(TR("copied"), TR("copied_msg"))
    
    def delete_entry(self, entry_id):
        """Удаление записи из истории"""
        if messagebox.askyesno(TR("delete_confirm"), "Удалить эту запись?"):
            if db_delete_history_entry(self.app.current_user, entry_id):
                self.refresh()
                messagebox.showinfo("Успех", "Запись удалена")
            else:
                messagebox.showerror(TR("error"), "Ошибка при удалении")
    
    def clear_all_history(self):
        """Очистка всей истории"""
        if messagebox.askyesno(TR("clear_history_confirm"), TR("clear_history_msg")):
            if db_clear_history(self.app.current_user):
                self.refresh()
                messagebox.showinfo("Успех", "История очищена")
            else:
                messagebox.showerror(TR("error"), "Ошибка при очистке")
    
    def display_pagination(self):
        """Отображение пагинации"""
        for w in self.pagination_frame.winfo_children():
            w.destroy()
        
        if self.total_pages <= 1:
            return
        
        t = T()
        
        # Кнопка "Первая"
        if self.current_page > 1:
            first_btn = tk.Button(self.pagination_frame, text="«", 
                                 font=("Courier", 10), bg=t["btn2"], fg=t["fg"],
                                 bd=0, padx=10, pady=4, command=self.first_page)
            first_btn.pack(side="left", padx=2)
        
        # Кнопка "Предыдущая"
        if self.current_page > 1:
            prev_btn = tk.Button(self.pagination_frame, text="‹", 
                                font=("Courier", 10), bg=t["btn2"], fg=t["fg"],
                                bd=0, padx=10, pady=4, command=self.prev_page)
            prev_btn.pack(side="left", padx=2)
        
        # Номера страниц
        start_page = max(1, self.current_page - 2)
        end_page = min(self.total_pages, start_page + 4)
        
        for page in range(start_page, end_page + 1):
            if page == self.current_page:
                btn = tk.Button(self.pagination_frame, text=str(page), 
                              font=("Courier", 10, "bold"), bg=t["btn"], fg="white",
                              bd=0, padx=12, pady=4)
            else:
                btn = tk.Button(self.pagination_frame, text=str(page), 
                              font=("Courier", 10), bg=t["btn2"], fg=t["fg"],
                              bd=0, padx=12, pady=4, 
                              command=lambda p=page: self.go_to_page(p))
            btn.pack(side="left", padx=2)
        
        # Кнопка "Следующая"
        if self.current_page < self.total_pages:
            next_btn = tk.Button(self.pagination_frame, text="›", 
                                font=("Courier", 10), bg=t["btn2"], fg=t["fg"],
                                bd=0, padx=10, pady=4, command=self.next_page)
            next_btn.pack(side="left", padx=2)
        
        # Кнопка "Последняя"
        if self.current_page < self.total_pages:
            last_btn = tk.Button(self.pagination_frame, text="»", 
                                font=("Courier", 10), bg=t["btn2"], fg=t["fg"],
                                bd=0, padx=10, pady=4, command=self.last_page)
            last_btn.pack(side="left", padx=2)
        
        # Информация о странице
        info_text = f"Страница {self.current_page} из {self.total_pages} (всего {len(self.current_history)} записей)"
        tk.Label(self.pagination_frame, text=info_text, font=("Courier", 9), 
                bg=t["bg"], fg=t["fg2"]).pack(side="right", padx=10)
    
    def first_page(self):
        """Переход на первую страницу"""
        self.current_page = 1
        self.refresh()
    
    def prev_page(self):
        """Предыдущая страница"""
        if self.current_page > 1:
            self.current_page -= 1
            self.refresh()
    
    def next_page(self):
        """Следующая страница"""
        if self.current_page < self.total_pages:
            self.current_page += 1
            self.refresh()
    
    def last_page(self):
        """Последняя страница"""
        self.current_page = self.total_pages
        self.refresh()
    
    def go_to_page(self, page):
        """Переход на определенную страницу"""
        self.current_page = page
        self.refresh()
    
    def show_login_message(self):
        """Показать сообщение о необходимости входа"""
        t = T()
        for w in self.history_container.winfo_children():
            w.destroy()
        tk.Label(self.history_container, text=TR("login_for_history"), 
                font=("Courier", 14), bg=t["bg"], fg=t["fg2"]).pack(pady=50)


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
        self._apply_current_theme()

    def _apply_current_theme(self):
        """Apply current theme to all UI elements"""
        t = T()
        self.configure(bg=t["bg"])
        if hasattr(self, 'header'):
            self.header.configure(bg=t["bg2"])
        if hasattr(self, 'body'):
            self.body.configure(bg=t["bg"])
        if hasattr(self, 'sidebar'):
            self.sidebar.configure(bg=t["bg2"])
        if hasattr(self, 'content'):
            self.content.configure(bg=t["bg"])
        
        for section in self.sections.values():
            section.configure(bg=t["bg"])

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
        if hasattr(self, 'header'):
            self.header.destroy()
        if hasattr(self, 'body'):
            self.body.destroy()
        self._photo_refs.clear()
        self._build_ui()

    # ── Layout ──
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
        if name == "history" and hasattr(self, 'history_panel'):
            self.history_panel.refresh()
        if name == "profile":
            self._refresh_profile()
        if name == "settings":
            self._refresh_settings_display()

    # ── Calculator (упрощенная версия) ──
    def _build_calculator(self):
        t = T()
        f = self.sections["calculator"]
        for w in f.winfo_children():
            w.destroy()
        
        # Простой интерфейс калькулятора
        main_frame = tk.Frame(f, bg=t["bg"])
        main_frame.pack(fill="both", expand=True, padx=50, pady=50)
        
        tk.Label(main_frame, text="Калькулятор расхода топлива", 
                font=("Georgia", 20, "bold"), bg=t["bg"], fg=t["fg"]).pack(pady=20)
        
        # Форма ввода
        form_frame = tk.Frame(main_frame, bg=t["bg2"], padx=30, pady=30)
        form_frame.pack(pady=20)
        
        # Расстояние
        tk.Label(form_frame, text="Расстояние (км):", font=("Courier", 12), 
                bg=t["bg2"], fg=t["fg"]).grid(row=0, column=0, pady=10, sticky="w")
        self.dist_entry = tk.Entry(form_frame, font=("Courier", 12), width=20)
        self.dist_entry.grid(row=0, column=1, pady=10, padx=10)
        
        # Топливо
        tk.Label(form_frame, text="Топливо (л):", font=("Courier", 12), 
                bg=t["bg2"], fg=t["fg"]).grid(row=1, column=0, pady=10, sticky="w")
        self.fuel_entry = tk.Entry(form_frame, font=("Courier", 12), width=20)
        self.fuel_entry.grid(row=1, column=1, pady=10, padx=10)
        
        # Цена
        tk.Label(form_frame, text="Цена за литр:", font=("Courier", 12), 
                bg=t["bg2"], fg=t["fg"]).grid(row=2, column=0, pady=10, sticky="w")
        self.price_entry = tk.Entry(form_frame, font=("Courier", 12), width=20)
        self.price_entry.grid(row=2, column=1, pady=10, padx=10)
        
        # Кнопки
        btn_frame = tk.Frame(form_frame, bg=t["bg2"])
        btn_frame.grid(row=3, column=0, columnspan=2, pady=20)
        
        calc_btn = tk.Button(btn_frame, text="Рассчитать", font=("Georgia", 12, "bold"),
                           bg=t["btn"], fg="white", padx=20, pady=10, command=self._calculate_simple)
        calc_btn.pack(side="left", padx=10)
        
        clear_btn = tk.Button(btn_frame, text="Очистить", font=("Georgia", 12),
                            bg=t["btn2"], fg=t["fg"], padx=20, pady=10, command=self._clear_simple)
        clear_btn.pack(side="left", padx=10)
        
        # Результат
        self.result_frame = tk.Frame(main_frame, bg=t["bg3"], padx=20, pady=20)
        self.result_frame.pack(pady=20, fill="x")
        
        self.result_label = tk.Label(self.result_frame, text="", font=("Georgia", 14), 
                                    bg=t["bg3"], fg=t["green"])
        self.result_label.pack()
    
    def _calculate_simple(self):
        try:
            distance = float(self.dist_entry.get())
            fuel = float(self.fuel_entry.get())
            price = float(self.price_entry.get())
            
            if distance <= 0:
                messagebox.showerror("Ошибка", "Расстояние должно быть больше 0")
                return
            
            consumption = (fuel / distance) * 100
            cost = fuel * price
            
            result_text = f"Расход: {consumption:.2f} л/100 км\nСтоимость: {cost:.2f} {get_currency_symbol()}"
            self.result_label.configure(text=result_text)
            
            # Сохраняем в историю
            if self.current_user and self._db_connected:
                entry = {
                    "date": datetime.now().strftime("%d.%m.%Y %H:%M"),
                    "car": TR("no_car"),
                    "distance": distance,
                    "fuel": round(fuel, 2),
                    "price": price,
                    "currency": get_currency_symbol(),
                    "consumption": round(consumption, 2),
                    "cost": round(cost, 2),
                }
                db_add_history(self.current_user, entry)
                if hasattr(self, 'history_panel'):
                    self.history_panel.refresh()
                
        except ValueError:
            messagebox.showerror("Ошибка", "Введите корректные числа")
    
    def _clear_simple(self):
        self.dist_entry.delete(0, tk.END)
        self.fuel_entry.delete(0, tk.END)
        self.price_entry.delete(0, tk.END)
        self.result_label.configure(text="")

    # ── Profile (упрощенная версия) ──
    def _build_profile(self):
        t = T()
        f = self.sections["profile"]
        for w in f.winfo_children():
            w.destroy()
        
        main_frame = tk.Frame(f, bg=t["bg"])
        main_frame.pack(fill="both", expand=True, padx=50, pady=50)
        
        if not self._db_connected:
            tk.Label(main_frame, text=TR("database_error"), font=("Georgia", 14, "bold"),
                    bg=t["bg"], fg=t["red"]).pack(pady=20)
            return
        
        if not self.current_user:
            self._build_login_form(main_frame)
        else:
            self._build_logged_in_profile(main_frame)
    
    def _build_login_form(self, parent):
        t = T()
        
        tk.Label(parent, text="Вход в аккаунт", font=("Georgia", 20, "bold"),
                bg=t["bg"], fg=t["fg"]).pack(pady=20)
        
        form_frame = tk.Frame(parent, bg=t["bg2"], padx=30, pady=30)
        form_frame.pack(pady=20)
        
        tk.Label(form_frame, text="Email:", font=("Courier", 12), 
                bg=t["bg2"], fg=t["fg"]).grid(row=0, column=0, pady=10, sticky="w")
        self.login_email = tk.Entry(form_frame, font=("Courier", 12), width=25)
        self.login_email.grid(row=0, column=1, pady=10, padx=10)
        
        tk.Label(form_frame, text="Пароль:", font=("Courier", 12), 
                bg=t["bg2"], fg=t["fg"]).grid(row=1, column=0, pady=10, sticky="w")
        self.login_password = tk.Entry(form_frame, font=("Courier", 12), show="•", width=25)
        self.login_password.grid(row=1, column=1, pady=10, padx=10)
        
        btn_frame = tk.Frame(form_frame, bg=t["bg2"])
        btn_frame.grid(row=2, column=0, columnspan=2, pady=20)
        
        login_btn = tk.Button(btn_frame, text="Войти", font=("Georgia", 12, "bold"),
                            bg=t["btn"], fg="white", padx=20, pady=10, command=self._do_login)
        login_btn.pack(side="left", padx=10)
        
        register_btn = tk.Button(btn_frame, text="Регистрация", font=("Georgia", 12),
                               bg=t["btn2"], fg=t["fg"], padx=20, pady=10, command=self._show_register)
        register_btn.pack(side="left", padx=10)
    
    def _do_login(self):
        email = self.login_email.get().strip().lower()
        password = self.login_password.get()
        
        if not email or not password:
            messagebox.showerror("Ошибка", "Заполните все поля")
            return
        
        try:
            stored_pw = db_get_user(email)
            if stored_pw and stored_pw == hash_pw(password):
                self.current_user = email
                self._load_user_settings()
                self._refresh_profile()
                self._apply_current_theme()
                if hasattr(self, 'history_panel'):
                    self.history_panel.load_cars_for_filter()
                    self.history_panel.refresh()
                messagebox.showinfo("Успех", f"Добро пожаловать, {email}!")
            else:
                messagebox.showerror("Ошибка", "Неверный email или пароль")
        except Exception as e:
            messagebox.showerror("Ошибка", f"Ошибка входа: {e}")
    
    def _show_register(self):
        # Упрощенная регистрация
        register_window = tk.Toplevel(self)
        register_window.title("Регистрация")
        register_window.geometry("400x350")
        register_window.configure(bg=T()["bg2"])
        
        tk.Label(register_window, text="Регистрация", font=("Georgia", 16, "bold"),
                bg=T()["bg2"], fg=T()["fg"]).pack(pady=20)
        
        form_frame = tk.Frame(register_window, bg=T()["bg2"], padx=20, pady=20)
        form_frame.pack()
        
        tk.Label(form_frame, text="Email:", font=("Courier", 11), 
                bg=T()["bg2"], fg=T()["fg"]).grid(row=0, column=0, pady=10, sticky="w")
        reg_email = tk.Entry(form_frame, font=("Courier", 11), width=25)
        reg_email.grid(row=0, column=1, pady=10, padx=10)
        
        tk.Label(form_frame, text="Пароль:", font=("Courier", 11), 
                bg=T()["bg2"], fg=T()["fg"]).grid(row=1, column=0, pady=10, sticky="w")
        reg_password = tk.Entry(form_frame, font=("Courier", 11), show="•", width=25)
        reg_password.grid(row=1, column=1, pady=10, padx=10)
        
        tk.Label(form_frame, text="Повторите пароль:", font=("Courier", 11), 
                bg=T()["bg2"], fg=T()["fg"]).grid(row=2, column=0, pady=10, sticky="w")
        reg_password2 = tk.Entry(form_frame, font=("Courier", 11), show="•", width=25)
        reg_password2.grid(row=2, column=1, pady=10, padx=10)
        
        def do_register():
            email = reg_email.get().strip().lower()
            pw = reg_password.get()
            pw2 = reg_password2.get()
            
            if not email or not pw:
                messagebox.showerror("Ошибка", "Заполните все поля")
                return
            
            if not is_valid_email(email):
                messagebox.showerror("Ошибка", "Неверный email")
                return
            
            if len(pw) < 6:
                messagebox.showerror("Ошибка", "Пароль должен быть не менее 6 символов")
                return
            
            if pw != pw2:
                messagebox.showerror("Ошибка", "Пароли не совпадают")
                return
            
            try:
                if db_user_exists(email):
                    messagebox.showerror("Ошибка", "Пользователь уже существует")
                    return
                
                db_create_user(email, hash_pw(pw))
                messagebox.showinfo("Успех", "Регистрация успешна! Теперь войдите.")
                register_window.destroy()
            except Exception as e:
                messagebox.showerror("Ошибка", f"Ошибка регистрации: {e}")
        
        reg_btn = tk.Button(register_window, text="Зарегистрироваться", font=("Georgia", 11, "bold"),
                          bg=T()["btn"], fg="white", padx=20, pady=10, command=do_register)
        reg_btn.pack(pady=20)
    
    def _build_logged_in_profile(self, parent):
        t = T()
        
        tk.Label(parent, text=f"Добро пожаловать, {self.current_user}!", 
                font=("Georgia", 18, "bold"), bg=t["bg"], fg=t["fg"]).pack(pady=20)
        
        logout_btn = tk.Button(parent, text="Выйти", font=("Georgia", 12),
                              bg=t["red"], fg="white", padx=20, pady=10,
                              command=self._logout)
        logout_btn.pack(pady=10)
        
        delete_btn = tk.Button(parent, text="Удалить аккаунт", font=("Georgia", 12),
                              bg=t["red"], fg="white", padx=20, pady=10,
                              command=self._delete_account)
        delete_btn.pack(pady=10)
    
    def _logout(self):
        self.current_user = None
        self._refresh_profile()
        if hasattr(self, 'history_panel'):
            self.history_panel.refresh()
    
    def _delete_account(self):
        if messagebox.askyesno("Удаление аккаунта", "Вы уверены? Все данные будут удалены."):
            try:
                db_delete_user(self.current_user)
                self.current_user = None
                self._refresh_profile()
                if hasattr(self, 'history_panel'):
                    self.history_panel.refresh()
                messagebox.showinfo("Успех", "Аккаунт удален")
            except Exception as e:
                messagebox.showerror("Ошибка", f"Ошибка при удалении: {e}")
    
    def _refresh_profile(self):
        f = self.sections["profile"]
        for w in f.winfo_children():
            w.destroy()
        self._build_profile()
    
    # ── History ──
    def _build_history(self):
        """Построение раздела истории"""
        t = T()
        f = self.sections["history"]
        for w in f.winfo_children():
            w.destroy()
        
        # Создаем скроллируемую область
        outer, inner = make_scrollable(f, t["bg"])
        outer.pack(fill="both", expand=True)
        
        # Контейнер для истории
        history_container = tk.Frame(inner, bg=t["bg"])
        history_container.pack(fill="both", expand=True, padx=24, pady=20)
        
        # Создаем панель истории
        self.history_panel = HistoryPanel(history_container, self)
        self.history_panel.build()
    
    # ── About ──
    def _build_about(self):
        t = T()
        f = self.sections["about"]
        for w in f.winfo_children():
            w.destroy()
        
        main_frame = tk.Frame(f, bg=t["bg"])
        main_frame.pack(fill="both", expand=True, padx=50, pady=50)
        
        tk.Label(main_frame, text="⛽", font=("Courier", 48), 
                bg=t["bg"], fg=t["accent"]).pack()
        tk.Label(main_frame, text=TR("app_title"), font=("Georgia", 22, "bold"),
                bg=t["bg"], fg=t["fg"]).pack()
        tk.Label(main_frame, text=TR("version"), font=("Courier", 10),
                bg=t["bg"], fg=t["fg2"]).pack(pady=4)
        tk.Label(main_frame, text=TR("about_section1"), font=("Courier", 11),
                bg=t["bg"], fg=t["fg2"], wraplength=600).pack(pady=20)
        tk.Label(main_frame, text=TR("footer"), font=("Courier", 9),
                bg=t["bg"], fg=t["fg2"]).pack(pady=16)
    
    # ── Settings ──
    def _build_settings(self):
        t = T()
        f = self.sections["settings"]
        for w in f.winfo_children():
            w.destroy()
        
        self.settings_pad = tk.Frame(f, bg=t["bg"])
        self.settings_pad.pack(fill="both", expand=True, padx=50, pady=50)
        self._refresh_settings_display()
    
    def _refresh_settings_display(self):
        t = T()
        for w in self.settings_pad.winfo_children():
            w.destroy()
        
        tk.Label(self.settings_pad, text=TR("settings_title"), font=("Georgia", 18, "bold"),
                bg=t["bg"], fg=t["fg"]).pack(anchor="w", pady=(0, 24))
        
        # Theme
        theme_frame = tk.Frame(self.settings_pad, bg=t["bg2"], pady=16)
        theme_frame.pack(fill="x", pady=8)
        tk.Label(theme_frame, text=TR("theme"), font=("Georgia", 13, "bold"),
                bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=20)
        
        theme_buttons = tk.Frame(theme_frame, bg=t["bg2"])
        theme_buttons.pack(padx=20, pady=12, anchor="w")
        
        def apply_theme(name):
            global current_theme
            current_theme = name
            if self.current_user:
                self._save_settings()
            self._rebuild_ui()
            self._show_section("settings")
        
        tk.Button(theme_buttons, text=TR("dark_theme"), font=("Georgia", 11),
                 bg="#1f2937", fg="#e5e7eb", padx=20, pady=8,
                 command=lambda: apply_theme("dark")).pack(side="left", padx=(0, 12))
        tk.Button(theme_buttons, text=TR("light_theme"), font=("Georgia", 11),
                 bg="#e2e8f0", fg="#1a202c", padx=20, pady=8,
                 command=lambda: apply_theme("light")).pack(side="left")
        
        # Currency
        currency_frame = tk.Frame(self.settings_pad, bg=t["bg2"], pady=16)
        currency_frame.pack(fill="x", pady=8)
        tk.Label(currency_frame, text=TR("currency_setting"), font=("Georgia", 13, "bold"),
                bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=20)
        
        cv = tk.StringVar(value=current_currency)
        ccombo = ttk.Combobox(currency_frame, textvariable=cv, values=list(CURRENCIES.keys()),
                             state="readonly", width=14, font=("Courier", 11))
        ccombo.pack(anchor="w", padx=20, pady=8)
        
        def apply_currency(event=None):
            global current_currency
            current_currency = cv.get()
            if self.current_user:
                self._save_settings()
        
        ccombo.bind("<<ComboboxSelected>>", apply_currency)
        
        # Language
        lang_frame = tk.Frame(self.settings_pad, bg=t["bg2"], pady=16)
        lang_frame.pack(fill="x", pady=8)
        tk.Label(lang_frame, text=TR("language_setting"), font=("Georgia", 13, "bold"),
                bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=20)
        
        lang_buttons = tk.Frame(lang_frame, bg=t["bg2"])
        lang_buttons.pack(padx=20, pady=12, anchor="w")
        
        def apply_language(lang):
            global current_language
            current_language = lang
            if self.current_user:
                self._save_settings()
            self._rebuild_ui()
            self._show_section("settings")
        
        tk.Button(lang_buttons, text="🇷🇺 Русский", font=("Georgia", 11),
                 bg=t["btn"] if current_language == "ru" else t["btn2"],
                 fg="white" if current_language == "ru" else t["fg"],
                 padx=20, pady=8,
                 command=lambda: apply_language("ru")).pack(side="left", padx=(0, 12))
        tk.Button(lang_buttons, text="🇬🇧 English", font=("Georgia", 11),
                 bg=t["btn"] if current_language == "en" else t["btn2"],
                 fg="white" if current_language == "en" else t["fg"],
                 padx=20, pady=8,
                 command=lambda: apply_language("en")).pack(side="left")


if __name__ == "__main__":
    app = FuelApp()
    app.mainloop()
