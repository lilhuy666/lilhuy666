import os, hashlib, re
from datetime import datetime
from contextlib import contextmanager

from flask import Flask, render_template, request, redirect, url_for, session, flash, jsonify
from werkzeug.utils import secure_filename
import psycopg2
from psycopg2 import pool
from jinja2 import DictLoader

# Пробуем Pillow для сжатия фото, но если нет — работаем без него
try:
    from PIL import Image
    PILLOW_AVAILABLE = True
except ImportError:
    PILLOW_AVAILABLE = False

# ═════════════════════════════════════════════════════════════════════
# НАСТРОЙКИ ПОДКЛЮЧЕНИЯ К БАЗЕ ДАННЫХ
# ═════════════════════════════════════════════════════════════════════
DB_CONFIG = {
    'dbname': 'fuel_calc',
    'user': 'postgres',
    'password': '12345',
    'host': 'localhost',
    'port': 5432,
}

connection_pool = None
ALLOWED_EXTENSIONS = {'jpg', 'jpeg', 'png', 'gif', 'bmp', 'webp', 'tiff', 'ico'}
MAX_IMAGE_SIZE = (1200, 1200)

# ═════════════════════════════════════════════════════════════════════
# ИНИЦИАЛИЗАЦИЯ БАЗЫ ДАННЫХ
# ═════════════════════════════════════════════════════════════════════

def init_db_pool():
    """Создаёт пул соединений с PostgreSQL (от 1 до 20 соединений)"""
    global connection_pool
    try:
        connection_pool = pool.SimpleConnectionPool(1, 20, **DB_CONFIG)
        print("✓ Пул соединений с БД создан")
        return True
    except Exception as e:
        print(f"✗ Ошибка подключения к БД: {e}")
        return False


@contextmanager
def get_db_connection():
    """
    Контекстный менеджер: берёт соединение из пула,
    при ошибке делает rollback, в конце возвращает в пул.
    """
    conn = connection_pool.getconn()
    try:
        yield conn
    except Exception:
        conn.rollback()
        raise
    finally:
        connection_pool.putconn(conn)


def db_exec(sql, params=None, fetch=False):
    """
    Единая функция для всех SQL-запросов.
    sql    – строка запроса
    params – параметры (кортеж)
    fetch  – True, если нужен результат SELECT
    """
    with get_db_connection() as conn:
        with conn.cursor() as cur:
            cur.execute(sql, params)
            if fetch:
                return cur.fetchall()
            conn.commit()


def create_tables():
    """Создаёт таблицы, если их ещё нет"""
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
    print("✓ Таблицы БД проверены/созданы")


# ═════════════════════════════════════════════════════════════════════
# РАБОТА С ИЗОБРАЖЕНИЯМИ
# ═════════════════════════════════════════════════════════════════════

def allowed_file(filename):
    """Проверяет расширение файла"""
    return '.' in filename and filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS


def process_image(image_path):
    """Сжимает большое изображение для экономии места на диске"""
    if not PILLOW_AVAILABLE:
        return True
    try:
        with Image.open(image_path) as img:
            if img.mode in ('RGBA', 'P'):
                img = img.convert('RGB')
            if img.size[0] > MAX_IMAGE_SIZE[0] or img.size[1] > MAX_IMAGE_SIZE[1]:
                img.thumbnail(MAX_IMAGE_SIZE, Image.Resampling.LANCZOS)
            img.save(image_path, optimize=True, quality=85)
        return True
    except Exception as e:
        print(f"Ошибка сжатия фото: {e}")
        return False


def save_car_photo(photo_file, old_photo_path=None):
    """
    Сохраняет фото авто в static/uploads/.
    Если передан old_photo_path — удаляет старый файл.
    Возвращает имя нового файла или None.
    """
    if not photo_file or not allowed_file(photo_file.filename):
        return None
    if old_photo_path:
        old_path = os.path.join('static', 'uploads', old_photo_path)
        if os.path.exists(old_path):
            os.remove(old_path)
    # Генерируем уникальное имя: car_датавремя_хэш.расширение
    safe_name = secure_filename(photo_file.filename)
    name_part, ext = os.path.splitext(safe_name)
    hash_part = hashlib.md5(name_part.encode()).hexdigest()[:8]
    new_name = f"car_{datetime.now().strftime('%Y%m%d%H%M%S')}_{hash_part}{ext}"
    file_path = os.path.join('static', 'uploads', new_name)
    photo_file.save(file_path)
    process_image(file_path)
    return new_name


# ═════════════════════════════════════════════════════════════════════
# CRUD — ОПЕРАЦИИ С БАЗОЙ ДАННЫХ
# ═════════════════════════════════════════════════════════════════════

# --- Пользователи ---

def db_create_user(email, pw_hash):
    db_exec("INSERT INTO users VALUES (%s,%s)", (email, pw_hash))
    db_exec("INSERT INTO settings (email) VALUES (%s)", (email,))

def db_get_user(email):
    rows = db_exec("SELECT password FROM users WHERE email=%s", (email,), fetch=True)
    return rows[0][0] if rows else None

def db_user_exists(email):
    return bool(db_exec("SELECT 1 FROM users WHERE email=%s", (email,), fetch=True))

def db_update_password(email, pw_hash):
    db_exec("UPDATE users SET password=%s WHERE email=%s", (pw_hash, email))

def db_delete_user(email):
    # Удаляем файлы фото перед каскадным удалением
    for (photo,) in db_exec("SELECT photo_path FROM cars WHERE email=%s", (email,), fetch=True):
        if photo:
            fp = os.path.join('static', 'uploads', photo)
            if os.path.exists(fp):
                os.remove(fp)
    db_exec("DELETE FROM users WHERE email=%s", (email,))

# --- Автомобили ---

def db_get_cars(email):
    rows = db_exec(
        "SELECT name, photo_path, avg_consumption FROM cars WHERE email=%s ORDER BY id",
        (email,), fetch=True
    )
    return [{"name": r[0], "photo": r[1], "avg_consumption": r[2]} for r in rows]

def db_add_car(email, name, photo=None):
    db_exec("INSERT INTO cars (email,name,photo_path) VALUES (%s,%s,%s)", (email, name, photo))

def db_update_car(email, old_name, new_name, photo=None):
    if photo:
        db_exec("UPDATE cars SET name=%s,photo_path=%s WHERE email=%s AND name=%s",
                (new_name, photo, email, old_name))
    else:
        db_exec("UPDATE cars SET name=%s WHERE email=%s AND name=%s", (new_name, email, old_name))

def db_delete_car(email, name):
    rows = db_exec("SELECT photo_path FROM cars WHERE email=%s AND name=%s", (email, name), fetch=True)
    if rows and rows[0][0]:
        fp = os.path.join('static', 'uploads', rows[0][0])
        if os.path.exists(fp):
            os.remove(fp)
    db_exec("DELETE FROM cars WHERE email=%s AND name=%s", (email, name))

def db_update_avg(email, car_name):
    """Пересчитывает и сохраняет средний расход авто по истории"""
    rows = db_exec(
        "SELECT AVG(consumption) FROM history WHERE email=%s AND car_name=%s AND consumption IS NOT NULL",
        (email, car_name), fetch=True
    )
    if rows and rows[0][0]:
        db_exec("UPDATE cars SET avg_consumption=%s WHERE email=%s AND name=%s",
                (round(rows[0][0], 2), email, car_name))

# --- История ---

def db_add_history(email, entry):
    db_exec(
        """INSERT INTO history (email,car_name,distance,fuel,price,currency,consumption,cost,date)
           VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s)""",
        (email, entry['car'], entry['distance'], entry['fuel'], entry['price'],
         entry.get('currency'), entry['consumption'], entry['cost'],
         datetime.strptime(entry['date'], "%d.%m.%Y %H:%M"))
    )

def db_get_history(email, car_name=None):
    query = """
        SELECT h.id, h.car_name, h.distance, h.fuel, h.price, h.currency,
               h.consumption, h.cost, TO_CHAR(h.date,'DD.MM.YYYY HH24:MI'), c.photo_path
        FROM history h LEFT JOIN cars c ON h.email=c.email AND h.car_name=c.name
        WHERE h.email=%s
    """
    params = [email]
    if car_name and car_name not in ("— Без авто —", "— No car —"):
        query += " AND h.car_name=%s"
        params.append(car_name)
    query += " ORDER BY h.date DESC"
    rows = db_exec(query, params, fetch=True)
    return [
        {'id': r[0], 'car': r[1], 'distance': r[2], 'fuel': r[3], 'price': r[4],
         'currency': r[5], 'consumption': r[6], 'cost': r[7], 'date': r[8], 'photo': r[9]}
        for r in rows
    ]

def db_clear_history(email):
    db_exec("DELETE FROM history WHERE email=%s", (email,))

def db_delete_history_entry(email, eid):
    if db_exec("SELECT 1 FROM history WHERE id=%s AND email=%s", (eid, email), fetch=True):
        db_exec("DELETE FROM history WHERE email=%s AND id=%s", (email, eid))

# --- Настройки и статистика ---

def db_get_settings(email):
    rows = db_exec("SELECT theme,language,currency FROM settings WHERE email=%s", (email,), fetch=True)
    if rows:
        return {'theme': rows[0][0], 'language': rows[0][1], 'currency': rows[0][2]}
    return {'theme': 'light', 'language': 'ru', 'currency': '₽ RUB'}

def db_update_settings(email, theme, lang, cur):
    db_exec("UPDATE settings SET theme=%s,language=%s,currency=%s WHERE email=%s", (theme, lang, cur, email))

def db_get_car_stats(email, car_name):
    rows = db_exec(
        "SELECT SUM(distance),SUM(cost),AVG(consumption) FROM history WHERE email=%s AND car_name=%s",
        (email, car_name), fetch=True
    )
    if rows and rows[0][0]:
        return {'total_distance': round(rows[0][0], 1), 'total_cost': round(rows[0][1], 2),
                'avg_consumption': round(rows[0][2], 2)}
    return {'total_distance': 0, 'total_cost': 0, 'avg_consumption': 0}


# ═════════════════════════════════════════════════════════════════════
# ЛОКАЛИЗАЦИЯ (русский + английский)
# ═════════════════════════════════════════════════════════════════════

TRANSLATIONS = {
    "ru": {
        "app_title": "FullCalcPro",
        "calculator": "Калькулятор",
        "profile": "Профиль",
        "login_register": "Вход / Регистрация",
        "history": "История",
        "about": "О приложении",
        "settings": "Настройки",
        "no_car": "-- Без авто --",
        "car": "Автомобиль",
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
        "edit": "Изменить",
        "copy": "Копировать",
        "car_name": "Название автомобиля",
        "save": "Сохранить",
        "cancel": "Отмена",
        "enter_name": "Введите название",
        "add_car_title": "Добавить автомобиль",
        "edit_car_title": "Изменить автомобиль",
        "history_title": "История расчётов",
        "login_for_history": "Войдите в аккаунт для просмотра истории",
        "clear_all": "Очистить всё",
        "history_empty": "История пуста",
        "copied": "Скопировано!",
        "delete_confirm": "Удалить?",
        "delete_car_confirm": "Удалить этот автомобиль?",
        "delete_history_confirm": "Удалить запись?",
        "delete_history_msg": "Вы уверены?",
        "clear_history_confirm": "Очистить историю?",
        "clear_history_msg": "Удалить всю историю?",
        "delete_account_confirm": "Удалить аккаунт?",
        "delete_account_msg": "Все данные будут удалены безвозвратно!",
        "settings_title": "Настройки",
        "currency_setting": "Валюта",
        "language_setting": "Язык",
        "error": "Ошибка",
        "enter_numbers": "Введите корректные числа!",
        "distance_positive": "Расстояние должно быть больше 0!",
        "distance_positive2": "Расстояние и расход должны быть больше 0!",
        "fuel_unit": "л",
        "km": "км",
        "about_title": "О приложении",
        "version": "Версия 1.0  •  2026",
        "footer": "© 2026 FullCalcPro",
        "database_error": "Ошибка подключения к базе данных",
        "old_password_wrong": "Неверный старый пароль",
        "date": "Дата",
        "show_more": "Показать ещё",
        "show_less": "Свернуть",
        "password_changed": "Пароль успешно изменён!",
        "total_distance": "Общий пробег",
        "total_cost": "Общие затраты",
        "clear_form_confirm": "Очистить форму?",
        "clear_form_msg": "Вы уверены?",
        "login_required": "Войдите в систему для сохранения расчётов",
        "lock_history": "Войдите для просмотра истории",
        # Контент для страницы "О приложении"
        "about_intro": "FullCalcPro — это профессиональный инструмент для точного учёта расхода топлива и затрат на поездки. Приложение создано для водителей, которые хотят контролировать свои расходы и вести статистику по каждому автомобилю.",
        "about_features_title": "Основные возможности",
        "about_feature_1": "Два режима расчёта — по фактически потраченному топливу или по среднему расходу. Выбирайте нужный режим в зависимости от того, какие данные у вас есть.",
        "about_feature_2": "Гараж автомобилей — добавляйте неограниченное количество машин с фотографиями. Для каждого авто автоматически рассчитывается средний расход на основе всей истории поездок.",
        "about_feature_3": "Полная история — каждый расчёт сохраняется в базе данных. В любой момент можно посмотреть: дату, пробег, объём топлива, цену, расход и итоговую стоимость. Историю можно фильтровать по автомобилю.",
        "about_feature_4": "Статистика по авто — общий пробег, общие затраты на топливо и средний расход за всё время. Наглядно видно, какая машина экономичнее.",
        "about_feature_5": "Мультивалютность — поддержка рубля (₽), доллара ($) и евро (€). Переключается в настройках в один клик.",
        "about_feature_6": "Двуязычный интерфейс — русский и английский языки. Переключение без перезагрузки страницы.",
        "about_feature_7": "Копирование данных — любой расчёт или статистику авто можно скопировать в буфер обмена одним нажатием.",
        "about_security_title": "Безопасность",
        "about_security_1": "Пароли хэшируются алгоритмом SHA-256 и не хранятся в открытом виде.",
        "about_security_2": "Все данные хранятся в надёжной базе PostgreSQL на вашем собственном сервере.",
        "about_security_3": "Никакие данные не передаются третьим лицам — приложение работает полностью локально.",
        "about_tech_title": "Технические детали",
        "about_tech_1": "Бэкенд: Python + Flask",
        "about_tech_2": "База данных: PostgreSQL",
        "about_tech_3": "Интерфейс: современный стеклянный дизайн с анимациями",
        "about_tech_4": "Адаптивная вёрстка — удобно пользоваться и с компьютера, и с телефона",
    },
    "en": {
        "app_title": "FullCalcPro",
        "calculator": "Calculator",
        "profile": "Profile",
        "login_register": "Login / Register",
        "history": "History",
        "about": "About",
        "settings": "Settings",
        "no_car": "-- No car --",
        "car": "Vehicle",
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
        "password_short": "Password too short (min. 6)",
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
        "edit": "Edit",
        "copy": "Copy",
        "car_name": "Vehicle name",
        "save": "Save",
        "cancel": "Cancel",
        "enter_name": "Enter name",
        "add_car_title": "Add vehicle",
        "edit_car_title": "Edit vehicle",
        "history_title": "Calculation history",
        "login_for_history": "Log in to view history",
        "clear_all": "Clear all",
        "history_empty": "History is empty",
        "copied": "Copied!",
        "delete_confirm": "Delete?",
        "delete_car_confirm": "Delete this vehicle?",
        "delete_history_confirm": "Delete entry?",
        "delete_history_msg": "Are you sure?",
        "clear_history_confirm": "Clear history?",
        "clear_history_msg": "Delete all history?",
        "delete_account_confirm": "Delete account?",
        "delete_account_msg": "All data will be permanently deleted!",
        "settings_title": "Settings",
        "currency_setting": "Currency",
        "language_setting": "Language",
        "error": "Error",
        "enter_numbers": "Enter valid numbers!",
        "distance_positive": "Distance must be greater than 0!",
        "distance_positive2": "Distance and Avg must be greater than 0!",
        "fuel_unit": "L",
        "km": "km",
        "about_title": "About",
        "version": "Version 1.0  •  2026",
        "footer": "© 2026 FullCalcPro",
        "database_error": "Database connection error",
        "old_password_wrong": "Current password is incorrect",
        "date": "Date",
        "show_more": "Show more",
        "show_less": "Show less",
        "password_changed": "Password changed successfully!",
        "total_distance": "Total distance",
        "total_cost": "Total cost",
        "clear_form_confirm": "Clear form?",
        "clear_form_msg": "Are you sure?",
        "login_required": "Log in to save calculations",
        "lock_history": "Log in to view history",
        "about_intro": "FullCalcPro is a professional tool for accurate fuel consumption tracking and trip cost calculation. Built for drivers who want to control expenses and maintain statistics for each vehicle.",
        "about_features_title": "Key Features",
        "about_feature_1": "Two calculation modes — by actual fuel used or by average consumption. Choose the mode that fits your available data.",
        "about_feature_2": "Vehicle garage — add unlimited cars with photos. Average consumption is automatically calculated for each vehicle based on trip history.",
        "about_feature_3": "Complete history — every calculation is saved in the database. View date, distance, fuel volume, price, consumption and total cost anytime. Filter by vehicle.",
        "about_feature_4": "Vehicle statistics — total distance, total fuel costs and average consumption over time. See which car is most economical.",
        "about_feature_5": "Multi-currency — supports ruble (₽), dollar ($) and euro (€). Switch in settings with one click.",
        "about_feature_6": "Bilingual interface — Russian and English. Switch without page reload.",
        "about_feature_7": "Data copy — copy any calculation or vehicle statistics to clipboard with a single tap.",
        "about_security_title": "Security",
        "about_security_1": "Passwords are hashed with SHA-256 algorithm and never stored in plain text.",
        "about_security_2": "All data is stored in a reliable PostgreSQL database on your own server.",
        "about_security_3": "No data is transmitted to third parties — the application runs completely locally.",
        "about_tech_title": "Technical Details",
        "about_tech_1": "Backend: Python + Flask",
        "about_tech_2": "Database: PostgreSQL",
        "about_tech_3": "Interface: modern glassmorphism design with animations",
        "about_tech_4": "Responsive layout — works great on both desktop and mobile",
    }
}

CURRENCIES = {"₽ RUB": "₽", "$ USD": "$", "€ EUR": "€"}


# ═════════════════════════════════════════════════════════════════════
# ВСПОМОГАТЕЛЬНЫЕ ФУНКЦИИ
# ═════════════════════════════════════════════════════════════════════

def tr(key):
    """Возвращает перевод ключа на текущий язык сессии"""
    lang = session.get('language', 'ru')
    return TRANSLATIONS.get(lang, TRANSLATIONS['ru']).get(key, key)

def hash_pw(pw):
    return hashlib.sha256(pw.encode()).hexdigest()

def is_valid_email(e):
    return re.match(r"^[^@\s]+@[^@\s]+\.[^@\s]+$", e) is not None

def render_profile_cars():
    """Рендерит блок с авто для AJAX-обновления в профиле"""
    try:
        cars = db_get_cars(session['user'])
        car_stats = {c['name']: db_get_car_stats(session['user'], c['name']) for c in cars}
    except:
        cars, car_stats = [], {}
    return render_template("profile_cars.html", cars=cars, car_stats=car_stats,
                           lang=session.get('language', 'ru'), user=session['user'],
                           currency_symbol=CURRENCIES.get(session.get('currency', '₽ RUB'), '₽'))


# ═════════════════════════════════════════════════════════════════════
# HTML-ШАБЛОНЫ (хранятся в словаре, рендерятся через DictLoader)
# ═════════════════════════════════════════════════════════════════════

TEMPLATES = {

    "base.html": r'''<!DOCTYPE html><html lang="{{ lang }}">
<head><meta charset="UTF-8"><meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>{{ tr('app_title') }}</title>
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
<style>
    body{background-image:url('{{ url_for('static', filename='images/background.jpg') }}');background-size:cover;background-position:center;background-attachment:fixed;min-height:100vh;font-family:'Inter',sans-serif;font-size:15px}
    body::before{content:'';position:fixed;top:0;left:0;right:0;bottom:0;background:linear-gradient(135deg,rgba(20,30,48,0.85),rgba(36,59,85,0.75));z-index:-1}
    *{transition:all 0.2s ease}
    .navbar{background:rgba(255,255,255,0.15)!important;backdrop-filter:blur(16px);border-bottom:1px solid rgba(255,255,255,0.2);padding:12px 0}
    .navbar-brand{font-weight:700;font-size:1.5rem;color:white!important;text-decoration:none}
    .logo-img{width:36px;height:36px;border-radius:10px;object-fit:cover;margin-right:10px}
    .nav-link{font-weight:500;padding:8px 18px!important;color:rgba(255,255,255,0.85)!important;font-size:0.9rem;border-radius:12px}
    .nav-link:hover{background:rgba(255,255,255,0.2);color:white!important}
    .card,.list-group-item,.modal-content,.lock-overlay{background:rgba(255,255,255,0.2);backdrop-filter:blur(12px);border:1px solid rgba(255,255,255,0.25);border-radius:20px;box-shadow:0 8px 32px rgba(0,0,0,0.1);color:white}
    .list-group-item{margin-bottom:10px;border-radius:16px!important}
    .list-group-item:hover{background:rgba(255,255,255,0.3);transform:translateY(-2px)}
    .btn{border-radius:12px;font-weight:600;font-size:0.85rem;padding:8px 16px}
    .btn-primary{background:linear-gradient(135deg,#0071e3,#005bb5);border:none;color:white!important}
    .btn-outline-secondary{border:1px solid rgba(255,255,255,0.4);color:white;background:rgba(255,255,255,0.1)}
    .btn-outline-secondary:hover{background:rgba(255,255,255,0.25);color:white}
    .btn-outline-danger{border:1px solid rgba(255,59,48,0.6);color:#ff6b6b;background:rgba(255,59,48,0.1)}
    .btn-outline-danger:hover{background:rgba(255,59,48,0.8);color:white}
    .form-control,.form-select{background:rgba(255,255,255,0.15);color:white;border:1px solid rgba(255,255,255,0.25);border-radius:12px;padding:10px 14px}
    .form-control:focus,.form-select:focus{border-color:rgba(255,255,255,0.6);box-shadow:0 0 0 3px rgba(0,113,227,0.3);background:rgba(255,255,255,0.2);color:white}
    .form-label{color:white;font-weight:600}.form-select option{background:#1a2a3a;color:white}
    .alert{border-radius:16px;background:rgba(255,255,255,0.25);backdrop-filter:blur(12px);color:white}
    .alert-success{background:rgba(52,199,89,0.25)}.alert-danger{background:rgba(255,59,48,0.25)}
    h1,h2,h3,h4,h5,h6{color:white;font-weight:700}.text-muted{color:rgba(255,255,255,0.7)!important}
    .mode-selector{display:flex;gap:12px;margin-bottom:20px}
    .mode-btn{flex:1;text-align:center;padding:12px;border-radius:12px;border:1px solid rgba(255,255,255,0.3);background:rgba(255,255,255,0.1);color:white;cursor:pointer;font-weight:600}
    .mode-btn.active{border-color:white;background:rgba(255,255,255,0.25)}
    .modal-content{background:rgba(30,40,50,0.95);backdrop-filter:blur(16px);border-radius:24px}
    .modal-header{border-bottom:1px solid rgba(255,255,255,0.15)}.modal-footer{border-top:1px solid rgba(255,255,255,0.15)}
    .btn-close{filter:brightness(0) invert(1)}
    .car-photo-small{width:56px;height:56px;object-fit:cover;border-radius:14px;border:2px solid rgba(255,255,255,0.3)}
    .car-photo-medium{height:160px;object-fit:cover;border-radius:16px;border:2px solid rgba(255,255,255,0.3)}
    .lock-overlay{min-height:200px;display:flex;align-items:center;justify-content:center;background:rgba(0,0,0,0.4);backdrop-filter:blur(12px);border-radius:20px}
    .lock-content{text-align:center;padding:2rem;color:white}
    .badge{border-radius:30px;padding:6px 14px;font-weight:600;font-size:0.8rem;background:linear-gradient(135deg,#0071e3,#005bb5);color:white}
    footer{margin-top:4rem;text-align:center;padding:2rem;font-size:0.85rem;color:rgba(255,255,255,0.6)}
    .card{animation:fadeInUp 0.4s ease-out}@keyframes fadeInUp{from{opacity:0;transform:translateY(20px)}to{opacity:1;transform:translateY(0)}}
    a{color:rgba(255,255,255,0.9);text-decoration:none}a:hover{color:white}
    .result-alert{font-size:1.2rem;font-weight:600;text-align:center}
    .car-photo-preview{max-width:300px;max-height:200px;border-radius:16px;border:2px solid rgba(255,255,255,0.3);object-fit:cover}
    .car-actions .btn{width:100%;margin-bottom:4px}
    .about-section{margin-bottom:2rem}
    .about-section h5{color:#fff;margin-bottom:0.8rem;font-size:1.2rem}
    .about-section p,.about-section li{color:rgba(255,255,255,0.85);font-size:0.95rem;line-height:1.6}
    .about-section ul{list-style:none;padding-left:0}
    .about-section ul li::before{content:"•";color:#0071e3;font-weight:bold;margin-right:8px}
    .feature-grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(280px,1fr));gap:1.5rem;margin:1.5rem 0}
    .feature-card{background:rgba(255,255,255,0.1);border-radius:16px;padding:1.2rem;border:1px solid rgba(255,255,255,0.15)}
    .feature-card i{font-size:1.5rem;color:#0071e3;margin-bottom:0.5rem}
    .feature-card h6{font-size:1rem;margin-bottom:0.4rem}
    .feature-card p{font-size:0.85rem;color:rgba(255,255,255,0.75);margin:0}
</style></head>
<body>
<nav class="navbar navbar-expand-lg sticky-top"><div class="container">
<a class="navbar-brand" href="/"><img src="{{ url_for('static', filename='logo.png') }}" class="logo-img" alt="Logo" onerror="this.style.display='none'">{{ tr('app_title') }}</a>
<button class="navbar-toggler" data-bs-toggle="collapse" data-bs-target="#navbarNav" style="border:1px solid rgba(255,255,255,0.3);background:rgba(255,255,255,0.1)"><span class="navbar-toggler-icon" style="filter:brightness(0) invert(1)"></span></button>
<div class="collapse navbar-collapse" id="navbarNav"><ul class="navbar-nav ms-auto">
<li class="nav-item"><a class="nav-link" href="/">{{ tr('calculator') }}</a></li>
{% if user %}<li class="nav-item"><a class="nav-link" href="/profile">{{ tr('profile') }}</a></li>
<li class="nav-item"><a class="nav-link" href="/settings">{{ tr('settings') }}</a></li>
{% else %}<li class="nav-item"><a class="nav-link" href="/auth">{{ tr('login_register') }}</a></li>{% endif %}
<li class="nav-item"><a class="nav-link" href="/about">{{ tr('about') }}</a></li></ul></div></div></nav>
<div class="container mt-4">
{% with messages = get_flashed_messages(with_categories=true) %}{% if messages %}
{% for cat, msg in messages %}<div class="alert alert-{{ cat }} alert-dismissible fade show">{{ msg }}<button class="btn-close" data-bs-dismiss="alert"></button></div>{% endfor %}
{% endif %}{% endwith %}
{% block content %}{% endblock %}</div>
<div class="modal fade" id="confirmModal"><div class="modal-dialog modal-dialog-centered"><div class="modal-content">
<div class="modal-header"><h5 class="modal-title fw-bold" id="confirmModalTitle"></h5><button class="btn-close" data-bs-dismiss="modal"></button></div>
<div class="modal-body" id="confirmModalBody" style="color:white"></div>
<div class="modal-footer"><button class="btn btn-outline-secondary" data-bs-dismiss="modal">{{ tr('cancel') }}</button><button class="btn btn-danger" id="confirmModalBtn"></button></div></div></div></div>
<footer>{{ tr('footer') }}</footer>
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
<script>
function togglePassword(b){let i=b.parentElement.querySelector('input'),ic=b.querySelector('i');i.type=i.type==='password'?'text':'password';ic.classList.toggle('fa-eye');ic.classList.toggle('fa-eye-slash')}
let confirmCallback=null;const confirmModal=new bootstrap.Modal(document.getElementById('confirmModal'));
function showConfirm(t,m,b,c){document.getElementById('confirmModalTitle').textContent=t;document.getElementById('confirmModalBody').textContent=m;document.getElementById('confirmModalBtn').textContent=b;confirmCallback=c;confirmModal.show()}
document.getElementById('confirmModalBtn').addEventListener('click',()=>{if(confirmCallback)confirmCallback();confirmModal.hide();confirmCallback=null});
</script>{% block scripts %}{% endblock %}</body></html>''',

    # ── Главная (калькулятор) ──
    "index.html": r'''{% extends "base.html" %}{% block content %}
<div style="max-width:540px;margin:0 auto">
<h2 class="mb-4 text-center">{{ tr('calculator') }}</h2>
{% if not user %}<div class="alert alert-info mb-4 text-center">{{ tr('login_required') }} <a href="/auth" style="color:white;text-decoration:underline">{{ tr('login_register') }}</a></div>{% endif %}
<div class="card p-4">
<div class="mb-3"><label class="form-label fw-semibold">{{ tr('car') }}</label>
<select id="carSelect" class="form-select"><option value="">{{ tr('no_car') }}</option>
{% if user %}{% for car in cars %}<option value="{{ car.name }}" data-photo="{% if car.photo %}{{ url_for('static', filename='uploads/' + car.photo) }}{% endif %}" data-avg="{{ car.avg_consumption or '' }}">{{ car.name }}</option>{% endfor %}{% endif %}
</select></div>
<div class="text-center mb-3"><img id="carPhotoPreview" src="" class="car-photo-preview" style="display:none"></div>
<form id="calcForm">
<div class="mode-selector"><button type="button" class="mode-btn active" data-mode="consumption">{{ tr('calc_mode_consumption') }}</button><button type="button" class="mode-btn" data-mode="cost">{{ tr('calc_mode_cost') }}</button></div>
<input type="hidden" name="mode" value="consumption" id="modeInput">
<div class="mb-3"><label class="form-label fw-semibold">{{ tr('distance_label') }}, км</label><input type="number" step="any" name="distance" id="distance" class="form-control" required></div>
<div id="fuelField" class="mb-3"><label class="form-label fw-semibold">{{ tr('fuel_used') }}, л</label><input type="number" step="any" name="fuel" id="fuel" class="form-control"></div>
<div id="avgField" class="mb-3" style="display:none"><label class="form-label fw-semibold">{{ tr('avg_consumption_label') }}, л/100км</label><input type="number" step="any" name="avg_consumption" id="avgConsumption" class="form-control"></div>
<div class="mb-3"><label class="form-label fw-semibold">{{ tr('fuel_price') }}, {{ currency_symbol }}/л</label><input type="number" step="any" name="price" id="price" class="form-control" required></div>
<div class="d-flex justify-content-center gap-2"><button type="submit" class="btn btn-primary px-5" id="calculateBtn">{{ tr('calculate') }}</button><button type="button" id="clearBtn" class="btn btn-outline-secondary px-4">{{ tr('clear') }}</button></div>
</form><div id="result" class="mt-3 alert result-alert" style="display:none"></div></div>
<div class="mt-5"><h4 class="fw-bold mb-3 text-center">{{ tr('history_title') }}</h4>
{% if user %}<div id="historyContainer">{% include 'history_list.html' %}</div>
{% else %}<div class="lock-overlay"><div class="lock-content"><i class="fas fa-lock fa-3x mb-3"></i><p>{{ tr('lock_history') }}</p><a href="/auth" class="btn btn-primary">{{ tr('login_register') }}</a></div></div>{% endif %}
</div></div>{% endblock %}
{% block scripts %}<script>
const INIT=10;let current=INIT,isLoggedIn={{ 'true' if user else 'false' }},isCalc=false;
document.querySelectorAll('.mode-btn').forEach(b=>{b.addEventListener('click',function(){document.querySelectorAll('.mode-btn').forEach(x=>x.classList.remove('active'));this.classList.add('active');let m=this.dataset.mode;document.getElementById('modeInput').value=m;document.getElementById('fuelField').style.display=m==='consumption'?'block':'none';document.getElementById('avgField').style.display=m==='consumption'?'none':'block';if(m==='cost'){let o=document.getElementById('carSelect'),v=o.options[o.selectedIndex].getAttribute('data-avg');if(v)document.getElementById('avgConsumption').value=v}})});
document.getElementById('carSelect').addEventListener('change',function(){let o=this.options[this.selectedIndex],u=o.getAttribute('data-photo'),img=document.getElementById('carPhotoPreview');img.style.display=u&&u!=='None'?'block':'none';if(u&&u!=='None')img.src=u;if(document.getElementById('modeInput').value==='cost'){let v=o.getAttribute('data-avg');if(v)document.getElementById('avgConsumption').value=v}if(isLoggedIn)loadHistory(this.value)});
document.getElementById('calcForm').addEventListener('submit',async function(e){e.preventDefault();if(isCalc)return;let btn=document.getElementById('calculateBtn');isCalc=true;btn.disabled=true;btn.innerHTML='<span class="spinner-border spinner-border-sm me-1"></span>';let fd=new FormData(this);fd.append('car',isLoggedIn?document.getElementById('carSelect').value:'{{ tr("no_car") }}');try{let r=await fetch('/calculate',{method:'POST',body:fd});let d=await r.json(),div=document.getElementById('result');if(d.error){div.className='mt-3 alert alert-danger result-alert';div.innerText=d.error}else{div.className='mt-3 alert alert-success result-alert';div.innerText=d.mode==='consumption'?`{{ tr('fuel_consumption') }}: ${d.consumption} {{ tr('fuel_unit') }}/100{{ tr('km') }}, {{ tr('cost') }}: ${d.cost} ${d.currency}`:`{{ tr('trip_cost') }}: ${d.cost} ${d.currency}`}div.style.display='block';if(isLoggedIn)loadHistory(document.getElementById('carSelect').value)}catch(err){console.error(err)}finally{isCalc=false;btn.disabled=false;btn.innerHTML='{{ tr("calculate") }}'}});
document.getElementById('clearBtn').addEventListener('click',()=>{showConfirm('{{ tr("clear_form_confirm") }}','{{ tr("clear_form_msg") }}','{{ tr("clear") }}',()=>{document.getElementById('calcForm').reset();document.getElementById('result').style.display='none';document.getElementById('fuelField').style.display='block';document.getElementById('avgField').style.display='none';document.querySelectorAll('.mode-btn').forEach(b=>b.classList.remove('active'));document.querySelector('.mode-btn[data-mode="consumption"]').classList.add('active');document.getElementById('modeInput').value='consumption'})});
{% if user %}
async function loadHistory(car=''){try{let r=await fetch(`/history_html?car=${encodeURIComponent(car)}`);document.getElementById('historyContainer').innerHTML=await r.text();current=INIT;apply()}catch(e){}}
function apply(){let items=document.querySelectorAll('.history-item'),btn=document.getElementById('toggleHistoryBtn');items.forEach((it,i)=>{it.style.display=i<current?'':'none'});if(btn){if(items.length>current){btn.style.display='block';btn.textContent='{{ tr("show_more") }}'}else if(current>INIT){btn.style.display='block';btn.textContent='{{ tr("show_less") }}'}else btn.style.display='none'}}
function toggleHistory(){let items=document.querySelectorAll('.history-item');if(!items.length)return;current=current>=items.length?INIT:Math.min(current+INIT,items.length);apply()}
function clearAllHistory(){showConfirm('{{ tr("clear_history_confirm") }}','{{ tr("clear_history_msg") }}','{{ tr("clear_all") }}',async()=>{await fetch('/clear_history',{method:'POST'});loadHistory(document.getElementById('carSelect').value)})}
function deleteHistoryEntry(id){showConfirm('{{ tr("delete_history_confirm") }}','{{ tr("delete_history_msg") }}','{{ tr("delete") }}',async()=>{await fetch('/delete_history/'+id,{method:'POST'});loadHistory(document.getElementById('carSelect').value)})}
function copyEntry(btn){navigator.clipboard.writeText(btn.getAttribute('data-text')).then(()=>{let o=btn.innerHTML;btn.innerHTML='<i class="fas fa-check"></i>';setTimeout(()=>btn.innerHTML=o,1500)}).catch(()=>alert('{{ tr("error") }}'))}
new MutationObserver(()=>apply()).observe(document.getElementById('historyContainer'),{childList:true,subtree:true});apply();
{% endif %}</script>{% endblock %}''',

    # ── История (AJAX-фрагмент) ──
    "history_list.html": r'''{% if user %}<div class="d-flex justify-content-end mb-3"><button class="btn btn-outline-danger" onclick="clearAllHistory()">{{ tr('clear_all') }}</button></div>
{% if history %}<div class="list-group" id="historyList">{% for e in history %}
<div class="list-group-item d-flex align-items-center p-3 history-item">
{% if e.photo %}<img src="{{ url_for('static', filename='uploads/' + e.photo) }}" class="car-photo-small me-3">
{% else %}<div class="car-photo-small me-3 d-flex align-items-center justify-content-center" style="background:rgba(255,255,255,0.1);border-radius:14px"><i class="fas fa-car" style="color:rgba(255,255,255,0.7)"></i></div>{% endif %}
<div class="flex-grow-1"><div class="fw-semibold mb-1">{{ e.date }} – {{ e.car or tr('no_car') }}</div>
<div class="small" style="color:rgba(255,255,255,0.7)"><span class="me-3"><i class="fas fa-road me-1"></i>{{ e.distance }} {{ tr('km') }}</span><span class="me-3"><i class="fas fa-gas-pump me-1"></i>{{ e.fuel }} {{ tr('fuel_unit') }}</span><span class="me-3"><i class="fas fa-tachometer-alt me-1"></i>{{ e.consumption }} {{ tr('fuel_unit') }}/100{{ tr('km') }}</span><span><i class="fas fa-coins me-1"></i>{{ "%.2f"|format(e.cost) }} {{ currency_symbol }}</span></div></div>
<div class="ms-auto d-flex flex-column align-items-end"><button class="btn btn-sm btn-outline-secondary mb-1" onclick="copyEntry(this)" data-text="{{ tr('date') }}: {{ e.date }}, {{ tr('car') }}: {{ e.car or tr('no_car') }}, {{ tr('fuel_consumption') }}: {{ e.consumption }} {{ tr('fuel_unit') }}/100{{ tr('km') }}, {{ tr('cost') }}: {{ '%.2f'|format(e.cost) }} {{ currency_symbol }}"><i class="fas fa-copy"></i></button><button class="btn btn-sm btn-outline-danger" onclick="deleteHistoryEntry({{ e.id }})"><i class="fas fa-trash"></i></button></div></div>{% endfor %}</div>
<div class="text-center mt-3"><button id="toggleHistoryBtn" class="btn btn-outline-secondary" onclick="toggleHistory()" style="display:none">{{ tr('show_more') }}</button></div>
{% else %}<div class="card p-4 text-center">{{ tr('history_empty') }}</div>{% endif %}
{% else %}<div class="card p-4 text-center">{{ tr('login_for_history') }}</div>{% endif %}''',

    # ── Авторизация ──
    "auth.html": r'''{% extends "base.html" %}{% block content %}<div class="row justify-content-center"><div class="col-md-5"><div class="card p-4">
<h2 class="mb-4 text-center fw-bold">{{ tr('login_register') }}</h2>
<ul class="nav nav-tabs mb-3" style="border-bottom:1px solid rgba(255,255,255,0.2)"><li><button class="nav-link active" data-bs-toggle="tab" data-bs-target="#login-pane" style="font-weight:600;color:white;background:rgba(255,255,255,0.1)">{{ tr('login') }}</button></li><li><button class="nav-link" data-bs-toggle="tab" data-bs-target="#register-pane" style="font-weight:600;color:white">{{ tr('register') }}</button></li></ul>
<div class="tab-content"><div class="tab-pane fade show active" id="login-pane">
<form method="post" action="/auth"><input type="hidden" name="form_type" value="login">
<div class="mb-3"><label class="form-label fw-semibold">{{ tr('email') }}</label><input type="email" name="email" class="form-control" required></div>
<div class="mb-3"><label class="form-label fw-semibold">{{ tr('password') }}</label><div class="input-group"><input type="password" name="password" class="form-control" required><button class="btn btn-outline-secondary" type="button" onclick="togglePassword(this)" style="border-radius:0 12px 12px 0"><i class="fas fa-eye"></i></button></div></div>
<button type="submit" class="btn btn-primary w-100">{{ tr('login_account') }}</button></form></div>
<div class="tab-pane fade" id="register-pane"><form method="post" action="/auth"><input type="hidden" name="form_type" value="register">
<div class="mb-3"><label class="form-label fw-semibold">{{ tr('email') }}</label><input type="email" name="email" class="form-control" required></div>
<div class="mb-3"><label class="form-label fw-semibold">{{ tr('password') }}</label><div class="input-group"><input type="password" name="password" class="form-control" required><button class="btn btn-outline-secondary" type="button" onclick="togglePassword(this)" style="border-radius:0 12px 12px 0"><i class="fas fa-eye"></i></button></div></div>
<div class="mb-3"><label class="form-label fw-semibold">{{ tr('repeat_password') }}</label><div class="input-group"><input type="password" name="password2" class="form-control" required><button class="btn btn-outline-secondary" type="button" onclick="togglePassword(this)" style="border-radius:0 12px 12px 0"><i class="fas fa-eye"></i></button></div></div>
<button type="submit" class="btn btn-primary w-100">{{ tr('create_account') }}</button></form></div></div></div></div></div>{% endblock %}''',

    # ── Профиль ──
    "profile.html": r'''{% extends "base.html" %}{% block content %}
<div class="d-flex justify-content-between align-items-center mb-4 flex-wrap gap-2"><h2 class="fw-bold">{{ tr('profile') }}</h2>
<div class="d-flex gap-2"><button class="btn btn-outline-secondary" onclick="openChangePasswordModal()">{{ tr('change_password') }}</button><a href="/logout" class="btn btn-outline-secondary">{{ tr('logout') }}</a><form method="post" action="/delete_account" onsubmit="return confirm('{{ tr('delete_account_msg') }}')" class="m-0"><button class="btn btn-outline-danger">{{ tr('delete_account') }}</button></form></div></div>
<div class="card p-4 mb-4"><p class="mb-0"><i class="fas fa-envelope me-2"></i><strong>{{ user }}</strong></p></div>
<div class="card p-4" id="carsSection">{% include 'profile_cars.html' %}</div>
<div class="modal fade" id="editCarModal"><div class="modal-dialog modal-dialog-centered"><div class="modal-content"><div class="modal-header border-0"><h5 class="modal-title fw-bold">{{ tr('edit_car_title') }}</h5><button class="btn-close" data-bs-dismiss="modal"></button></div><div class="modal-body"><form id="editCarForm" onsubmit="submitEditCar(event)" enctype="multipart/form-data"><input type="hidden" name="old_name" id="editOldName"><div class="mb-3"><label class="form-label fw-semibold">{{ tr('car_name') }}</label><input type="text" name="new_name" id="editNewName" class="form-control" required></div><div class="mb-3"><label class="form-label fw-semibold">Фото</label><input type="file" name="photo" class="form-control" accept="image/*"><small class="text-muted">Оставьте пустым, чтобы не менять фото</small></div><button type="submit" class="btn btn-primary w-100">{{ tr('save') }}</button></form></div></div></div></div>
<div class="modal fade" id="changePasswordModal"><div class="modal-dialog modal-dialog-centered"><div class="modal-content"><div class="modal-header border-0"><h5 class="modal-title fw-bold">{{ tr('change_password') }}</h5><button class="btn-close" data-bs-dismiss="modal"></button></div><div class="modal-body"><form id="changePasswordForm" onsubmit="submitChangePassword(event)"><div class="mb-3"><label class="form-label fw-semibold">{{ tr('old_password') }}</label><div class="input-group"><input type="password" name="old_password" class="form-control" required><button class="btn btn-outline-secondary" type="button" onclick="togglePassword(this)" style="border-radius:0 12px 12px 0"><i class="fas fa-eye"></i></button></div></div><div class="mb-3"><label class="form-label fw-semibold">{{ tr('new_password') }}</label><div class="input-group"><input type="password" name="new_password" class="form-control" required><button class="btn btn-outline-secondary" type="button" onclick="togglePassword(this)" style="border-radius:0 12px 12px 0"><i class="fas fa-eye"></i></button></div></div><div class="mb-3"><label class="form-label fw-semibold">{{ tr('repeat_new_password') }}</label><div class="input-group"><input type="password" name="new_password2" class="form-control" required><button class="btn btn-outline-secondary" type="button" onclick="togglePassword(this)" style="border-radius:0 12px 12px 0"><i class="fas fa-eye"></i></button></div></div><button type="submit" class="btn btn-primary w-100 mt-2">{{ tr('change_password_btn') }}</button></form></div></div></div></div>{% endblock %}
{% block scripts %}<script>
let editCarModal,passwordModal;document.addEventListener('DOMContentLoaded',()=>{editCarModal=new bootstrap.Modal(document.getElementById('editCarModal'));passwordModal=new bootstrap.Modal(document.getElementById('changePasswordModal'))});
function openChangePasswordModal(){passwordModal.show()}
async function submitChangePassword(e){e.preventDefault();let fd=new FormData(e.target);try{let r=await fetch('/change_password',{method:'POST',body:fd});let d=await r.json();alert(d.message);if(d.success){e.target.reset();passwordModal.hide()}}catch(e){alert('{{ tr("error") }}')}}
async function addCar(e){e.preventDefault();try{let r=await fetch('/add_car',{method:'POST',body:new FormData(e.target)});document.getElementById('carsSection').innerHTML=await r.text()}catch(e){alert('{{ tr("error") }}')}}
function editCar(name){document.getElementById('editOldName').value=name;document.getElementById('editNewName').value=name;editCarModal.show()}
async function submitEditCar(e){e.preventDefault();try{let r=await fetch('/edit_car',{method:'POST',body:new FormData(e.target)});document.getElementById('carsSection').innerHTML=await r.text();editCarModal.hide()}catch(e){alert('{{ tr("error") }}')}}
function copyCarData(name,distance,cost,avg){let text=`{{ tr('car') }}: ${name}\n{{ tr('total_distance') }}: ${distance} {{ tr('km') }}\n{{ tr('total_cost') }}: ${cost} {{ currency_symbol }}\n{{ tr('avg_consumption') }}: ${avg} {{ tr('fuel_unit') }}/100{{ tr('km') }}`;navigator.clipboard.writeText(text).then(()=>alert('{{ tr("copied") }}')).catch(()=>alert('{{ tr("error") }}'))}
function deleteCar(name){showConfirm('{{ tr("delete_car_confirm") }}','{{ tr("delete_car_confirm") }}','{{ tr("delete") }}',async()=>{try{let r=await fetch('/delete_car/'+name,{method:'POST'});document.getElementById('carsSection').innerHTML=await r.text()}catch(e){alert('{{ tr("error") }}')}})}
</script>{% endblock %}''',

    # ── Карточки авто (AJAX-фрагмент) ──
    "profile_cars.html": r'''<h5 class="fw-bold mb-3">{{ tr('add_car_title') }}</h5>
<form onsubmit="addCar(event)" enctype="multipart/form-data" class="row g-3 mb-4"><div class="col-md-5"><input type="text" name="name" class="form-control" placeholder="{{ tr('car_name') }}" required></div><div class="col-md-5"><input type="file" name="photo" class="form-control" accept="image/*"></div><div class="col-md-2"><button type="submit" class="btn btn-primary w-100">{{ tr('save') }}</button></div></form>
<h5 class="fw-bold">{{ tr('my_cars') }}</h5>
<div class="row">{% for car in cars %}<div class="col-md-4 mb-4 d-flex"><div class="card p-3 w-100" style="border-radius:20px">
{% if car.photo %}<img src="{{ url_for('static', filename='uploads/' + car.photo) }}" class="car-photo-medium w-100 mb-3" style="object-fit:cover;height:160px;border-radius:16px">
{% else %}<div class="d-flex align-items-center justify-content-center w-100 mb-3" style="height:160px;background:rgba(255,255,255,0.1);border-radius:16px"><i class="fas fa-car fa-4x" style="color:rgba(255,255,255,0.5)"></i></div>{% endif %}
<div class="text-center d-flex flex-column flex-grow-1"><h6 class="fw-bold">{{ car.name }}</h6>
{% if car.avg_consumption %}<span class="badge mb-2">{{ "%.1f"|format(car.avg_consumption) }} {{ tr('fuel_unit') }}/100{{ tr('km') }}</span>{% endif %}
<div class="mt-auto"><div class="mt-2 small" style="color:rgba(255,255,255,0.7)">{% set s = car_stats.get(car.name, {}) %}
<div>{{ tr('total_distance') }}: {{ s.total_distance }} {{ tr('km') }}</div><div>{{ tr('total_cost') }}: {{ s.total_cost }} {{ currency_symbol }}</div><div>{{ tr('avg_consumption') }}: {{ s.avg_consumption }} {{ tr('fuel_unit') }}/100{{ tr('km') }}</div></div>
<div class="mt-3 car-actions"><button class="btn btn-sm btn-outline-secondary" onclick="editCar('{{ car.name }}')">{{ tr('edit') }}</button><button class="btn btn-sm btn-outline-secondary" onclick="copyCarData('{{ car.name }}','{{ s.total_distance }}','{{ s.total_cost }}','{{ s.avg_consumption }}')">{{ tr('copy') }}</button><button class="btn btn-sm btn-outline-danger" onclick="deleteCar('{{ car.name }}')">{{ tr('delete') }}</button></div></div></div></div></div>{% else %}<div class="col-12"><p class="text-muted">{{ tr('no_cars') }}</p></div>{% endfor %}</div>''',

    # ── Настройки ──
    "settings.html": r'''{% extends "base.html" %}{% block content %}<h2 class="mb-4 fw-bold">{{ tr('settings_title') }}</h2><div class="card p-4"><form id="settingsForm" onsubmit="submitSettings(event)"><div class="mb-4"><label class="form-label fw-bold">{{ tr('language_setting') }}</label><select name="language" class="form-select" style="max-width:300px"><option value="ru" {% if settings.language=='ru' %}selected{% endif %}>Русский</option><option value="en" {% if settings.language=='en' %}selected{% endif %}>English</option></select></div><div class="mb-4"><label class="form-label fw-bold">{{ tr('currency_setting') }}</label><select name="currency" class="form-select" style="max-width:200px">{% for k,s in currencies.items() %}<option value="{{ k }}" {% if settings.currency==k %}selected{% endif %}>{{ s }}</option>{% endfor %}</select></div><button type="submit" class="btn btn-primary">{{ tr('save') }}</button></form></div>{% endblock %}{% block scripts %}<script>async function submitSettings(e){e.preventDefault();try{let r=await fetch('/settings',{method:'POST',body:new FormData(e.target)});if(r.ok)location.reload()}catch(e){}}</script>{% endblock %}''',

    # ── О приложении (красивое оформление) ──
    "about.html": r'''{% extends "base.html" %}{% block content %}
<h2 class="mb-4 fw-bold text-center">{{ tr('about_title') }}</h2>

<!-- Вступление -->
<div class="card p-4 mb-4 text-center">
    <p class="mb-0" style="font-size:1.1rem;line-height:1.7">{{ tr('about_intro') }}</p>
</div>

<!-- Возможности — сетка карточек -->
<div class="card p-4 mb-4">
    <h4 class="fw-bold mb-3">{{ tr('about_features_title') }}</h4>
    <div class="feature-grid">
        <div class="feature-card"><i class="fas fa-exchange-alt"></i><h6>{{ tr('calc_mode_consumption') }} / {{ tr('calc_mode_cost') }}</h6><p>{{ tr('about_feature_1') }}</p></div>
        <div class="feature-card"><i class="fas fa-car"></i><h6>{{ tr('my_cars') }}</h6><p>{{ tr('about_feature_2') }}</p></div>
        <div class="feature-card"><i class="fas fa-history"></i><h6>{{ tr('history') }}</h6><p>{{ tr('about_feature_3') }}</p></div>
        <div class="feature-card"><i class="fas fa-chart-bar"></i><h6>{{ tr('total_distance') }} / {{ tr('total_cost') }}</h6><p>{{ tr('about_feature_4') }}</p></div>
        <div class="feature-card"><i class="fas fa-coins"></i><h6>{{ tr('currency_setting') }}</h6><p>{{ tr('about_feature_5') }}</p></div>
        <div class="feature-card"><i class="fas fa-globe"></i><h6>{{ tr('language_setting') }}</h6><p>{{ tr('about_feature_6') }}</p></div>
        <div class="feature-card"><i class="fas fa-copy"></i><h6>{{ tr('copy') }}</h6><p>{{ tr('about_feature_7') }}</p></div>
    </div>
</div>

<!-- Безопасность + Технические детали — две колонки -->
<div class="row">
    <div class="col-md-6 mb-4">
        <div class="card p-4 h-100">
            <h4 class="fw-bold mb-3"><i class="fas fa-shield-alt me-2"></i>{{ tr('about_security_title') }}</h4>
            <ul style="list-style:none;padding:0">
                <li class="mb-2" style="color:rgba(255,255,255,0.85)"><i class="fas fa-check-circle me-2" style="color:#34c759"></i>{{ tr('about_security_1') }}</li>
                <li class="mb-2" style="color:rgba(255,255,255,0.85)"><i class="fas fa-check-circle me-2" style="color:#34c759"></i>{{ tr('about_security_2') }}</li>
                <li style="color:rgba(255,255,255,0.85)"><i class="fas fa-check-circle me-2" style="color:#34c759"></i>{{ tr('about_security_3') }}</li>
            </ul>
        </div>
    </div>
    <div class="col-md-6 mb-4">
        <div class="card p-4 h-100">
            <h4 class="fw-bold mb-3"><i class="fas fa-cogs me-2"></i>{{ tr('about_tech_title') }}</h4>
            <ul style="list-style:none;padding:0">
                <li class="mb-2" style="color:rgba(255,255,255,0.85)"><i class="fas fa-server me-2" style="color:#0071e3"></i>{{ tr('about_tech_1') }}</li>
                <li class="mb-2" style="color:rgba(255,255,255,0.85)"><i class="fas fa-database me-2" style="color:#0071e3"></i>{{ tr('about_tech_2') }}</li>
                <li class="mb-2" style="color:rgba(255,255,255,0.85)"><i class="fas fa-paint-brush me-2" style="color:#0071e3"></i>{{ tr('about_tech_3') }}</li>
                <li style="color:rgba(255,255,255,0.85)"><i class="fas fa-mobile-alt me-2" style="color:#0071e3"></i>{{ tr('about_tech_4') }}</li>
            </ul>
        </div>
    </div>
</div>

<hr style="border-color:rgba(255,255,255,0.1)">
<p class="text-muted text-center">{{ tr('version') }}</p>
{% endblock %}'''
}


# ═════════════════════════════════════════════════════════════════════
# ИНИЦИАЛИЗАЦИЯ FLASK
# ═════════════════════════════════════════════════════════════════════

app = Flask(__name__)
app.jinja_loader = DictLoader(TEMPLATES)
app.secret_key = os.urandom(24)

UPLOAD_FOLDER = os.path.join('static', 'uploads')
os.makedirs(UPLOAD_FOLDER, exist_ok=True)
app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER
app.config['MAX_CONTENT_LENGTH'] = 50 * 1024 * 1024

app.add_template_global(tr, 'tr')
app.add_template_global(CURRENCIES, 'currencies')


# ═════════════════════════════════════════════════════════════════════
# МАРШРУТЫ
# ═════════════════════════════════════════════════════════════════════

@app.route('/')
def index():
    cars, history = [], []
    if 'user' in session:
        try:
            cars = db_get_cars(session['user'])
            history = db_get_history(session['user'])
        except:
            flash(tr('database_error'), 'error')
    return render_template("index.html", cars=cars, history=history,
                           lang=session.get('language', 'ru'),
                           currency_symbol=CURRENCIES.get(session.get('currency', '₽ RUB'), '₽'),
                           user=session.get('user'))

@app.route('/calculate', methods=['POST'])
def calculate():
    try:
        dist = float(request.form.get('distance', 0))
        price = float(request.form.get('price', 0))
    except:
        return jsonify({'error': tr('enter_numbers')}), 400
    if dist <= 0: return jsonify({'error': tr('distance_positive')}), 400
    if price <= 0: return jsonify({'error': 'Цена должна быть больше 0'}), 400

    mode = request.form.get('mode', 'consumption')
    car_name = request.form.get('car', tr('no_car')) if 'user' in session else tr('no_car')
    currency = CURRENCIES.get(session.get('currency', '₽ RUB'), '₽') if 'user' in session else '₽'

    if mode == 'consumption':
        try: fuel = float(request.form.get('fuel', 0))
        except: return jsonify({'error': tr('enter_numbers')}), 400
        if fuel <= 0: return jsonify({'error': 'Расход топлива должен быть больше 0'}), 400
        consumption, cost = (fuel / dist) * 100, fuel * price
    else:
        try: avg_input = float(request.form.get('avg_consumption', 0))
        except: return jsonify({'error': tr('enter_numbers')}), 400
        if avg_input <= 0: return jsonify({'error': tr('distance_positive2')}), 400
        consumption, fuel, cost = avg_input, (avg_input / 100) * dist, ((avg_input / 100) * dist) * price

    if 'user' in session:
        try:
            db_add_history(session['user'], {
                'date': datetime.now().strftime("%d.%m.%Y %H:%M"), 'car': car_name,
                'distance': dist, 'fuel': round(fuel, 2), 'price': price, 'currency': currency,
                'consumption': round(consumption, 2), 'cost': round(cost, 2)
            })
            if car_name not in (tr('no_car'), '-- No car --'):
                db_update_avg(session['user'], car_name)
        except: pass

    return jsonify({'consumption': round(consumption, 2), 'fuel': round(fuel, 2), 'cost': round(cost, 2), 'mode': mode, 'currency': currency})

@app.route('/history_html')
def history_html():
    if 'user' not in session: return "<div class='card p-4 text-center'>" + tr('login_for_history') + "</div>"
    return render_template("history_list.html", history=db_get_history(session['user'], request.args.get('car', '')),
                           lang=session.get('language', 'ru'), user=session['user'],
                           currency_symbol=CURRENCIES.get(session.get('currency', '₽ RUB'), '₽'))

@app.route('/auth', methods=['GET', 'POST'])
def auth():
    if 'user' in session: return redirect(url_for('profile'))
    if request.method == 'POST':
        email, pw = request.form['email'].strip().lower(), request.form['password']
        if request.form.get('form_type') == 'register':
            pw2 = request.form['password2']
            if not email or not pw: flash(tr('fill_all'), 'error')
            elif not is_valid_email(email): flash(tr('invalid_email'), 'error')
            elif len(pw) < 6: flash(tr('password_short'), 'error')
            elif pw != pw2: flash(tr('passwords_mismatch'), 'error')
            elif db_user_exists(email): flash(tr('user_exists'), 'error')
            else:
                try:
                    db_create_user(email, hash_pw(pw))
                    session['user'], session['language'], session['currency'] = email, 'ru', '₽ RUB'
                    return redirect(url_for('profile'))
                except: flash(tr('database_error'), 'error')
        else:
            if not is_valid_email(email): flash(tr('invalid_email'), 'error')
            else:
                stored = db_get_user(email)
                if stored and stored == hash_pw(pw):
                    session['user'] = email
                    s = db_get_settings(email); session['language'], session['currency'] = s['language'], s['currency']
                    return redirect(url_for('profile'))
                flash(tr('wrong_credentials'), 'error')
    return render_template("auth.html", lang=session.get('language', 'ru'))

@app.route('/logout')
def logout():
    session.clear()
    return redirect(url_for('index'))

@app.route('/profile')
def profile():
    if 'user' not in session: return redirect(url_for('auth'))
    try:
        cars = db_get_cars(session['user'])
        car_stats = {c['name']: db_get_car_stats(session['user'], c['name']) for c in cars}
    except:
        cars, car_stats = [], []
    return render_template("profile.html", cars=cars, car_stats=car_stats,
                           lang=session.get('language', 'ru'), user=session['user'],
                           currency_symbol=CURRENCIES.get(session.get('currency', '₽ RUB'), '₽'))

@app.route('/profile_cars_html')
def profile_cars_html():
    if 'user' not in session: return ''
    return render_profile_cars()

@app.route('/add_car', methods=['POST'])
def add_car():
    if 'user' not in session: return redirect(url_for('auth'))
    name = request.form['name'].strip()
    if not name: return redirect(url_for('profile'))
    photo = request.files.get('photo')
    photo_name = save_car_photo(photo) if photo and photo.filename else None
    try: db_add_car(session['user'], name, photo_name)
    except: flash(tr('database_error'), 'error')
    return render_profile_cars()

@app.route('/edit_car', methods=['POST'])
def edit_car():
    if 'user' not in session: return redirect(url_for('auth'))
    old, new = request.form['old_name'].strip(), request.form['new_name'].strip()
    if not new: return jsonify({'success': False, 'message': tr('enter_name')})
    cars = db_get_cars(session['user'])
    old_photo = next((c['photo'] for c in cars if c['name'] == old), None)
    photo = request.files.get('photo')
    photo_name = save_car_photo(photo, old_photo) if photo and photo.filename else None
    try: db_update_car(session['user'], old, new, photo_name)
    except: flash(tr('database_error'), 'error')
    return render_profile_cars()

@app.route('/delete_car/<name>', methods=['POST'])
def delete_car(name):
    if 'user' not in session: return redirect(url_for('auth'))
    try: db_delete_car(session['user'], name)
    except: flash(tr('database_error'), 'error')
    return render_profile_cars()

@app.route('/change_password', methods=['POST'])
def change_password():
    if 'user' not in session: return jsonify({'success': False}), 401
    old, new, new2 = request.form['old_password'], request.form['new_password'], request.form['new_password2']
    if not old or not new: return jsonify({'success': False, 'message': tr('fill_all')})
    if len(new) < 6: return jsonify({'success': False, 'message': tr('password_short')})
    if new != new2: return jsonify({'success': False, 'message': tr('passwords_mismatch')})
    if db_get_user(session['user']) != hash_pw(old): return jsonify({'success': False, 'message': tr('old_password_wrong')})
    try: db_update_password(session['user'], hash_pw(new)); return jsonify({'success': True, 'message': tr('password_changed')})
    except: return jsonify({'success': False, 'message': tr('database_error')})

@app.route('/delete_account', methods=['POST'])
def delete_account():
    if 'user' not in session: return redirect(url_for('auth'))
    try: db_delete_user(session['user']); session.clear()
    except: flash(tr('database_error'), 'error')
    return redirect(url_for('index'))

@app.route('/clear_history', methods=['POST'])
def clear_history():
    if 'user' not in session: return '', 401
    db_clear_history(session['user']); return '', 200

@app.route('/delete_history/<int:eid>', methods=['POST'])
def delete_history(eid):
    if 'user' not in session: return '', 401
    db_delete_history_entry(session['user'], eid); return '', 200

@app.route('/settings', methods=['GET', 'POST'])
def settings():
    if 'user' not in session: return redirect(url_for('auth'))
    if request.method == 'POST':
        lang, cur = request.form.get('language', 'ru'), request.form.get('currency', '₽ RUB')
        session['language'], session['currency'] = lang, cur
        db_update_settings(session['user'], 'light', lang, cur); return redirect(url_for('settings'))
    return render_template("settings.html", settings=db_get_settings(session['user']),
                           currencies=CURRENCIES, lang=session.get('language', 'ru'), user=session.get('user'))

@app.route('/about')
def about():
    return render_template("about.html", lang=session.get('language', 'ru'), user=session.get('user'))


# ═════════════════════════════════════════════════════════════════════
# ТОЧКА ВХОДА
# ═════════════════════════════════════════════════════════════════════

if __name__ == '__main__':
    if init_db_pool():
        create_tables()
        print("\n" + "=" * 50)
        print("  FullCalcPro запущен!")
        print("  Открой в браузере: http://127.0.0.1:5000")
        print("=" * 50 + "\n")
        app.run(debug=False, host='127.0.0.1', port=5000)
    else:
        print("\n!!! НЕ УДАЛОСЬ ПОДКЛЮЧИТЬСЯ К POSTGRESQL !!!")
        print("Проверь DB_CONFIG в начале файла\n")
