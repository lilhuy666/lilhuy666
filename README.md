import os
import hashlib
import re
from datetime import datetime
from contextlib import contextmanager
from flask import Flask, render_template, request, redirect, url_for, session, flash, jsonify
from werkzeug.utils import secure_filename
import psycopg2
from psycopg2 import pool
from jinja2 import DictLoader

# ---------- Настройки БД ----------
DB_CONFIG = {
    'dbname': 'fuel_calc',
    'user': 'postgres',
    'password': '12345',
    'host': 'localhost',
    'port': 5432
}

connection_pool = None

def init_db_pool():
    global connection_pool
    try:
        connection_pool = pool.SimpleConnectionPool(1, 20, **DB_CONFIG)
        return True
    except Exception as e:
        print(f"DB pool error: {e}")
        return False

@contextmanager
def get_db_connection():
    conn = connection_pool.getconn()
    try:
        yield conn
    finally:
        connection_pool.putconn(conn)

def create_tables():
    with psycopg2.connect(**DB_CONFIG) as conn:
        with conn.cursor() as cur:
            cur.execute("""
                CREATE TABLE IF NOT EXISTS users (
                    email TEXT PRIMARY KEY,
                    password TEXT NOT NULL
                );
                CREATE TABLE IF NOT EXISTS cars (
                    id SERIAL PRIMARY KEY,
                    email TEXT REFERENCES users(email) ON DELETE CASCADE,
                    name TEXT NOT NULL,
                    photo_path TEXT,
                    avg_consumption FLOAT,
                    UNIQUE(email, name)
                );
                CREATE TABLE IF NOT EXISTS history (
                    id SERIAL PRIMARY KEY,
                    email TEXT REFERENCES users(email) ON DELETE CASCADE,
                    car_name TEXT,
                    date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    distance FLOAT,
                    fuel FLOAT,
                    price FLOAT,
                    currency TEXT,
                    consumption FLOAT,
                    cost FLOAT
                );
                CREATE TABLE IF NOT EXISTS settings (
                    email TEXT PRIMARY KEY REFERENCES users(email) ON DELETE CASCADE,
                    theme TEXT DEFAULT 'light',
                    language TEXT DEFAULT 'ru',
                    currency TEXT DEFAULT '₽ RUB'
                );
            """)
            conn.commit()

# ----- CRUD функции -----
def db_create_user(email, password_hash):
    with get_db_connection() as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO users (email, password) VALUES (%s, %s)", (email, password_hash))
            cur.execute("INSERT INTO settings (email, theme, language, currency) VALUES (%s, 'light', 'ru', '₽ RUB')", (email,))
        conn.commit()

def db_get_user(email):
    with get_db_connection() as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT password FROM users WHERE email = %s", (email,))
            row = cur.fetchone()
            return row[0] if row else None

def db_user_exists(email):
    with get_db_connection() as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT 1 FROM users WHERE email = %s", (email,))
            return cur.fetchone() is not None

def db_update_password(email, new_hash):
    with get_db_connection() as conn:
        with conn.cursor() as cur:
            cur.execute("UPDATE users SET password = %s WHERE email = %s", (new_hash, email))
        conn.commit()

def db_delete_user(email):
    with get_db_connection() as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT photo_path FROM cars WHERE email = %s", (email,))
            for row in cur.fetchall():
                if row[0]:
                    file_path = os.path.join('static', 'uploads', row[0])
                    if os.path.exists(file_path):
                        try:
                            os.remove(file_path)
                        except:
                            pass
            cur.execute("DELETE FROM users WHERE email = %s", (email,))
        conn.commit()

def db_get_cars(email):
    with get_db_connection() as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT name, photo_path, avg_consumption FROM cars WHERE email = %s ORDER BY id", (email,))
            return [{"name": r[0], "photo": r[1], "avg_consumption": r[2]} for r in cur.fetchall()]

def db_add_car(email, name, photo_name=None):
    with get_db_connection() as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO cars (email, name, photo_path) VALUES (%s, %s, %s)", (email, name, photo_name))
        conn.commit()

def db_delete_car(email, name):
    with get_db_connection() as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT photo_path FROM cars WHERE email = %s AND name = %s", (email, name))
            row = cur.fetchone()
            if row and row[0]:
                file_path = os.path.join('static', 'uploads', row[0])
                if os.path.exists(file_path):
                    try:
                        os.remove(file_path)
                    except:
                        pass
            cur.execute("DELETE FROM cars WHERE email = %s AND name = %s", (email, name))
        conn.commit()

def db_update_car_consumption(email, car_name, consumption):
    with get_db_connection() as conn:
        with conn.cursor() as cur:
            cur.execute("UPDATE cars SET avg_consumption = %s WHERE email = %s AND name = %s", (consumption, email, car_name))
        conn.commit()

def db_add_history(email, entry):
    with get_db_connection() as conn:
        with conn.cursor() as cur:
            cur.execute("""
                INSERT INTO history (email, car_name, distance, fuel, price, currency, consumption, cost, date)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
            """, (email, entry.get('car'), entry['distance'], entry['fuel'],
                  entry['price'], entry.get('currency'), entry['consumption'],
                  entry['cost'], datetime.strptime(entry['date'], "%d.%m.%Y %H:%M")))
        conn.commit()

def db_get_history(email, car_name=None, days=None):
    with get_db_connection() as conn:
        with conn.cursor() as cur:
            query = """
                SELECT h.id, h.car_name, h.distance, h.fuel, h.price, h.currency,
                       h.consumption, h.cost,
                       TO_CHAR(h.date, 'DD.MM.YYYY HH24:MI') as date_str,
                       c.photo_path
                FROM history h
                LEFT JOIN cars c ON h.email = c.email AND h.car_name = c.name
                WHERE h.email = %s
            """
            params = [email]
            if car_name and car_name not in ("— Без авто —", "— No car —"):
                query += " AND h.car_name = %s"
                params.append(car_name)
            if days:
                query += " AND h.date >= CURRENT_DATE - %s * INTERVAL '1 day'"
                params.append(days)
            query += " ORDER BY h.date DESC"
            cur.execute(query, params)
            results = []
            for r in cur.fetchall():
                results.append({
                    'id': r[0], 'car': r[1], 'distance': r[2], 'fuel': r[3],
                    'price': r[4], 'currency': r[5], 'consumption': r[6], 'cost': r[7],
                    'date': r[8], 'photo': r[9]
                })
            return results

def db_clear_history(email):
    with get_db_connection() as conn:
        with conn.cursor() as cur:
            cur.execute("DELETE FROM history WHERE email = %s", (email,))
        conn.commit()

def db_delete_history_entry(email, entry_id):
    with get_db_connection() as conn:
        with conn.cursor() as cur:
            cur.execute("DELETE FROM history WHERE email = %s AND id = %s", (email, entry_id))
        conn.commit()

def db_get_settings(email):
    with get_db_connection() as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT theme, language, currency FROM settings WHERE email = %s", (email,))
            row = cur.fetchone()
    if row:
        return {'theme': row[0], 'language': row[1], 'currency': row[2]}
    return {'theme': 'light', 'language': 'ru', 'currency': '₽ RUB'}

def db_update_settings(email, theme, language, currency):
    with get_db_connection() as conn:
        with conn.cursor() as cur:
            cur.execute("UPDATE settings SET theme=%s, language=%s, currency=%s WHERE email=%s",
                        (theme, language, currency, email))
        conn.commit()

def db_count_cars(email):
    with get_db_connection() as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT COUNT(*) FROM cars WHERE email = %s", (email,))
            return cur.fetchone()[0]

def db_get_history_for_chart(email, car_name, days):
    with get_db_connection() as conn:
        with conn.cursor() as cur:
            query = """
                SELECT DATE(date) as day, AVG(consumption)
                FROM history
                WHERE email = %s AND date >= CURRENT_DATE - %s * INTERVAL '1 day'
            """
            params = [email, days]
            if car_name and car_name not in ("— Без авто —", "— No car —"):
                query += " AND car_name = %s"
                params.append(car_name)
            query += " GROUP BY DATE(date) ORDER BY day"
            cur.execute(query, params)
            return [(r[0], float(r[1])) for r in cur.fetchall() if r[0] and r[1] is not None]

def db_get_car_stats(email, car_name):
    with get_db_connection() as conn:
        with conn.cursor() as cur:
            cur.execute("""
                SELECT SUM(distance), SUM(cost), AVG(consumption)
                FROM history
                WHERE email = %s AND car_name = %s
            """, (email, car_name))
            row = cur.fetchone()
            return {
                'total_distance': round(row[0], 1) if row[0] else 0,
                'total_cost': round(row[1], 2) if row[1] else 0,
                'avg_consumption': round(row[2], 2) if row[2] else 0
            }

# ---------- Переводы и темы ----------
MAX_CARS_PER_USER = 25

TRANSLATIONS = {
    "ru": {
        "app_title": "FullCalcPro",
        "calculator": "Калькулятор",
        "profile": "Профиль",
        "login_register": "Вход / Регистрация",
        "history": "История",
        "about": "О приложении",
        "settings": "Настройки",
        "back": "← Назад",
        "no_car": "-- Без авто --",
        "car": "Автомобиль",
        "quick_stats": "Статистика",
        "avg_consumption": "Средний расход",
        "distance": "Расстояние",
        "fuel": "Топливо",
        "cost": "Стоимость",
        "calculate": "Рассчитать",
        "clear": "Очистить",
        "fuel_consumption": "Расход топлива",
        "trip_cost": "Стоимость поездки",
        "distance_label": "Расстояние",
        "fuel_used": "Израсходовано топлива",
        "fuel_price": "Цена топлива за литр",
        "avg_consumption_label": "Средний расход",
        "calc_mode_consumption": "Расход на 100 км",
        "calc_mode_cost": "Стоимость поездки",
        "login": "Вход",
        "register": "Регистрация",
        "create_account": "Создать аккаунт",
        "login_account": "Войти",
        "email": "Email",
        "password": "Пароль",
        "repeat_password": "Повторите пароль",
        "fill_all": "Заполните все поля",
        "invalid_email": "Некорректный email",
        "password_short": "Пароль слишком короткий (мин. 6 символов)",
        "passwords_mismatch": "Пароли не совпадают",
        "user_exists": "Пользователь уже существует",
        "wrong_credentials": "Неверный email или пароль",
        "logout": "Выйти",
        "delete_account": "Удалить аккаунт",
        "change_password": "Сменить пароль",
        "old_password": "Старый пароль",
        "new_password": "Новый пароль",
        "repeat_new_password": "Повторите новый пароль",
        "change_password_btn": "Сменить пароль",
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
        "copied_msg": "Расчёт скопирован!",
        "delete_entry": "Удалить запись",
        "delete_confirm": "Удалить?",
        "delete_car_confirm": "Удалить этот автомобиль?",
        "delete_history_confirm": "Удалить запись?",
        "delete_history_msg": "Вы уверены, что хотите удалить эту запись?",
        "clear_history_confirm": "Очистить историю?",
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
        "language_setting": "Язык",
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
        "version": "Версия 1.0  •  2026",
        "about_section1_title": "О приложении",
        "about_section1": "Профессиональное приложение для расчёта расхода топлива с использованием PostgreSQL.",
        "about_section2_title": "Возможности",
        "about_section2": (f"• Точный расчёт расхода топлива\n"
                           f"• Расчёт стоимости поездки\n"
                           f"• Хранение до {MAX_CARS_PER_USER} автомобилей\n"
                           f"• История расчётов\n"
                           f"• Интерактивные графики\n"
                           f"• Поддержка нескольких валют и тем оформления"),
        "about_section3_title": "Как использовать",
        "about_section3": ("1. Зарегистрируйтесь или войдите в аккаунт\n"
                           "2. Добавьте автомобиль с фото\n"
                           "3. Выберите режим расчёта\n"
                           "4. Введите данные поездки\n"
                           "5. Нажмите «Рассчитать» — результат сразу сохранится в истории"),
        "about_section4_title": "Безопасность",
        "about_section4": "Пароли шифруются с использованием современных алгоритмов. Ваши данные надёжно защищены в базе данных PostgreSQL.",
        "about_section5_title": "Интерпретация расхода",
        "about_section5": ("До 8 л/100 км — отличная экономичность\n"
                           "8–12 л/100 км — средний расход\n"
                           "Более 12 л/100 км — высокий расход, возможно, требуется диагностика"),
        "about_section6_title": "Советы по экономии",
        "about_section6": ("• Плавный разгон и торможение\n"
                           "• Поддерживайте давление в шинах\n"
                           "• Убирайте лишний груз\n"
                           "• Регулярное техническое обслуживание\n"
                           "• Используйте качественное топливо"),
        "footer": "© 2026 FullCalcPro",
        "no_cars_limit": f"Максимум {MAX_CARS_PER_USER} автомобилей",
        "database_error": "Ошибка подключения к базе данных",
        "database_retry": "Попробуйте позже или проверьте настройки PostgreSQL",
        "cancel": "Отмена",
        "old_password_wrong": "Неверный старый пароль",
        "date": "Дата",
        "show_more": "Показать ещё",
        "show_less": "Свернуть",
        "password_changed": "Пароль успешно изменён",
        "clear_history_btn": "Очистить всю историю",
        "car_stats_title": "Статистика автомобиля",
        "total_distance": "Общий пробег",
        "total_cost": "Общие затраты",
        "km": "км",
        "no_data": "Нет данных"
    },
    "en": {
        "app_title": "FullCalcPro",
        "calculator": "Calculator",
        "profile": "Profile",
        "login_register": "Login / Register",
        "history": "History",
        "about": "About",
        "settings": "Settings",
        "back": "← Back",
        "no_car": "-- No car --",
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
        "calc_mode_consumption": "Consumption per 100 km",
        "calc_mode_cost": "Trip cost",
        "login": "Login",
        "register": "Register",
        "create_account": "Create Account",
        "login_account": "Sign In",
        "email": "Email",
        "password": "Password",
        "repeat_password": "Repeat password",
        "fill_all": "Fill in all fields",
        "invalid_email": "Enter a valid email",
        "password_short": "Password is too short (min. 6 characters)",
        "passwords_mismatch": "Passwords do not match",
        "user_exists": "User already exists",
        "wrong_credentials": "Invalid email or password",
        "logout": "Logout",
        "delete_account": "Delete account",
        "change_password": "Change password",
        "old_password": "Current password",
        "new_password": "New password",
        "repeat_new_password": "Repeat new password",
        "change_password_btn": "Change password",
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
        "copied_msg": "Calculation copied!",
        "delete_entry": "Delete entry",
        "delete_confirm": "Delete?",
        "delete_car_confirm": "Delete this vehicle?",
        "delete_history_confirm": "Delete entry?",
        "delete_history_msg": "Are you sure you want to delete this entry?",
        "clear_history_confirm": "Clear history?",
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
        "language_setting": "Language",
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
        "version": "Version 1.0  •  2026",
        "about_section1_title": "About the app",
        "about_section1": "Professional fuel calculator with PostgreSQL database.",
        "about_section2_title": "Features",
        "about_section2": (f"• Accurate consumption calculation\n"
                           f"• Trip cost estimation\n"
                           f"• Up to {MAX_CARS_PER_USER} vehicles\n"
                           f"• History tracking\n"
                           f"• Interactive charts\n"
                           f"• Multiple currencies and themes"),
        "about_section3_title": "How to use",
        "about_section3": ("1. Register or sign in\n"
                           "2. Add a vehicle with photo\n"
                           "3. Select calculation mode\n"
                           "4. Enter trip data\n"
                           "5. Press Calculate – result will be saved automatically"),
        "about_section4_title": "Security",
        "about_section4": "Passwords are encrypted with modern algorithms. Your data is securely stored in the PostgreSQL database.",
        "about_section5_title": "Fuel consumption interpretation",
        "about_section5": ("Up to 8 L/100 km – excellent economy\n"
                           "8–12 L/100 km – average\n"
                           "Over 12 L/100 km – high, consider vehicle check"),
        "about_section6_title": "Fuel saving tips",
        "about_section6": ("• Smooth acceleration and braking\n"
                           "• Maintain proper tire pressure\n"
                           "• Remove unnecessary weight\n"
                           "• Regular maintenance\n"
                           "• Use quality fuel"),
        "footer": "© 2026 FullCalcPro",
        "no_cars_limit": f"Maximum {MAX_CARS_PER_USER} vehicles",
        "database_error": "Database connection error",
        "database_retry": "Please try again later or check PostgreSQL settings",
        "cancel": "Cancel",
        "old_password_wrong": "Current password is incorrect",
        "date": "Date",
        "show_more": "Show more",
        "show_less": "Show less",
        "password_changed": "Password changed successfully",
        "clear_history_btn": "Clear all history",
        "car_stats_title": "Vehicle statistics",
        "total_distance": "Total distance",
        "total_cost": "Total cost",
        "km": "km",
        "no_data": "No data"
    }
}

CURRENCIES = {"₽ RUB": "₽", "$ USD": "$", "€ EUR": "€"}

THEMES = {
    "dark": {
        "bg": "#0a0e17",
        "bg2": "#111827",
        "bg3": "#1f2937",
        "fg": "#f3f4f6",
        "fg2": "#9ca3af",
        "accent": "#3b82f6",
        "green": "#10b981",
        "yellow": "#f59e0b",
        "red": "#ef4444",
        "border": "#374151",
        "btn": "#3b82f6",
        "btn_hover": "#2563eb",
        "btn2": "#1f2937",
        "btn2_hover": "#374151",
        "input_bg": "#111827",
        "nav_bg": "rgba(17, 24, 39, 0.95)",
        "card_bg": "rgba(17, 24, 39, 0.8)",
        "card_border": "rgba(59, 130, 246, 0.2)",
        "glass_bg": "rgba(17, 24, 39, 0.7)"
    },
    "light": {
        "bg": "#f0f4f8",
        "bg2": "#ffffff",
        "bg3": "#e2e8f0",
        "fg": "#1e293b",
        "fg2": "#64748b",
        "accent": "#3b82f6",
        "green": "#10b981",
        "yellow": "#f59e0b",
        "red": "#ef4444",
        "border": "#e2e8f0",
        "btn": "#3b82f6",
        "btn_hover": "#2563eb",
        "btn2": "#e2e8f0",
        "btn2_hover": "#cbd5e1",
        "input_bg": "#ffffff",
        "nav_bg": "rgba(255, 255, 255, 0.95)",
        "card_bg": "rgba(255, 255, 255, 0.9)",
        "card_border": "rgba(59, 130, 246, 0.15)",
        "glass_bg": "rgba(255, 255, 255, 0.8)"
    }
}

# ---------- Шаблоны ----------
TEMPLATES = {
    "base.html": r'''<!DOCTYPE html>
<html lang="{{ lang }}">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ tr('app_title') }}</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700;800;900&display=swap" rel="stylesheet">
    <style>
        :root {
            --bg: {{ theme.bg }};
            --bg2: {{ theme.bg2 }};
            --bg3: {{ theme.bg3 }};
            --fg: {{ theme.fg }};
            --fg2: {{ theme.fg2 }};
            --accent: {{ theme.accent }};
            --accent-rgb: 59, 130, 246;
            --green: {{ theme.green }};
            --green-rgb: 16, 185, 129;
            --yellow: {{ theme.yellow }};
            --red: {{ theme.red }};
            --border: {{ theme.border }};
            --btn: {{ theme.btn }};
            --btn-hover: {{ theme.btn_hover }};
            --input-bg: {{ theme.input_bg }};
            --nav-bg: {{ theme.nav_bg }};
            --card-bg: {{ theme.card_bg }};
            --card-border: {{ theme.card_border }};
            --glass-bg: {{ theme.glass_bg }};
        }
        
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        
        body {
            background: linear-gradient(135deg, var(--bg) 0%, var(--bg2) 50%, var(--bg3) 100%);
            color: var(--fg);
            min-height: 100vh;
            font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
            position: relative;
            overflow-x: hidden;
        }
        
        body::before {
            content: '';
            position: fixed;
            top: -50%;
            left: -50%;
            width: 200%;
            height: 200%;
            background: 
                radial-gradient(circle at 20% 80%, rgba(var(--accent-rgb), 0.08) 0%, transparent 50%),
                radial-gradient(circle at 80% 20%, rgba(var(--green-rgb), 0.08) 0%, transparent 50%),
                radial-gradient(circle at 50% 50%, rgba(var(--accent-rgb), 0.05) 0%, transparent 50%);
            animation: bgRotate 30s linear infinite;
            z-index: 0;
        }
        
        @keyframes bgRotate {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
        
        .navbar {
            background: var(--nav-bg) !important;
            backdrop-filter: blur(20px);
            -webkit-backdrop-filter: blur(20px);
            border-bottom: 1px solid var(--card-border);
            box-shadow: 0 4px 30px rgba(0, 0, 0, 0.1);
            position: sticky;
            top: 0;
            z-index: 1000;
        }
        
        .navbar .container {
            position: relative;
        }
        
        .logo-container {
            display: flex;
            align-items: center;
            text-decoration: none;
        }
        
        .logo-icon {
            width: 45px;
            height: 45px;
            border-radius: 12px;
            object-fit: contain;
            margin-right: 12px;
            filter: drop-shadow(0 4px 6px rgba(0, 0, 0, 0.2));
            transition: transform 0.3s ease;
        }
        
        .logo-icon:hover {
            transform: scale(1.05);
        }
        
        .logo-icon.fallback {
            width: 45px;
            height: 45px;
            background: linear-gradient(135deg, var(--accent), var(--green));
            border-radius: 12px;
            display: none;
            align-items: center;
            justify-content: center;
            color: white;
            font-size: 1.4rem;
            box-shadow: 0 4px 15px rgba(var(--accent-rgb), 0.4);
        }
        
        .navbar-brand-text {
            font-weight: 800;
            font-size: 1.5rem;
            background: linear-gradient(135deg, var(--accent), var(--green));
            -webkit-background-clip: text;
            -webkit-text-fill-color: transparent;
            background-clip: text;
            letter-spacing: -0.5px;
        }
        
        .nav-link {
            font-weight: 600;
            position: relative;
            margin: 0 8px;
            color: var(--fg) !important;
            transition: color 0.3s ease;
        }
        
        .nav-link:hover {
            color: var(--accent) !important;
        }
        
        .nav-link::after {
            content: '';
            position: absolute;
            bottom: -2px;
            left: 50%;
            width: 0;
            height: 2px;
            background: linear-gradient(90deg, var(--accent), var(--green));
            transition: all 0.3s ease;
            transform: translateX(-50%);
        }
        
        .nav-link:hover::after {
            width: 100%;
        }
        
        .card, .white-bg, .window {
            background: var(--card-bg) !important;
            backdrop-filter: blur(20px);
            -webkit-backdrop-filter: blur(20px);
            border: 1px solid var(--card-border);
            border-radius: 20px;
            box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
            transition: all 0.3s ease;
            position: relative;
            z-index: 1;
        }
        
        .card:hover {
            transform: translateY(-2px);
            box-shadow: 0 12px 40px rgba(0, 0, 0, 0.15);
            border-color: rgba(var(--accent-rgb), 0.3);
        }
        
        .btn {
            border-radius: 12px;
            font-weight: 600;
            transition: all 0.3s ease;
            letter-spacing: 0.3px;
            position: relative;
            overflow: hidden;
        }
        
        .btn::before {
            content: '';
            position: absolute;
            top: 50%;
            left: 50%;
            width: 0;
            height: 0;
            border-radius: 50%;
            background: rgba(255, 255, 255, 0.3);
            transform: translate(-50%, -50%);
            transition: width 0.6s, height 0.6s;
        }
        
        .btn:hover::before {
            width: 300px;
            height: 300px;
        }
        
        .btn-success {
            background: linear-gradient(135deg, var(--green), #059669);
            border: none;
            color: white;
        }
        
        .btn-success:hover {
            background: linear-gradient(135deg, #059669, var(--green));
            transform: translateY(-2px);
            box-shadow: 0 8px 25px rgba(var(--green-rgb), 0.4);
        }
        
        .btn-danger {
            background: linear-gradient(135deg, var(--red), #dc2626);
            border: none;
            color: white;
        }
        
        .btn-danger:hover {
            background: linear-gradient(135deg, #dc2626, var(--red));
            transform: translateY(-2px);
            box-shadow: 0 8px 25px rgba(239, 68, 68, 0.4);
        }
        
        .btn-primary {
            background: linear-gradient(135deg, var(--accent), #2563eb);
            border: none;
            color: white;
        }
        
        .btn-primary:hover {
            background: linear-gradient(135deg, #2563eb, var(--accent));
            transform: translateY(-2px);
            box-shadow: 0 8px 25px rgba(var(--accent-rgb), 0.4);
        }
        
        .btn-outline-dark {
            border: 2px solid var(--border);
            color: var(--fg);
            background: transparent;
            backdrop-filter: blur(10px);
        }
        
        .btn-outline-dark:hover {
            background: var(--accent);
            border-color: var(--accent);
            color: white;
            transform: translateY(-2px);
            box-shadow: 0 8px 25px rgba(var(--accent-rgb), 0.3);
        }
        
        .form-control, .form-select {
            background: var(--input-bg);
            color: var(--fg);
            border: 2px solid var(--border);
            border-radius: 12px;
            padding: 12px 16px;
            transition: all 0.3s ease;
            backdrop-filter: blur(10px);
        }
        
        .form-control:focus, .form-select:focus {
            background: var(--input-bg);
            border-color: var(--accent);
            box-shadow: 0 0 0 4px rgba(var(--accent-rgb), 0.1);
            color: var(--fg);
        }
        
        .form-control::placeholder {
            color: var(--fg2);
        }
        
        .alert {
            border-radius: 16px;
            font-weight: 500;
            backdrop-filter: blur(20px);
            border: 1px solid var(--card-border);
            animation: slideIn 0.3s ease;
        }
        
        @keyframes slideIn {
            from {
                opacity: 0;
                transform: translateY(-10px);
            }
            to {
                opacity: 1;
                transform: translateY(0);
            }
        }
        
        .alert-success {
            background: rgba(var(--green-rgb), 0.1);
            border-color: rgba(var(--green-rgb), 0.3);
            color: var(--green);
        }
        
        .alert-danger {
            background: rgba(239, 68, 68, 0.1);
            border-color: rgba(239, 68, 68, 0.3);
            color: var(--red);
        }
        
        .alert-warning {
            background: rgba(245, 158, 11, 0.1);
            border-color: rgba(245, 158, 11, 0.3);
            color: var(--yellow);
        }
        
        .list-group-item {
            background: var(--glass-bg);
            backdrop-filter: blur(10px);
            border: 1px solid var(--card-border);
            border-radius: 16px;
            margin-bottom: 12px;
            transition: all 0.3s ease;
        }
        
        .list-group-item:hover {
            background: var(--card-bg);
            transform: translateX(5px);
            border-color: rgba(var(--accent-rgb), 0.3);
            box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
        }
        
        .chart-container {
            background: var(--glass-bg);
            backdrop-filter: blur(20px);
            border-radius: 20px;
            padding: 20px;
            border: 1px solid var(--card-border);
        }
        
        .car-photo-small {
            width: 65px;
            height: 65px;
            object-fit: cover;
            border-radius: 14px;
            box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
            border: 2px solid var(--card-border);
        }
        
        .car-photo-medium {
            height: 180px;
            object-fit: cover;
            border-radius: 16px;
            box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
            border: 2px solid var(--card-border);
        }
        
        .input-group .btn-eye {
            background: var(--input-bg);
            border: 2px solid var(--border);
            color: var(--fg2);
            backdrop-filter: blur(10px);
            transition: all 0.3s ease;
        }
        
        .input-group .btn-eye:hover {
            background: var(--border);
            color: var(--fg);
        }
        
        .badge {
            border-radius: 20px;
            padding: 6px 14px;
            font-weight: 600;
            background: linear-gradient(135deg, var(--accent), #2563eb);
            color: white;
        }
        
        .history-hidden {
            display: none;
        }
        
        footer {
            margin-top: 4rem;
            text-align: center;
            padding: 2rem;
            font-size: 0.9rem;
            color: var(--fg2);
            position: relative;
            z-index: 1;
        }
        
        h1, h2, h3, h4, h5, h6 {
            font-weight: 700;
            letter-spacing: -0.3px;
        }
        
        h2 {
            background: linear-gradient(135deg, var(--fg), var(--fg2));
            -webkit-background-clip: text;
            -webkit-text-fill-color: transparent;
            background-clip: text;
        }
        
        .nav-tabs .nav-link {
            border: none;
            color: var(--fg2);
            border-radius: 12px;
            padding: 10px 20px;
            font-weight: 600;
            transition: all 0.3s ease;
        }
        
        .nav-tabs .nav-link.active {
            background: linear-gradient(135deg, var(--accent), #2563eb);
            color: white;
            box-shadow: 0 4px 15px rgba(var(--accent-rgb), 0.3);
        }
        
        .nav-tabs .nav-link:hover:not(.active) {
            color: var(--fg);
            background: var(--glass-bg);
        }
        
        .form-check-input:checked {
            background-color: var(--accent);
            border-color: var(--accent);
        }
        
        .container {
            position: relative;
            z-index: 1;
        }
        
        @media (max-width: 768px) {
            .navbar-brand-text {
                font-size: 1.2rem;
            }
            .logo-icon {
                width: 35px;
                height: 35px;
            }
            .card {
                border-radius: 16px;
            }
        }
    </style>
</head>
<body>
    <nav class="navbar navbar-expand-lg">
        <div class="container">
            <a class="logo-container" href="/">
                <img src="{{ url_for('static', filename='logo.png') }}" class="logo-icon" alt="Logo" onerror="this.style.display='none';this.nextElementSibling.style.display='flex';">
                <span class="logo-icon fallback"><i class="fas fa-gas-pump"></i></span>
                <span class="navbar-brand-text">{{ tr('app_title') }}</span>
            </a>
            <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarNav">
                <span class="navbar-toggler-icon"></span>
            </button>
            <div class="collapse navbar-collapse" id="navbarNav">
                <ul class="navbar-nav ms-auto">
                    <li class="nav-item"><a class="nav-link" href="/"><i class="fas fa-calculator me-1"></i>{{ tr('calculator') }}</a></li>
                    {% if user %}
                    <li class="nav-item"><a class="nav-link" href="/profile"><i class="fas fa-user me-1"></i>{{ tr('profile') }}</a></li>
                    <li class="nav-item"><a class="nav-link" href="/settings"><i class="fas fa-cog me-1"></i>{{ tr('settings') }}</a></li>
                    {% else %}
                    <li class="nav-item"><a class="nav-link" href="/auth"><i class="fas fa-sign-in-alt me-1"></i>{{ tr('login_register') }}</a></li>
                    {% endif %}
                    <li class="nav-item"><a class="nav-link" href="/about"><i class="fas fa-info-circle me-1"></i>{{ tr('about') }}</a></li>
                </ul>
            </div>
        </div>
    </nav>
    <div class="container mt-4">
        {% with messages = get_flashed_messages(with_categories=true) %}
            {% if messages %}
                {% for category, message in messages %}
                    <div class="alert alert-{{ category }} alert-dismissible fade show" role="alert">
                        {{ message }}
                        <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
                    </div>
                {% endfor %}
            {% endif %}
        {% endwith %}
        {% block content %}{% endblock %}
    </div>
    <footer>
        {{ tr('footer') }}
    </footer>
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <script>
        function togglePassword(btn) {
            const input = btn.parentElement.querySelector('input');
            const icon = btn.querySelector('i');
            if (input.type === 'password') {
                input.type = 'text';
                icon.classList.replace('fa-eye', 'fa-eye-slash');
            } else {
                input.type = 'password';
                icon.classList.replace('fa-eye-slash', 'fa-eye');
            }
        }
    </script>
    {% block scripts %}{% endblock %}
</body>
</html>''',

    "index.html": r'''{% extends "base.html" %}
{% block content %}
<h2 class="mb-4"><i class="fas fa-calculator me-2"></i>{{ tr('calculator') }}</h2>
<div class="row">
    <div class="col-md-6">
        <div class="card p-4 mb-4">
            <div class="mb-3">
                <label class="form-label"><i class="fas fa-car me-1"></i>{{ tr('car') }}</label>
                <select id="carSelect" class="form-select">
                    <option value="">{{ tr('no_car') }}</option>
                    {% for car in cars %}
                    <option value="{{ car.name }}" data-photo="{% if car.photo %}{{ url_for('static', filename='uploads/' + car.photo) }}{% endif %}">
                        {{ car.name }}
                    </option>
                    {% endfor %}
                </select>
                <div class="mt-2 text-center">
                    <img id="carPhoto" src="" class="img-fluid rounded" style="max-height:200px; display:none;">
                </div>
            </div>
            <form id="calcForm">
                <div class="mb-3">
                    <div class="form-check form-check-inline">
                        <input class="form-check-input" type="radio" name="mode" value="consumption" checked>
                        <label class="form-check-label">{{ tr('calc_mode_consumption') }}</label>
                    </div>
                    <div class="form-check form-check-inline">
                        <input class="form-check-input" type="radio" name="mode" value="cost">
                        <label class="form-check-label">{{ tr('calc_mode_cost') }}</label>
                    </div>
                </div>
                <div class="mb-2">
                    <label class="form-label"><i class="fas fa-road me-1"></i>{{ tr('distance_label') }}, км</label>
                    <input type="number" step="any" name="distance" id="distance" class="form-control" required>
                </div>
                <div id="fuelField" class="mb-2">
                    <label class="form-label"><i class="fas fa-gas-pump me-1"></i>{{ tr('fuel_used') }}, л</label>
                    <input type="number" step="any" name="fuel" id="fuel" class="form-control">
                </div>
                <div id="avgField" class="mb-2" style="display:none;">
                    <label class="form-label"><i class="fas fa-tachometer-alt me-1"></i>{{ tr('avg_consumption_label') }}, л/100км</label>
                    <input type="number" step="any" name="avg_consumption" id="avgConsumption" class="form-control">
                </div>
                <div class="mb-2">
                    <label class="form-label"><i class="fas fa-coins me-1"></i>{{ tr('fuel_price') }}, {{ currency_symbol }}/л</label>
                    <input type="number" step="any" name="price" id="price" class="form-control" required>
                </div>
                <button type="submit" class="btn btn-success me-2"><i class="fas fa-play me-1"></i>{{ tr('calculate') }}</button>
                <button type="button" id="clearBtn" class="btn btn-danger"><i class="fas fa-undo me-1"></i>{{ tr('clear') }}</button>
            </form>
            <div id="result" class="mt-3 alert alert-success" style="display:none;"></div>
        </div>
    </div>
    <div class="col-md-6">
        <div class="card p-4">
            <h5><i class="fas fa-chart-line me-2"></i>{{ tr('fuel_chart_title') }}</h5>
            <div class="mb-2">
                <label class="form-label">{{ tr('chart_days') }}</label>
                <select id="chartDays" class="form-select w-auto d-inline">
                    <option value="7">7</option>
                    <option value="30" selected>30</option>
                    <option value="90">90</option>
                    <option value="180">180</option>
                    <option value="360">360</option>
                </select>
            </div>
            <div class="chart-container">
                <canvas id="fuelChart"></canvas>
            </div>
        </div>
    </div>
</div>

<div class="mt-5">
    <div class="d-flex justify-content-between align-items-center mb-3">
        <h4><i class="fas fa-history me-2"></i>{{ tr('history_title') }}</h4>
        {% if user %}
        <button id="clearHistoryBtn" class="btn btn-outline-dark" onclick="clearAllHistory()"><i class="fas fa-trash-alt me-1"></i>{{ tr('clear_all') }}</button>
        {% endif %}
    </div>
    <div id="historyContainer">
        {% include 'history_list.html' %}
    </div>
</div>
{% endblock %}
{% block scripts %}
<script>
    document.querySelectorAll('input[name="mode"]').forEach(r => {
        r.addEventListener('change', function() {
            if (this.value === 'consumption') {
                document.getElementById('fuelField').style.display = 'block';
                document.getElementById('avgField').style.display = 'none';
            } else {
                document.getElementById('fuelField').style.display = 'none';
                document.getElementById('avgField').style.display = 'block';
            }
        });
    });

    document.getElementById('carSelect').addEventListener('change', function() {
        const opt = this.options[this.selectedIndex];
        const url = opt.getAttribute('data-photo');
        const img = document.getElementById('carPhoto');
        if (url && url !== 'None') { img.src = url; img.style.display = 'block'; }
        else { img.style.display = 'none'; }
        loadChart(this.value);
    });

    document.getElementById('calcForm').addEventListener('submit', async function(e) {
        e.preventDefault();
        const fd = new FormData(this);
        fd.append('car', document.getElementById('carSelect').value);
        const res = await fetch('/calculate', { method: 'POST', body: fd });
        const data = await res.json();
        const div = document.getElementById('result');
        if (data.error) {
            div.className = 'mt-3 alert alert-danger';
            div.innerText = data.error;
        } else {
            div.className = 'mt-3 alert alert-success';
            div.innerText = (data.mode === 'consumption')
                ? `Расход: ${data.consumption} л/100км, Стоимость: ${data.cost} ${data.currency}`
                : `Стоимость поездки: ${data.cost} ${data.currency}`;
        }
        div.style.display = 'block';
        loadHistory();
        loadChart(document.getElementById('carSelect').value);
    });

    document.getElementById('clearBtn').addEventListener('click', function() {
        document.getElementById('calcForm').reset();
        document.getElementById('result').style.display = 'none';
        document.getElementById('fuelField').style.display = 'block';
        document.getElementById('avgField').style.display = 'none';
    });

    let chart;
    async function loadChart(car = '') {
        const days = document.getElementById('chartDays').value || 30;
        const res = await fetch(`/chart_data?car=${encodeURIComponent(car)}&days=${days}`);
        const json = await res.json();
        const ctx = document.getElementById('fuelChart').getContext('2d');
        if (chart) chart.destroy();
        const gradient = ctx.createLinearGradient(0, 0, 0, 300);
        gradient.addColorStop(0, '{{ theme.accent }}');
        gradient.addColorStop(1, '{{ theme.green }}');
        chart = new Chart(ctx, {
            type: 'line',
            data: {
                labels: json.labels,
                datasets: [{
                    label: 'л/100км',
                    data: json.values,
                    borderColor: gradient,
                    backgroundColor: 'rgba({{ theme.accent|replace("#", "")|regex_replace("(..)(..)(..)", "\\1, \\2, \\3") }}, 0.2)',
                    fill: true,
                    tension: 0.4,
                    pointBackgroundColor: '{{ theme.accent }}',
                    pointBorderColor: '#fff',
                    pointBorderWidth: 2,
                    pointRadius: 5,
                    pointHoverRadius: 8,
                    borderWidth: 3
                }]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                plugins: {
                    legend: { labels: { color: '{{ theme.fg }}' } }
                },
                scales: {
                    y: { beginAtZero: false, grid: { color: '{{ theme.border }}' }, ticks: { color: '{{ theme.fg2 }}' } },
                    x: { grid: { color: '{{ theme.border }}' }, ticks: { color: '{{ theme.fg2 }}' } }
                }
            }
        });
    }
    loadChart();
    document.getElementById('chartDays').addEventListener('change', () => loadChart(document.getElementById('carSelect').value));

    let currentVisible = 10;
    async function loadHistory() {
        const car = document.getElementById('carSelect').value;
        const res = await fetch(`/history_html?car=${encodeURIComponent(car)}`);
        const html = await res.text();
        document.getElementById('historyContainer').innerHTML = html;
        currentVisible = 10;
    }

    async function clearAllHistory() {
        if (!confirm('{{ tr("clear_history_msg") }}')) return;
        await fetch('/clear_history', { method: 'POST' });
        loadHistory();
    }

    async function deleteHistoryEntry(id) {
        if (!confirm('{{ tr("delete_history_msg") }}')) return;
        await fetch('/delete_history/' + id, { method: 'POST' });
        loadHistory();
    }

    function copyEntry(btn) {
        const text = btn.getAttribute('data-text');
        navigator.clipboard.writeText(text).then(() => {
            alert('{{ tr("copied_msg") }}');
        }).catch(() => {
            alert('Ошибка копирования');
        });
    }

    window.toggleHistory = function() {
        const items = document.querySelectorAll('.history-item');
        const total = items.length;
        const btn = document.getElementById('toggleHistoryBtn');
        if (!btn) return;

        if (currentVisible >= total) {
            const newVisible = 10;
            for (let i = newVisible; i < currentVisible; i++) {
                if (items[i]) items[i].classList.add('history-hidden');
            }
            currentVisible = newVisible;
        } else {
            const nextLimit = Math.min(currentVisible + 10, total);
            for (let i = currentVisible; i < nextLimit; i++) {
                if (items[i]) items[i].classList.remove('history-hidden');
            }
            currentVisible = nextLimit;
        }

        btn.textContent = (currentVisible >= total) ? '{{ tr("show_less") }}' : '{{ tr("show_more") }}';
    };
</script>
{% endblock %}''',

    "history_list.html": r'''{% if user %}
    {% if history %}
        <div class="list-group" id="historyList">
        {% for entry in history %}
            <div class="list-group-item d-flex align-items-center p-3 history-item {% if loop.index > 10 %}history-hidden{% endif %}">
                {% if entry.photo %}
                <img src="{{ url_for('static', filename='uploads/' + entry.photo) }}" class="car-photo-small me-3">
                {% else %}
                <div class="car-photo-small me-3 bg-secondary d-flex align-items-center justify-content-center text-white" style="min-width:65px;">
                    <i class="fas fa-car"></i>
                </div>
                {% endif %}
                <div class="flex-grow-1">
                    <div class="fw-semibold mb-1">{{ entry.date }} – {{ entry.car or tr('no_car') }}</div>
                    <div class="small">
                        <span class="me-3"><i class="fas fa-road me-1"></i>{{ entry.distance }} км</span>
                        <span class="me-3"><i class="fas fa-gas-pump me-1"></i>{{ entry.fuel }} л</span>
                        <span class="me-3"><i class="fas fa-tachometer-alt me-1"></i>{{ entry.consumption }} л/100км</span>
                        <span><i class="fas fa-coins me-1"></i>{{ "%.2f"|format(entry.cost) }} {{ currency_symbol }}</span>
                    </div>
                </div>
                <div class="ms-auto d-flex flex-column align-items-end">
                    <button class="btn btn-sm btn-outline-secondary mb-1" onclick="copyEntry(this)" data-text="Дата: {{ entry.date }}, Авто: {{ entry.car or tr('no_car') }}, Расход: {{ entry.consumption }} л/100км, Стоимость: {{ '%.2f'|format(entry.cost) }} {{ currency_symbol }}">
                        <i class="fas fa-copy"></i>
                    </button>
                    <button class="btn btn-sm btn-outline-danger" onclick="deleteHistoryEntry({{ entry.id }})"><i class="fas fa-trash"></i></button>
                </div>
            </div>
        {% endfor %}
        </div>
        {% if history|length > 10 %}
        <div class="text-center mt-3">
            <button id="toggleHistoryBtn" class="btn btn-outline-dark" onclick="toggleHistory()">{{ tr('show_more') }}</button>
        </div>
        {% endif %}
    {% else %}
        <div class="card p-4 text-center">{{ tr('history_empty') }}</div>
    {% endif %}
{% else %}
    <div class="card p-4 text-center">{{ tr('login_for_history') }}</div>
{% endif %}''',

    "auth.html": r'''{% extends "base.html" %}
{% block content %}
<div class="row justify-content-center">
    <div class="col-md-5">
        <div class="card p-4">
            <h2 class="mb-4 text-center"><i class="fas fa-user-lock me-2"></i>{{ tr('login_register') }}</h2>
            <ul class="nav nav-tabs mb-3" id="authTab" role="tablist">
                <li class="nav-item" role="presentation">
                    <button class="nav-link active" id="login-tab" data-bs-toggle="tab" data-bs-target="#login-pane" type="button" role="tab">{{ tr('login') }}</button>
                </li>
                <li class="nav-item" role="presentation">
                    <button class="nav-link" id="register-tab" data-bs-toggle="tab" data-bs-target="#register-pane" type="button" role="tab">{{ tr('register') }}</button>
                </li>
            </ul>
            <div class="tab-content">
                <div class="tab-pane fade show active" id="login-pane" role="tabpanel">
                    <form method="post" action="/auth">
                        <input type="hidden" name="form_type" value="login">
                        <div class="mb-3">
                            <label>{{ tr('email') }}</label>
                            <input type="email" name="email" class="form-control" required>
                        </div>
                        <div class="mb-3">
                            <label>{{ tr('password') }}</label>
                            <div class="input-group">
                                <input type="password" name="password" class="form-control" required>
                                <button class="btn btn-outline-secondary btn-eye" type="button" onclick="togglePassword(this)">
                                    <i class="fas fa-eye"></i>
                                </button>
                            </div>
                        </div>
                        <button type="submit" class="btn btn-primary w-100">{{ tr('login_account') }}</button>
                    </form>
                </div>
                <div class="tab-pane fade" id="register-pane" role="tabpanel">
                    <form method="post" action="/auth">
                        <input type="hidden" name="form_type" value="register">
                        <div class="mb-3">
                            <label>{{ tr('email') }}</label>
                            <input type="email" name="email" class="form-control" required>
                        </div>
                        <div class="mb-3">
                            <label>{{ tr('password') }}</label>
                            <div class="input-group">
                                <input type="password" name="password" class="form-control" required>
                                <button class="btn btn-outline-secondary btn-eye" type="button" onclick="togglePassword(this)">
                                    <i class="fas fa-eye"></i>
                                </button>
                            </div>
                        </div>
                        <div class="mb-3">
                            <label>{{ tr('repeat_password') }}</label>
                            <div class="input-group">
                                <input type="password" name="password2" class="form-control" required>
                                <button class="btn btn-outline-secondary btn-eye" type="button" onclick="togglePassword(this)">
                                    <i class="fas fa-eye"></i>
                                </button>
                            </div>
                        </div>
                        <button type="submit" class="btn btn-primary w-100">{{ tr('create_account') }}</button>
                    </form>
                </div>
            </div>
        </div>
    </div>
</div>
{% endblock %}''',

    "profile.html": r'''{% extends "base.html" %}
{% block content %}
<div class="d-flex justify-content-between align-items-center mb-4">
    <h2><i class="fas fa-user-circle me-2"></i>{{ tr('profile') }}</h2>
    <div class="d-flex gap-2">
        <button id="togglePasswordBtn" class="btn btn-outline-dark" onclick="toggleChangePassword()">
            <i class="fas fa-lock me-1"></i>{{ tr('change_password') }}
        </button>
        <a href="/logout" class="btn btn-outline-dark"><i class="fas fa-sign-out-alt me-1"></i>{{ tr('logout') }}</a>
        <form method="post" action="/delete_account" onsubmit="return confirm('{{ tr('delete_account_msg') }}')" class="m-0">
            <button class="btn btn-danger"><i class="fas fa-user-times me-1"></i>{{ tr('delete_account') }}</button>
        </form>
    </div>
</div>
<div class="card p-4 mb-4">
    <p><i class="fas fa-envelope me-2"></i><strong>{{ user }}</strong></p>
</div>

<div class="card p-4 mb-4" id="changePasswordCard" style="display:none;">
    <h5><i class="fas fa-lock me-2"></i>{{ tr('change_password') }}</h5>
    <form method="post" action="/change_password" onsubmit="submitChangePassword(event)">
        <div class="mb-2">
            <label>{{ tr('old_password') }}</label>
            <div class="input-group">
                <input type="password" name="old_password" class="form-control" required>
                <button class="btn btn-outline-secondary btn-eye" type="button" onclick="togglePassword(this)">
                    <i class="fas fa-eye"></i>
                </button>
            </div>
        </div>
        <div class="mb-2">
            <label>{{ tr('new_password') }}</label>
            <div class="input-group">
                <input type="password" name="new_password" class="form-control" required>
                <button class="btn btn-outline-secondary btn-eye" type="button" onclick="togglePassword(this)">
                    <i class="fas fa-eye"></i>
                </button>
            </div>
        </div>
        <div class="mb-2">
            <label>{{ tr('repeat_new_password') }}</label>
            <div class="input-group">
                <input type="password" name="new_password2" class="form-control" required>
                <button class="btn btn-outline-secondary btn-eye" type="button" onclick="togglePassword(this)">
                    <i class="fas fa-eye"></i>
                </button>
            </div>
        </div>
        <button type="submit" class="btn btn-primary">{{ tr('change_password_btn') }}</button>
    </form>
</div>

<div class="card p-4" id="carsSection">
    {% include 'profile_cars.html' %}
</div>
{% endblock %}
{% block scripts %}
<script>
function toggleChangePassword() {
    const card = document.getElementById('changePasswordCard');
    if (card.style.display === 'none') {
        card.style.display = 'block';
    } else {
        card.style.display = 'none';
    }
}

async function submitChangePassword(e) {
    e.preventDefault();
    const form = e.target;
    const fd = new FormData(form);
    const res = await fetch('/change_password', { method: 'POST', body: fd });
    const data = await res.json();
    alert(data.message);
    if (data.success) {
        form.reset();
    }
}

async function addCar(e) {
    e.preventDefault();
    const form = e.target;
    const fd = new FormData(form);
    const res = await fetch('/add_car', { method: 'POST', body: fd });
    const html = await res.text();
    document.getElementById('carsSection').innerHTML = html;
}

async function deleteCar(name) {
    if (!confirm('{{ tr("delete_car_confirm") }}')) return;
    const res = await fetch('/delete_car/' + name, { method: 'POST' });
    const html = await res.text();
    document.getElementById('carsSection').innerHTML = html;
}
</script>
{% endblock %}''',

    "profile_cars.html": r'''<h5><i class="fas fa-car me-2"></i>{{ tr('my_cars') }}</h5>
<h6 class="mt-2"><i class="fas fa-plus-circle me-1"></i>{{ tr('add_car_title') }}</h6>
<form onsubmit="addCar(event)" enctype="multipart/form-data" class="row g-2 mb-4">
    <div class="col-md-5">
        <input type="text" name="name" class="form-control" placeholder="{{ tr('car_name') }}" required>
    </div>
    <div class="col-md-5">
        <input type="file" name="photo" class="form-control" accept="image/*">
    </div>
    <div class="col-md-2">
        <button type="submit" class="btn btn-success w-100">{{ tr('save') }}</button>
    </div>
</form>

<div class="row">
    {% for car in cars %}
    <div class="col-md-4 mb-4">
        <div class="card p-2 h-100">
            {% if car.photo %}
            <img src="{{ url_for('static', filename='uploads/' + car.photo) }}" class="car-photo-medium w-100 mb-2">
            {% else %}
            <div class="bg-secondary d-flex align-items-center justify-content-center car-photo-medium w-100 mb-2" style="height:180px;">
                <i class="fas fa-car fa-3x text-white"></i>
            </div>
            {% endif %}
            <div class="p-2 text-center">
                <h6 class="fw-bold">{{ car.name }}</h6>
                {% if car.avg_consumption %}
                <span class="badge bg-primary">{{ "%.1f"|format(car.avg_consumption) }} л/100км</span>
                {% endif %}
                <div class="mt-2 small text-muted">
                    {% set stats = car_stats.get(car.name, {}) %}
                    <div>{{ tr('total_distance') }}: {{ stats.total_distance }} {{ tr('km') }}</div>
                    <div>{{ tr('total_cost') }}: {{ stats.total_cost }} {{ currency_symbol }}</div>
                    <div>{{ tr('avg_consumption') }}: {{ stats.avg_consumption }} л/100км</div>
                </div>
                <button class="btn btn-sm btn-danger mt-2" onclick="deleteCar('{{ car.name }}')"><i class="fas fa-trash-alt me-1"></i>{{ tr('delete') }}</button>
            </div>
        </div>
    </div>
    {% else %}
    <div class="col-12"><p>{{ tr('no_cars') }}</p></div>
    {% endfor %}
</div>''',

    "settings.html": r'''{% extends "base.html" %}
{% block content %}
<h2 class="mb-4"><i class="fas fa-sliders-h me-2"></i>{{ tr('settings_title') }}</h2>
<div class="card p-4">
    <form id="settingsForm" onsubmit="submitSettings(event)">
        <div class="mb-4">
            <label class="form-label fw-bold"><i class="fas fa-globe me-2"></i>{{ tr('language_setting') }}</label>
            <select name="language" class="form-select">
                <option value="ru" {% if settings.language == 'ru' %}selected{% endif %}>Русский</option>
                <option value="en" {% if settings.language == 'en' %}selected{% endif %}>English</option>
            </select>
        </div>
        <div class="mb-4">
            <label class="form-label fw-bold"><i class="fas fa-money-bill-wave me-2"></i>{{ tr('currency_setting') }}</label>
            <select name="currency" class="form-select">
                {% for key, sym in currencies.items() %}
                <option value="{{ key }}" {% if settings.currency == key %}selected{% endif %}>{{ sym }}</option>
                {% endfor %}
            </select>
        </div>
        <button type="submit" class="btn btn-primary"><i class="fas fa-save me-1"></i>{{ tr('save') }}</button>
    </form>
</div>
{% endblock %}
{% block scripts %}
<script>
async function submitSettings(e) {
    e.preventDefault();
    const form = e.target;
    const fd = new FormData(form);
    const res = await fetch('/settings', { method: 'POST', body: fd });
    if (res.ok) {
        location.reload();
    }
}
</script>
{% endblock %}''',

    "about.html": r'''{% extends "base.html" %}
{% block content %}
<h2 class="mb-4"><i class="fas fa-info-circle me-2"></i>{{ tr('about_title') }}</h2>
<div class="card p-4">
    <div class="row">
        <div class="col-md-6">
            <h5>{{ tr('about_section1_title') }}</h5>
            <p>{{ tr('about_section1') }}</p>
            <h5>{{ tr('about_section2_title') }}</h5>
            <p>{{ tr('about_section2')|replace('\n', '<br>')|safe }}</p>
            <h5>{{ tr('about_section3_title') }}</h5>
            <p>{{ tr('about_section3')|replace('\n', '<br>')|safe }}</p>
        </div>
        <div class="col-md-6">
            <h5>{{ tr('about_section4_title') }}</h5>
            <p>{{ tr('about_section4') }}</p>
            <h5>{{ tr('about_section5_title') }}</h5>
            <p>{{ tr('about_section5')|replace('\n', '<br>')|safe }}</p>
            <h5>{{ tr('about_section6_title') }}</h5>
            <p>{{ tr('about_section6')|replace('\n', '<br>')|safe }}</p>
        </div>
    </div>
    <hr>
    <p class="text-muted text-center">{{ tr('footer') }}</p>
</div>
{% endblock %}'''
}

# ---------- Flask init ----------
app = Flask(__name__)
app.jinja_loader = DictLoader(TEMPLATES)
app.secret_key = os.urandom(24)
UPLOAD_FOLDER = os.path.join('static', 'uploads')
os.makedirs(UPLOAD_FOLDER, exist_ok=True)
app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER

def hash_pw(pw):
    return hashlib.sha256(pw.encode()).hexdigest()

def is_valid_email(email):
    return re.match(r"^[^@\s]+@[^@\s]+\.[^@\s]+$", email) is not None

def tr(key):
    lang = session.get('language', 'ru')
    return TRANSLATIONS.get(lang, TRANSLATIONS['ru']).get(key, key)

def current_theme():
    theme_name = session.get('theme', 'light')
    return THEMES.get(theme_name, THEMES['light'])

@app.template_filter('regex_replace')
def regex_replace(s, find, replace):
    return re.sub(find, replace, s)

app.add_template_global(tr, 'tr')

# ---------- Routes ----------
@app.route('/')
def index():
    cars = []
    history = []
    if 'user' in session:
        cars = db_get_cars(session['user'])
        history = db_get_history(session['user'])
    return render_template("index.html", cars=cars, history=history,
                           theme=current_theme(), lang=session.get('language','ru'),
                           currency_symbol=CURRENCIES.get(session.get('currency','₽ RUB'),'₽'),
                           user=session.get('user'))

@app.route('/calculate', methods=['POST'])
def calculate():
    if 'user' not in session:
        return jsonify({'error': 'Войдите в систему'}), 401
    email = session['user']
    try:
        dist = float(request.form.get('distance', 0))
        price = float(request.form.get('price', 0))
    except ValueError:
        return jsonify({'error': tr('enter_numbers')}), 400
    if dist <= 0:
        return jsonify({'error': tr('distance_positive')}), 400
    if price <= 0:
        return jsonify({'error': 'Цена должна быть больше 0'}), 400

    mode = request.form.get('mode', 'consumption')
    car_name = request.form.get('car', tr('no_car'))
    currency = CURRENCIES.get(session.get('currency', '₽ RUB'), '₽')

    if mode == 'consumption':
        try:
            fuel = float(request.form.get('fuel', 0))
        except ValueError:
            return jsonify({'error': tr('enter_numbers')}), 400
        if fuel <= 0:
            return jsonify({'error': 'Расход топлива должен быть больше 0'}), 400
        consumption = (fuel / dist) * 100
        cost = fuel * price
    else:
        try:
            avg_input = float(request.form.get('avg_consumption', 0))
        except ValueError:
            return jsonify({'error': tr('enter_numbers')}), 400
        if avg_input <= 0:
            return jsonify({'error': tr('distance_positive2')}), 400
        consumption = avg_input
        fuel = (avg_input / 100) * dist
        cost = fuel * price

    entry = {
        'date': datetime.now().strftime("%d.%m.%Y %H:%M"),
        'car': car_name,
        'distance': dist,
        'fuel': round(fuel, 2),
        'price': price,
        'currency': currency,
        'consumption': round(consumption, 2),
        'cost': round(cost, 2)
    }
    db_add_history(email, entry)
    if car_name not in (tr('no_car'), '-- No car --'):
        db_update_car_consumption(email, car_name, round(consumption, 2))

    return jsonify({
        'consumption': round(consumption, 2),
        'fuel': round(fuel, 2),
        'cost': round(cost, 2),
        'mode': mode,
        'currency': currency
    })

@app.route('/chart_data')
def chart_data():
    if 'user' not in session:
        return jsonify({'labels':[], 'values':[]})
    email = session['user']
    car = request.args.get('car', '')
    days = int(request.args.get('days', 30))
    data = db_get_history_for_chart(email, car, days)
    labels = [d[0].strftime('%d.%m.%Y') for d in data]
    values = [d[1] for d in data]
    return jsonify({'labels': labels, 'values': values})

@app.route('/history_html')
def history_html():
    if 'user' not in session:
        return "<div class='card p-4 text-center'>" + tr('login_for_history') + "</div>"
    car = request.args.get('car', '')
    history = db_get_history(session['user'], car_name=car if car else None)
    return render_template("history_list.html",
                           history=history,
                           theme=current_theme(),
                           lang=session.get('language','ru'),
                           user=session['user'],
                           currency_symbol=CURRENCIES.get(session.get('currency','₽ RUB'),'₽'))

@app.route('/auth', methods=['GET', 'POST'])
def auth():
    if 'user' in session:
        return redirect(url_for('index'))
    if request.method == 'POST':
        form_type = request.form.get('form_type', 'login')
        email = request.form['email'].strip().lower()
        pw = request.form['password']
        if form_type == 'register':
            pw2 = request.form['password2']
            if not email or not pw:
                flash(tr('fill_all'), 'error')
            elif not is_valid_email(email):
                flash(tr('invalid_email'), 'error')
            elif len(pw) < 6:
                flash(tr('password_short'), 'error')
            elif pw != pw2:
                flash(tr('passwords_mismatch'), 'error')
            elif db_user_exists(email):
                flash(tr('user_exists'), 'error')
            else:
                db_create_user(email, hash_pw(pw))
                session['user'] = email
                session['theme'] = 'light'
                session['language'] = 'ru'
                session['currency'] = '₽ RUB'
                flash('Регистрация прошла успешно', 'success')
                return redirect(url_for('index'))
        else:
            if not is_valid_email(email):
                flash(tr('invalid_email'), 'error')
            else:
                stored = db_get_user(email)
                if stored and stored == hash_pw(pw):
                    session['user'] = email
                    settings = db_get_settings(email)
                    session['theme'] = settings['theme']
                    session['language'] = settings['language']
                    session['currency'] = settings['currency']
                    flash('Вы вошли', 'success')
                    return redirect(url_for('index'))
                else:
                    flash(tr('wrong_credentials'), 'error')
    return render_template("auth.html", theme=current_theme(), lang=session.get('language','ru'))

@app.route('/logout')
def logout():
    session.clear()
    return redirect(url_for('index'))

@app.route('/profile')
def profile():
    if 'user' not in session:
        flash(tr('login_for_history'), 'warning')
        return redirect(url_for('auth'))
    cars = db_get_cars(session['user'])
    car_stats = {}
    for car in cars:
        car_stats[car['name']] = db_get_car_stats(session['user'], car['name'])
    return render_template("profile.html", cars=cars, car_stats=car_stats,
                           theme=current_theme(), lang=session.get('language','ru'),
                           user=session['user'], currency_symbol=CURRENCIES.get(session.get('currency','₽ RUB'),'₽'))

@app.route('/add_car', methods=['POST'])
def add_car():
    if 'user' not in session:
        return redirect(url_for('auth'))
    if db_count_cars(session['user']) >= MAX_CARS_PER_USER:
        flash(tr('no_cars_limit'), 'error')
        return redirect(url_for('profile'))
    name = request.form['name'].strip()
    if not name:
        flash(tr('enter_name'), 'error')
        return redirect(url_for('profile'))
    photo = request.files.get('photo')
    photo_name = None
    if photo and photo.filename:
        filename = secure_filename(photo.filename)
        ext = os.path.splitext(filename)[1].lower()
        allowed = {'.jpg', '.jpeg', '.png', '.gif'}
        if ext in allowed:
            unique_name = f"car_{datetime.now().strftime('%Y%m%d%H%M%S')}{ext}"
            photo_path = os.path.join(app.config['UPLOAD_FOLDER'], unique_name)
            photo.save(photo_path)
            photo_name = unique_name
        else:
            flash('Разрешены только JPG, PNG, GIF', 'error')
            return render_profile_cars()
    try:
        db_add_car(session['user'], name, photo_name)
    except Exception as e:
        flash(tr('database_error'), 'error')
    return render_profile_cars()

def render_profile_cars():
    cars = db_get_cars(session['user'])
    car_stats = {}
    for car in cars:
        car_stats[car['name']] = db_get_car_stats(session['user'], car['name'])
    return render_template("profile_cars.html", cars=cars, car_stats=car_stats,
                           theme=current_theme(), lang=session.get('language','ru'),
                           user=session['user'], currency_symbol=CURRENCIES.get(session.get('currency','₽ RUB'),'₽'))

@app.route('/delete_car/<name>', methods=['POST'])
def delete_car(name):
    if 'user' not in session:
        return redirect(url_for('auth'))
    try:
        db_delete_car(session['user'], name)
    except:
        pass
    return render_profile_cars()

@app.route('/change_password', methods=['POST'])
def change_password():
    if 'user' not in session:
        return jsonify({'success': False, 'message': 'Войдите'}), 401
    old = request.form['old_password']
    new = request.form['new_password']
    new2 = request.form['new_password2']
    if not old or not new:
        return jsonify({'success': False, 'message': tr('fill_all')})
    if len(new) < 6:
        return jsonify({'success': False, 'message': tr('password_short')})
    if new != new2:
        return jsonify({'success': False, 'message': tr('passwords_mismatch')})
    stored = db_get_user(session['user'])
    if stored != hash_pw(old):
        return jsonify({'success': False, 'message': tr('old_password_wrong')})
    db_update_password(session['user'], hash_pw(new))
    return jsonify({'success': True, 'message': tr('password_changed')})

@app.route('/delete_account', methods=['POST'])
def delete_account():
    if 'user' not in session:
        return redirect(url_for('auth'))
    db_delete_user(session['user'])
    session.clear()
    flash('Аккаунт удалён', 'success')
    return redirect(url_for('index'))

@app.route('/clear_history', methods=['POST'])
def clear_history():
    if 'user' not in session:
        return '', 401
    db_clear_history(session['user'])
    return '', 200

@app.route('/delete_history/<int:entry_id>', methods=['POST'])
def delete_history(entry_id):
    if 'user' not in session:
        return '', 401
    db_delete_history_entry(session['user'], entry_id)
    return '', 200

@app.route('/settings', methods=['GET', 'POST'])
def settings():
    if 'user' not in session:
        flash(tr('login_for_history'), 'warning')
        return redirect(url_for('auth'))
    if request.method == 'POST':
        theme = session.get('theme', 'light')
        lang = request.form.get('language', 'ru')
        currency = request.form.get('currency', '₽ RUB')
        session['language'] = lang
        session['currency'] = currency
        db_update_settings(session['user'], theme, lang, currency)
        flash('Настройки сохранены', 'success')
        return redirect(url_for('settings'))
    user_settings = db_get_settings(session['user'])
    return render_template("settings.html", settings=user_settings,
                           currencies=CURRENCIES, theme=current_theme(),
                           lang=session.get('language','ru'), user=session.get('user'))

@app.route('/about')
def about():
    return render_template("about.html", theme=current_theme(), lang=session.get('language','ru'),
                           user=session.get('user'))

if __name__ == '__main__':
    if init_db_pool():
        create_tables()
        app.run(debug=True)
    else:
        print("Не удалось подключиться к PostgreSQL. Проверьте настройки DB_CONFIG.")
