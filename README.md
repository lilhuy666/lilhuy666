import tkinter as tk
from tkinter import ttk, messagebox, filedialog
import json
import os
import re
import hashlib
import shutil
from datetime import datetime, timedelta
import math
import webbrowser

try:
    from PIL import Image, ImageTk, ImageDraw, ImageFont
except ImportError:
    import subprocess, sys
    subprocess.check_call([sys.executable, "-m", "pip", "install", "Pillow"])
    from PIL import Image, ImageTk, ImageDraw, ImageFont

# ─── Data storage ──────────────────────────────────────────────────────────────
DATA_FILE = os.path.join(os.path.expanduser("~"), ".fuel_calc_data.json")
PHOTO_DIR = os.path.join(os.path.expanduser("~"), ".fuel_calc_photos")
MAP_DIR = os.path.join(os.path.expanduser("~"), ".fuel_calc_maps")
os.makedirs(PHOTO_DIR, exist_ok=True)
os.makedirs(MAP_DIR, exist_ok=True)

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

# ─── Real fuel prices data ────────────────────────────────────────────────────
FUEL_PRICES = {
    "АИ-92": {
        "Москва": 48.50, "Санкт-Петербург": 48.20, "Казань": 46.80,
        "Екатеринбург": 46.50, "Новосибирск": 45.90, "Сочи": 49.00,
        "Краснодар": 47.50, "Владивосток": 51.00, "Нижний Новгород": 47.30,
        "Самара": 46.70, "Омск": 45.50, "Челябинск": 46.20,
        "Ростов-на-Дону": 47.80, "Уфа": 46.40
    },
    "АИ-95": {
        "Москва": 52.50, "Санкт-Петербург": 52.00, "Казань": 50.50,
        "Екатеринбург": 50.20, "Новосибирск": 49.80, "Сочи": 53.50,
        "Краснодар": 51.50, "Владивосток": 55.00, "Нижний Новгород": 51.00,
        "Самара": 50.00, "Омск": 48.50, "Челябинск": 49.80,
        "Ростов-на-Дону": 51.50, "Уфа": 50.00
    },
    "ДТ": {
        "Москва": 56.50, "Санкт-Петербург": 56.00, "Казань": 54.50,
        "Екатеринбург": 54.00, "Новосибирск": 53.50, "Сочи": 57.00,
        "Краснодар": 55.00, "Владивосток": 58.50, "Нижний Новгород": 55.00,
        "Самара": 54.00, "Омск": 52.50, "Челябинск": 53.50,
        "Ростов-на-Дону": 55.50, "Уфа": 54.00
    }
}

# ─── Gas stations data with real coordinates ──────────────────────────────────
GAS_STATIONS_RUSSIA = [
    # Москва
    {"name": "Лукойл №1", "city": "Москва", "address": "Ленинградское шоссе, 1", 
     "lat": 55.7558, "lon": 37.6173, "brand": "lukoil", 
     "fuel_types": {"АИ-92": 48.50, "АИ-95": 52.50, "ДТ": 56.50}},
    {"name": "Газпромнефть №1", "city": "Москва", "address": "Проспект Мира, 100", 
     "lat": 55.7512, "lon": 37.6184, "brand": "gazprom", 
     "fuel_types": {"АИ-92": 48.30, "АИ-95": 52.30, "ДТ": 56.20}},
    {"name": "Роснефть №1", "city": "Москва", "address": "Варшавское шоссе, 25", 
     "lat": 55.7600, "lon": 37.6200, "brand": "rosneft", 
     "fuel_types": {"АИ-92": 48.00, "АИ-95": 52.00, "ДТ": 56.00}},
    {"name": "Лукойл №2", "city": "Москва", "address": "Кутузовский пр., 30", 
     "lat": 55.7400, "lon": 37.5500, "brand": "lukoil", 
     "fuel_types": {"АИ-92": 48.60, "АИ-95": 52.60, "ДТ": 56.80}},
    {"name": "Shell", "city": "Москва", "address": "МКАД, 15 км", 
     "lat": 55.7800, "lon": 37.6300, "brand": "shell", 
     "fuel_types": {"АИ-92": 49.00, "АИ-95": 53.00, "ДТ": 57.00}},
    
    # Санкт-Петербург
    {"name": "Лукойл №3", "city": "Санкт-Петербург", "address": "Невский пр., 50", 
     "lat": 59.9343, "lon": 30.3351, "brand": "lukoil", 
     "fuel_types": {"АИ-92": 48.20, "АИ-95": 52.00, "ДТ": 56.00}},
    {"name": "Газпромнефть №2", "city": "Санкт-Петербург", "address": "Московский пр., 150", 
     "lat": 59.9290, "lon": 30.3200, "brand": "gazprom", 
     "fuel_types": {"АИ-92": 48.00, "АИ-95": 51.80, "ДТ": 55.80}},
    {"name": "Роснефть №2", "city": "Санкт-Петербург", "address": "Лиговский пр., 100", 
     "lat": 59.9200, "lon": 30.3500, "brand": "rosneft", 
     "fuel_types": {"АИ-92": 47.80, "АИ-95": 51.50, "ДТ": 55.50}},
    
    # Казань
    {"name": "Роснефть №3", "city": "Казань", "address": "Пр. Победы, 100", 
     "lat": 55.7961, "lon": 49.1064, "brand": "rosneft", 
     "fuel_types": {"АИ-92": 46.80, "АИ-95": 50.50, "ДТ": 54.50}},
    {"name": "Газпромнефть №3", "city": "Казань", "address": "Ул. Декабристов, 50", 
     "lat": 55.8000, "lon": 49.1200, "brand": "gazprom", 
     "fuel_types": {"АИ-92": 47.00, "АИ-95": 50.80, "ДТ": 54.80}},
    
    # Екатеринбург
    {"name": "Лукойл №4", "city": "Екатеринбург", "address": "Ул. Ленина, 50", 
     "lat": 56.8389, "lon": 60.6057, "brand": "lukoil", 
     "fuel_types": {"АИ-92": 46.50, "АИ-95": 50.20, "ДТ": 54.00}},
    
    # Новосибирск
    {"name": "Газпромнефть №4", "city": "Новосибирск", "address": "Красный пр., 100", 
     "lat": 55.0302, "lon": 82.9204, "brand": "gazprom", 
     "fuel_types": {"АИ-92": 45.90, "АИ-95": 49.80, "ДТ": 53.50}},
    
    # Сочи
    {"name": "Роснефть №4", "city": "Сочи", "address": "Курортный пр., 50", 
     "lat": 43.5855, "lon": 39.7231, "brand": "rosneft", 
     "fuel_types": {"АИ-92": 49.00, "АИ-95": 53.50, "ДТ": 57.00}},
    
    # Краснодар
    {"name": "Лукойл №5", "city": "Краснодар", "address": "Ул. Красная, 100", 
     "lat": 45.0355, "lon": 38.9753, "brand": "lukoil", 
     "fuel_types": {"АИ-92": 47.50, "АИ-95": 51.50, "ДТ": 55.00}},
    
    # Владивосток
    {"name": "Газпромнефть №5", "city": "Владивосток", "address": "Океанский пр., 50", 
     "lat": 43.1155, "lon": 131.8855, "brand": "gazprom", 
     "fuel_types": {"АИ-92": 51.00, "АИ-95": 55.00, "ДТ": 58.50}},
    
    # Нижний Новгород
    {"name": "Роснефть №5", "city": "Нижний Новгород", "address": "Ул. Большая Покровская, 50", 
     "lat": 56.3268, "lon": 44.0065, "brand": "rosneft", 
     "fuel_types": {"АИ-92": 47.30, "АИ-95": 51.00, "ДТ": 55.00}},
    
    # Самара
    {"name": "Лукойл №6", "city": "Самара", "address": "Московское шоссе, 50", 
     "lat": 53.1959, "lon": 50.1002, "brand": "lukoil", 
     "fuel_types": {"АИ-92": 46.70, "АИ-95": 50.00, "ДТ": 54.00}},
    
    # Омск
    {"name": "Газпромнефть №6", "city": "Омск", "address": "Пр. Маркса, 50", 
     "lat": 54.9893, "lon": 73.3682, "brand": "gazprom", 
     "fuel_types": {"АИ-92": 45.50, "АИ-95": 48.50, "ДТ": 52.50}},
    
    # Челябинск
    {"name": "Роснефть №6", "city": "Челябинск", "address": "Пр. Ленина, 50", 
     "lat": 55.1644, "lon": 61.4368, "brand": "rosneft", 
     "fuel_types": {"АИ-92": 46.20, "АИ-95": 49.80, "ДТ": 53.50}},
    
    # Ростов-на-Дону
    {"name": "Лукойл №7", "city": "Ростов-на-Дону", "address": "Ул. Большая Садовая, 50", 
     "lat": 47.2357, "lon": 39.7015, "brand": "lukoil", 
     "fuel_types": {"АИ-92": 47.80, "АИ-95": 51.50, "ДТ": 55.50}},
    
    # Уфа
    {"name": "Газпромнефть №7", "city": "Уфа", "address": "Пр. Октября, 50", 
     "lat": 54.7348, "lon": 55.9578, "brand": "gazprom", 
     "fuel_types": {"АИ-92": 46.40, "АИ-95": 50.00, "ДТ": 54.00}},
]

# ─── Translations ──────────────────────────────────────────────────────────────
TRANSLATIONS = {
    "ru": {
        "app_title": "Калькулятор расхода топлива",
        "app_subtitle": "Расчёт расхода топлива",
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
        "gs_title": "Карта заправок России",
        "gs_city": "Поиск по городу:",
        "gs_search": "🔍 Найти",
        "gs_open_map": "🗺 Открыть интерактивную карту",
        "gs_to_calc": "В калькулятор",
        "gs_enter_distance": "Ваше расстояние до заправки (км):",
        "gs_avg_from_car": "Средний расход авто",
        "gs_no_avg": "Нет данных о расходе",
        "gs_selected_station": "Выбранная заправка:",
        "gs_click_hint": "Нажмите на маркер на карте для просмотра цен",
        "gs_distance_note": "💡 Введите примерное расстояние до заправки",
        "gs_all_cities": "Все города",
        "gs_fuel_price_hint": "Цена топлива:",
        "gs_fuel_type": "Тип топлива:",
        "brands": {
            "lukoil": "Лукойл", "gazprom": "Газпромнефть", 
            "rosneft": "Роснефть", "shell": "Shell"
        },
        "about_title": "О приложении",
        "version": "Версия 5.0  •  2024",
        "about_section1_title": "О приложении",
        "about_section1": "Профессиональное приложение для расчёта расхода топлива с интерактивной картой заправок.",
        "about_section2_title": "Возможности",
        "about_section2": "• Точный расчёт расхода\n• Расчёт стоимости поездки\n• Хранение до 25 автомобилей\n• История расчётов\n• Интерактивная карта заправок\n• Реальные цены на топливо",
        "about_section3_title": "Как использовать",
        "about_section3": "1. Зарегистрируйтесь\n2. Добавьте автомобиль\n3. Выберите режим расчёта\n4. Введите данные\n5. Нажмите «Рассчитать»",
        "about_section4_title": "Безопасность",
        "about_section4": "Пароли зашифрованы (SHA-256). Данные хранятся локально.",
        "about_section5_title": "Интерпретация",
        "about_section5": "До 8 л/100 км — экономично\n8-12 л/100 км — средне\nБолее 12 л/100 км — высокий расход",
        "footer": "© 2024 Калькулятор расхода топлива",
        "no_cars_limit": "Максимум 25 автомобилей",
    },
    "en": {
        "app_title": "Fuel Calculator",
        "app_subtitle": "Fuel consumption calculation",
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
        "gs_title": "Russian Gas Stations Map",
        "gs_city": "Search by city:",
        "gs_search": "🔍 Search",
        "gs_open_map": "🗺 Open interactive map",
        "gs_to_calc": "To Calculator",
        "gs_enter_distance": "Your distance to station (km):",
        "gs_avg_from_car": "Vehicle avg. consumption",
        "gs_no_avg": "No consumption data",
        "gs_selected_station": "Selected station:",
        "gs_click_hint": "Click on map marker to see prices",
        "gs_distance_note": "💡 Enter approximate distance to station",
        "gs_all_cities": "All cities",
        "gs_fuel_price_hint": "Fuel price:",
        "gs_fuel_type": "Fuel type:",
        "brands": {
            "lukoil": "Lukoil", "gazprom": "Gazpromneft", 
            "rosneft": "Rosneft", "shell": "Shell"
        },
        "about_title": "About",
        "version": "Version 5.0  •  2024",
        "about_section1_title": "About",
        "about_section1": "Professional fuel calculator with interactive gas stations map.",
        "about_section2_title": "Features",
        "about_section2": "• Accurate consumption calculation\n• Trip cost calculation\n• Up to 25 vehicles\n• Calculation history\n• Interactive gas stations map\n• Real fuel prices",
        "about_section3_title": "How to use",
        "about_section3": "1. Register\n2. Add a vehicle\n3. Select calculation mode\n4. Enter data\n5. Press Calculate",
        "about_section4_title": "Security",
        "about_section4": "Passwords encrypted (SHA-256). Data stored locally.",
        "about_section5_title": "Interpretation",
        "about_section5": "Up to 8 L/100 km — economical\n8-12 L/100 km — average\nOver 12 L/100 km — high",
        "footer": "© 2024 Fuel Consumption Calculator",
        "no_cars_limit": "Maximum 25 vehicles",
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
        "chart_bar1": "#3fb950", "chart_bar2": "#d29922", "chart_bar3": "#f85149",
        "map_bg": "#1a2a3a",
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
        "map_bg": "#e8f0e8",
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

# ─── HTML Map Generator ───────────────────────────────────────────────────────
def generate_map_html(selected_station=None):
    """Генерирует HTML с интерактивной картой Leaflet"""
    
    # JavaScript для карты
    stations_js = json.dumps(GAS_STATIONS_RUSSIA, ensure_ascii=False)
    
    html = f'''<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Карта заправок России</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    <style>
        body {{ margin: 0; padding: 0; font-family: Arial, sans-serif; }}
        #map {{ height: 100vh; width: 100%; }}
        
        .station-popup {{
            padding: 10px;
            max-width: 300px;
        }}
        .station-popup h3 {{
            margin: 0 0 5px 0;
            color: #e53935;
        }}
        .station-popup .brand {{
            color: #666;
            font-size: 12px;
            margin-bottom: 10px;
        }}
        .station-popup .price-table {{
            width: 100%;
            border-collapse: collapse;
            margin: 10px 0;
        }}
        .station-popup .price-table td {{
            padding: 4px 8px;
            border-bottom: 1px solid #eee;
            font-size: 13px;
        }}
        .station-popup .price-table .fuel-type {{
            font-weight: bold;
        }}
        .station-popup .price-table .fuel-price {{
            color: #2e7d32;
            font-weight: bold;
            text-align: right;
        }}
        .station-popup .address {{
            color: #666;
            font-size: 11px;
            margin-top: 5px;
            border-top: 1px solid #eee;
            padding-top: 5px;
        }}
        
        .legend {{
            background: white;
            padding: 10px;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.2);
        }}
        .legend-item {{
            display: flex;
            align-items: center;
            margin: 3px 0;
            font-size: 12px;
        }}
        .legend-marker {{
            width: 12px;
            height: 12px;
            border-radius: 50%;
            margin-right: 5px;
            border: 2px solid white;
            box-shadow: 0 1px 3px rgba(0,0,0,0.3);
        }}
        
        .control-panel {{
            background: white;
            padding: 10px;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.2);
        }}
        .control-panel select {{
            padding: 5px;
            border-radius: 3px;
            border: 1px solid #ddd;
            width: 200px;
        }}
    </style>
</head>
<body>
    <div id="map"></div>
    
    <script>
        // Инициализация карты
        var map = L.map('map').setView([57.0, 60.0], 4);
        
        // Добавление слоя OpenStreetMap
        L.tileLayer('https://{{s}}.tile.openstreetmap.org/{{z}}/{{x}}/{{y}}.png', {{
            attribution: '© OpenStreetMap contributors',
            maxZoom: 18
        }}).addTo(map);
        
        // Цвета для брендов
        var brandColors = {{
            'lukoil': '#e53935',
            'gazprom': '#1e88e5',
            'rosneft': '#fdd835',
            'shell': '#f4511e'
        }};
        
        // Данные заправок
        var stations = {stations_js};
        
        // Создание маркеров
        var markers = [];
        var markerCluster = L.markerClusterGroup();
        
        stations.forEach(function(station) {{
            var color = brandColors[station.brand] || '#666';
            var icon = L.divIcon({{
                html: '<div style="background: ' + color + '; width: 12px; height: 12px; border-radius: 50%; border: 2px solid white; box-shadow: 0 1px 3px rgba(0,0,0,0.3);"></div>',
                className: 'custom-marker',
                iconSize: [12, 12],
                iconAnchor: [6, 6]
            }});
            
            var popupContent = '<div class="station-popup">' +
                '<h3>' + station.name + '</h3>' +
                '<div class="brand">' + getBrandName(station.brand) + '</div>' +
                '<table class="price-table">';
            
            // Добавление цен на топливо
            if (station.fuel_types) {{
                for (var fuel in station.fuel_types) {{
                    popupContent += '<tr>' +
                        '<td class="fuel-type">' + fuel + '</td>' +
                        '<td class="fuel-price">' + station.fuel_types[fuel] + ' ₽</td>' +
                        '</tr>';
                }}
            }}
            
            popupContent += '</table>' +
                '<div class="address">📍 ' + station.city + ', ' + station.address + '</div>' +
                '</div>';
            
            var marker = L.marker([station.lat, station.lon], {{icon: icon}})
                .bindPopup(popupContent, {{maxWidth: 300}});
            
            marker.stationData = station;
            markers.push(marker);
            markerCluster.addLayer(marker);
        }});
        
        map.addLayer(markerCluster);
        
        // Функция получения названия бренда
        function getBrandName(brand) {{
            var brands = {{
                'lukoil': '⛽ Лукойл',
                'gazprom': '⛽ Газпромнефть',
                'rosneft': '⛽ Роснефть',
                'shell': '🐚 Shell'
            }};
            return brands[brand] || brand;
        }}
        
        // Добавление легенды
        var legend = L.control({{position: 'bottomright'}});
        legend.onAdd = function(map) {{
            var div = L.DomUtil.create('div', 'legend');
            div.innerHTML = '<strong>Бренды АЗС:</strong><br>';
            
            var brands = [
                {{name: 'Лукойл', color: '#e53935'}},
                {{name: 'Газпромнефть', color: '#1e88e5'}},
                {{name: 'Роснефть', color: '#fdd835'}},
                {{name: 'Shell', color: '#f4511e'}}
            ];
            
            brands.forEach(function(brand) {{
                div.innerHTML += '<div class="legend-item">' +
                    '<div class="legend-marker" style="background: ' + brand.color + ';"></div>' +
                    brand.name +
                    '</div>';
            }});
            
            return div;
        }};
        legend.addTo(map);
        
        // Добавление панели управления
        var control = L.control({{position: 'topleft'}});
        control.onAdd = function(map) {{
            var div = L.DomUtil.create('div', 'control-panel');
            div.innerHTML = '<strong>🏙 Выбрать город:</strong><br>' +
                '<select id="city-select" onchange="zoomToCity(this.value)">' +
                '<option value="">— Все города —</option>';
            
            // Получение уникальных городов
            var cities = [...new Set(stations.map(s => s.city))].sort();
            cities.forEach(function(city) {{
                div.innerHTML += '<option value="' + city + '">' + city + '</option>';
            }});
            
            div.innerHTML += '</select>';
            return div;
        }};
        control.addTo(map);
        
        // Функция масштабирования к городу
        window.zoomToCity = function(cityName) {{
            if (!cityName) {{
                map.setView([57.0, 60.0], 4);
                return;
            }}
            
            var cityStations = stations.filter(function(s) {{
                return s.city === cityName;
            }});
            
            if (cityStations.length > 0) {{
                var bounds = L.latLngBounds(
                    cityStations.map(function(s) {{ return [s.lat, s.lon]; }})
                );
                map.fitBounds(bounds, {{padding: [50, 50], maxZoom: 12}});
            }}
        }};
        
        // Поиск ближайших заправок
        map.on('click', function(e) {{
            var latlng = e.latlng;
            var nearest = null;
            var minDist = Infinity;
            
            stations.forEach(function(station) {{
                var dist = map.distance(latlng, L.latLng(station.lat, station.lon));
                if (dist < minDist && dist < 50000) {{ // 50km radius
                    minDist = dist;
                    nearest = station;
                }}
            }});
            
            if (nearest) {{
                var popup = L.popup()
                    .setLatLng([nearest.lat, nearest.lon])
                    .setContent('<div class="station-popup">' +
                        '<h3>' + nearest.name + '</h3>' +
                        '<div>Расстояние: ' + (minDist / 1000).toFixed(1) + ' км</div>' +
                        '<div class="address">📍 ' + nearest.city + ', ' + nearest.address + '</div>' +
                        '</div>')
                    .openOn(map);
            }}
        }});
    </script>
</body>
</html>'''
    
    return html

# ─── Scroll helper ─────────────────────────────────────────────────────────────
def make_scrollable(parent, bg=None):
    """Создаёт скроллируемый контейнер с поддержкой мыши"""
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

    # Универсальный скроллинг мышкой
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

        self.data = load_data()
        self.current_user = self.data.get("current_user")

        s = self.data.get("settings", {})
        global current_theme, current_language, current_currency
        current_theme = s.get("theme", "dark")
        current_language = s.get("language", "ru")
        current_currency = s.get("currency", "₽ RUB")

        self._photo_refs = {}
        self._calc_mode = tk.StringVar(value="consumption")
        self._selected_station = None

        self._build_ui()

    def _save_settings(self):
        self.data.setdefault("settings", {})
        self.data["settings"] = {
            "theme": current_theme,
            "language": current_language,
            "currency": current_currency
        }
        save_data(self.data)

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
        for name in ["calculator", "profile", "history", "gas_stations", "about", "settings"]:
            self.sections[name] = tk.Frame(self.content, bg=t["bg"])

        self._build_calculator()
        self._build_profile()
        self._build_history()
        self._build_gas_stations()
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
            ("⛽  " + TR("gas_stations"), "gas_stations"),
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
        if name == "gas_stations":
            self._refresh_gas_stations()

    # ── Calculator (упрощен для компактности) ─────────────────────────────────
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
        if self.current_user:
            user = self.data["users"].get(self.current_user, {})
            cars += [c["name"] for c in user.get("cars", [])]
        self.calc_car_combo["values"] = cars
        self.calc_car_var.set(cars[0])

    def _on_car_selected(self, event=None):
        sel = self.calc_car_var.get()
        if sel == TR("no_car") or not self.current_user:
            self._set_calc_car_image(None)
            self._draw_fuel_chart()
            return
        for car in self.data["users"].get(self.current_user, {}).get("cars", []):
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

        if mode == "consumption":
            self.result_consumption_label.configure(text=f"{consumption:.2f} л/100 км")
        else:
            self.result_cost_label.configure(text=f"{cost:,.2f} {sym}")

        self.stat_labels["avg"].configure(text=f"{consumption:.2f} л/100 км")
        self.stat_labels["dist"].configure(text=f"{dist:.0f} км")
        self.stat_labels["fuel"].configure(text=f"{fuel:.1f} л")
        self.stat_labels["cost"].configure(text=f"{cost:,.2f} {sym}")

        if self.current_user:
            entry = {
                "date": datetime.now().strftime("%d.%m.%Y %H:%M"),
                "car": self.calc_car_var.get(),
                "distance": dist, "fuel": round(fuel, 2),
                "price": price, "currency": sym,
                "consumption": round(consumption, 2), "cost": round(cost, 2),
            }
            user = self.data["users"][self.current_user]
            user.setdefault("history", []).insert(0, entry)
            if self.calc_car_var.get() != TR("no_car"):
                for car in user.get("cars", []):
                    if car["name"] == self.calc_car_var.get():
                        car["avg_consumption"] = round(consumption, 2)
                        break
            save_data(self.data)
            self._draw_fuel_chart()

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
            d = math.sqrt((event.x - cx)**2 + (event.y - cy)**2)
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
        if not hasattr(self, 'chart_canvas'):
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
        history = self.data["users"][self.current_user].get("history", [])
        cutoff = datetime.now() - timedelta(days=days_limit)
        filtered = []
        for e in history:
            try:
                dt = datetime.strptime(e["date"], "%d.%m.%Y %H:%M")
            except Exception:
                continue
            if dt < cutoff:
                continue
            if sel == TR("no_car") or e.get("car") == sel:
                filtered.append((dt, e["consumption"]))

        if not filtered:
            c.create_text(W // 2, H // 2, text=TR("no_history_chart"),
                          fill=t["fg2"], font=("Courier", 12))
            return

        filtered.sort(key=lambda x: x[0])
        
        daily_data = {}
        for dt, val in filtered:
            day_key = dt.strftime("%Y-%m-%d")
            if day_key not in daily_data:
                daily_data[day_key] = []
            daily_data[day_key].append(val)
        
        daily_avg = []
        for day_key in sorted(daily_data.keys()):
            vals = daily_data[day_key]
            dt = datetime.strptime(day_key, "%Y-%m-%d")
            avg_val = sum(vals) / len(vals)
            daily_avg.append((dt, avg_val))

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
            c.create_rectangle(x - bar_width/2, y, x + bar_width/2, H - pb,
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
                    if email in self.data.get("users", {}):
                        status.configure(text=TR("user_exists"))
                        return
                    self.data.setdefault("users", {})[email] = {
                        "password": hash_pw(pw), "cars": [], "history": []
                    }
                    self.current_user = email
                    self.data["current_user"] = email
                    save_data(self.data)
                    self._refresh_car_combo()
                    self._refresh_profile()
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
            if email not in self.data.get("users", {}):
                status.configure(text="Email не найден")
                return
            if len(pw) < 6:
                status.configure(text=TR("password_short"))
                return
            self.data["users"][email]["password"] = hash_pw(pw)
            save_data(self.data)
            messagebox.showinfo(TR("password_changed"), TR("new_password_set"))
            dialog.destroy()

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
            self.data["users"][self.current_user]["password"] = hash_pw(np1)
            save_data(self.data)
            pw_status.configure(text=TR("password_changed"), fg=t["green"])
            self._new_pw_entry.delete(0, "end")
            self._new_pw2_entry.delete(0, "end")

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
        cars = self.data["users"][self.current_user].get("cars", [])
        if len(cars) >= 25:
            messagebox.showwarning(TR("error"), TR("no_cars_limit"))
            return

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
            car = {"name": name, "photo": photo_path_var.get() or None}
            self.data["users"][self.current_user].setdefault("cars", []).append(car)
            save_data(self.data)
            self._add_car_form_frame.pack_forget()
            self._add_car_form_visible = False
            self._render_cars()
            self._refresh_car_combo()

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
        if not self.current_user:
            return
        cars = self.data["users"][self.current_user].get("cars", [])
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
                          command=lambda i=idx: self._delete_car(i)).pack(pady=(3, 0))

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

        outer, inner = make_scrollable(f, t["bg"])
        outer.pack(fill="both", expand=True)
        pad = tk.Frame(inner, bg=t["bg"])
        pad.pack(fill="both", expand=True, padx=24, pady=20)

        tk.Label(pad, text=TR("history_title"), font=("Georgia", 16, "bold"),
                 bg=t["bg"], fg=t["fg"]).pack(anchor="w", pady=(0, 4))

        if not self.current_user:
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
        if not self.current_user:
            return
        history = self.data["users"][self.current_user].get("history", [])
        if not history:
            tk.Label(self._hist_inner, text=TR("history_empty"),
                     font=("Courier", 12), bg=t["bg"], fg=t["fg2"]).pack(pady=40)
            return

        for i, entry in enumerate(history):
            card = tk.Frame(self._hist_inner, bg=t["bg2"], pady=8)
            card.pack(fill="x", pady=6, ipadx=8)

            ph = None
            car_name = entry.get("car", "—")
            if car_name not in (TR("no_car"), "— Без авто —", "— No car —"):
                for car in self.data["users"].get(self.current_user, {}).get("cars", []):
                    if car["name"] == car_name and car.get("photo"):
                        ph = car["photo"]
                        break

            img_box = tk.Frame(card, bg=t["bg2"], width=200, height=120)
            img_box.pack_propagate(False)
            img_box.pack(side="left", padx=12)
            img = load_car_image(ph, 200, 120)
            photo = ImageTk.PhotoImage(img)
            self._photo_refs[f"hist_car_{i}"] = photo
            tk.Label(img_box, image=photo, bg=t["bg2"]).place(relx=0.5, rely=0.5, anchor="center")

            info = tk.Frame(card, bg=t["bg2"])
            info.pack(side="left", fill="x", expand=True, padx=8)
            sym = entry.get("currency", "₽")
            tk.Label(info, text=f"📅 {entry['date']}",
                     font=("Courier", 10, "bold"), bg=t["bg2"], fg=t["fg"]).pack(anchor="w")
            tk.Label(info, text=f"🚗 {car_name}",
                     font=("Courier", 10), bg=t["bg2"], fg=t["accent"]).pack(anchor="w")
            tk.Label(info,
                     text=f"📏 {entry['distance']} км  ⛽ {entry['fuel']} л  💰 {entry['price']} {sym}/л",
                     font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", pady=2)
            tk.Label(info,
                     text=f"💧 {entry['consumption']} л/100км  💳 {entry['cost']:,.2f} {sym}",
                     font=("Courier", 11, "bold"), bg=t["bg2"], fg=t["green"]).pack(anchor="w")

            btn_col = tk.Frame(card, bg=t["bg2"], width=110)
            btn_col.pack_propagate(False)
            btn_col.pack(side="right", padx=12, pady=4)

            copy_text = (
                f"Дата: {entry['date']}\n"
                f"Авто: {car_name}\n"
                f"Расстояние: {entry['distance']} км\n"
                f"Топливо: {entry['fuel']} л\n"
                f"Цена: {entry['price']} {sym}/л\n"
                f"Расход: {entry['consumption']} л/100км\n"
                f"Стоимость: {entry['cost']:,.2f} {sym}"
            )

            def copy_entry(txt=copy_text):
                self.clipboard_clear()
                self.clipboard_append(txt)
                messagebox.showinfo(TR("copied"), TR("copied_msg"))

            tk.Button(btn_col, text=TR("copy"), font=("Courier", 9),
                      bg=t["green"], fg="white", bd=0, cursor="hand2",
                      command=copy_entry, width=12, height=2).pack(fill="x", pady=(0, 4))
            tk.Button(btn_col, text=TR("delete_entry"), font=("Courier", 9),
                      bg=t["red"], fg="white", bd=0, cursor="hand2",
                      command=lambda idx=i: self._delete_history_entry(idx),
                      width=12, height=2).pack(fill="x")

    def _delete_history_entry(self, idx):
        if messagebox.askyesno(TR("delete_confirm"), "Удалить эту запись?"):
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

    # ─── Gas Stations with interactive map ─────────────────────────────────────
    def _build_gas_stations(self):
        t = T()
        f = self.sections["gas_stations"]
        for w in f.winfo_children():
            w.destroy()

        outer, inner = make_scrollable(f, t["bg"])
        outer.pack(fill="both", expand=True)
        pad = tk.Frame(inner, bg=t["bg"])
        pad.pack(fill="both", expand=True, padx=48, pady=32)

        tk.Label(pad, text=TR("gs_title"), font=("Georgia", 18, "bold"),
                 bg=t["bg"], fg=t["fg"]).pack(anchor="w", pady=(0, 8))

        # Кнопка открытия интерактивной карты
        map_card = tk.Frame(pad, bg=t["bg2"], pady=16)
        map_card.pack(fill="x", pady=8)
        
        tk.Label(map_card, text="🌍 Интерактивная карта заправок",
                 font=("Georgia", 14, "bold"), bg=t["bg2"], fg=t["fg"]).pack(pady=(0, 8))
        tk.Label(map_card, text="Нажмите на карту для поиска ближайших заправок\n"
                 "Кликайте по маркерам для просмотра цен на топливо",
                 font=("Courier", 10), bg=t["bg2"], fg=t["fg2"],
                 justify="center").pack(pady=(0, 12))
        
        def open_map():
            # Генерируем HTML с картой
            html = generate_map_html()
            # Сохраняем во временный файл
            map_path = os.path.join(MAP_DIR, "gas_stations_map.html")
            with open(map_path, "w", encoding="utf-8") as f:
                f.write(html)
            # Открываем в браузере
            webbrowser.open(f"file://{map_path}")
        
        tk.Button(map_card, text=TR("gs_open_map"),
                  font=("Georgia", 14, "bold"),
                  bg=t["btn"], fg="white", bd=0, pady=15, cursor="hand2",
                  command=open_map).pack(fill="x", padx=20, pady=10)

        # Поиск по городу
        search_card = tk.Frame(pad, bg=t["bg2"], pady=16)
        search_card.pack(fill="x", pady=8)
        tk.Label(search_card, text=TR("gs_city"), font=("Georgia", 13, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=20, pady=(0, 8))

        search_row = tk.Frame(search_card, bg=t["bg2"])
        search_row.pack(fill="x", padx=20)
        
        cities = sorted(list(set([s["city"] for s in GAS_STATIONS_RUSSIA])))
        
        search_var = tk.StringVar()
        search_entry = ttk.Combobox(search_row, textvariable=search_var,
                                    values=cities, font=("Courier", 12), width=30)
        search_entry.pack(side="left", padx=(0, 10))
        
        def search_city():
            city = search_var.get().strip()
            if city:
                city_stations = [s for s in GAS_STATIONS_RUSSIA if s["city"] == city]
                if city_stations:
                    self._show_city_stations(city, city_stations)
                else:
                    self._station_info_text.configure(text=f"Заправки в городе {city} не найдены")
        
        tk.Button(search_row, text=TR("gs_search"), font=("Courier", 11),
                  bg=t["btn"], fg="white", bd=0, padx=15, pady=8,
                  command=search_city).pack(side="left")
        tk.Button(search_row, text=TR("gs_open_map"), font=("Courier", 11),
                  bg=t["btn2"], fg=t["fg"], bd=0, padx=15, pady=8,
                  command=open_map).pack(side="left", padx=(10, 0))

        # Информация о заправках
        info_card = tk.Frame(pad, bg=t["bg2"], pady=16)
        info_card.pack(fill="x", pady=8)
        self._station_info_text = tk.Label(info_card,
                                           text=TR("gs_click_hint"),
                                           font=("Courier", 11), bg=t["bg2"], fg=t["fg2"],
                                           justify="left", wraplength=800)
        self._station_info_text.pack(anchor="w", padx=20, pady=8)

        # Выбор авто
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

        # Расстояние
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
                 bg=t["input_bg"], fg=t["fg"], bd=0).pack(side="left", fill="x", expand=True, pady=8)
        tk.Label(dist_row, text="км", font=("Courier", 10), bg=t["input_bg"], fg=t["fg2"], padx=8).pack(side="right")

        # Цена топлива
        price_card = tk.Frame(pad, bg=t["bg2"], pady=16)
        price_card.pack(fill="x", pady=8)
        tk.Label(price_card, text=TR("gs_fuel_price_hint"), font=("Georgia", 13, "bold"),
                 bg=t["bg2"], fg=t["fg"]).pack(anchor="w", padx=20, pady=(0, 8))
        price_row = tk.Frame(price_card, bg=t["input_bg"], bd=1, relief="solid")
        price_row.pack(fill="x", padx=20, pady=(0, 12))
        tk.Label(price_row, text="💰", font=("Courier", 13), bg=t["input_bg"], fg=t["fg2"], padx=8).pack(side="left")
        self.gs_price_var = tk.StringVar()
        tk.Entry(price_row, textvariable=self.gs_price_var, font=("Courier", 13),
                 bg=t["input_bg"], fg=t["fg"], bd=0).pack(side="left", fill="x", expand=True, pady=8)
        tk.Label(price_row, text=f"{get_currency_symbol()}/л", font=("Courier", 10),
                 bg=t["input_bg"], fg=t["fg2"], padx=8).pack(side="right")

        # Кнопка
        tk.Button(pad, text=TR("gs_to_calc"), font=("Georgia", 12, "bold"),
                  bg=t["btn"], fg="white", bd=0, pady=12,
                  command=self._gs_to_calculator).pack(fill="x", pady=16)

    def _show_city_stations(self, city, stations):
        """Показывает информацию о заправках в городе"""
        t = T()
        info_text = f"🏙 Заправки в городе {city}:\n\n"
        
        for station in stations[:10]:
            brand = TR("brands").get(station["brand"], station["brand"])
            info_text += f"⛽ {brand} — {station['name']}\n"
            info_text += f"   📍 {station['address']}\n"
            
            if "fuel_types" in station:
                info_text += "   💰 Цены:\n"
                for fuel_type, price in station["fuel_types"].items():
                    info_text += f"      {fuel_type}: {price} ₽/л\n"
            
            info_text += "\n"
        
        if len(stations) > 10:
            info_text += f"... и ещё {len(stations) - 10} заправок\n"
        
        info_text += f"\n💡 Откройте интерактивную карту для просмотра всех заправок на карте"
        
        self._station_info_text.configure(text=info_text, fg=t["fg"])
        
        if stations and "fuel_types" in stations[0]:
            # Берем цену АИ-95 по умолчанию
            price = stations[0]["fuel_types"].get("АИ-95", 50)
            self.gs_price_var.set(str(price))

    def _refresh_gs_car_combo(self):
        cars = [TR("no_car")]
        if self.current_user:
            user = self.data["users"].get(self.current_user, {})
            cars += [c["name"] for c in user.get("cars", [])]
        if hasattr(self, 'gs_car_combo'):
            self.gs_car_combo["values"] = cars
            self.gs_car_var.set(cars[0])

    def _on_gs_car_selected(self, event=None):
        self._update_gs_avg_display()

    def _update_gs_avg_display(self):
        if not hasattr(self, 'gs_avg_lbl'):
            return
        sel = self.gs_car_var.get() if hasattr(self, 'gs_car_var') else TR("no_car")
        if sel == TR("no_car") or not self.current_user:
            self.gs_avg_lbl.configure(text="")
            return
        for car in self.data["users"].get(self.current_user, {}).get("cars", []):
            if car["name"] == sel:
                avg = car.get("avg_consumption")
                if avg:
                    self.gs_avg_lbl.configure(text=f"💧 {TR('gs_avg_from_car')}: {avg} л/100 км", fg=T()["accent"])
                    return
        self.gs_avg_lbl.configure(text=TR("gs_no_avg"), fg=T()["fg2"])

    def _gs_to_calculator(self):
        ds = self.gs_dist_var.get().strip()
        ps = self.gs_price_var.get().strip()
        if not ds:
            messagebox.showwarning(TR("error"), TR("gs_enter_distance"))
            return
        self._show_section("calculator")
        if ds:
            self.dist_var.set(ds)
        if ps:
            self.price_var.set(ps)
        sel = self.gs_car_var.get()
        if sel != TR("no_car") and self.current_user:
            for car in self.data["users"].get(self.current_user, {}).get("cars", []):
                if car["name"] == sel:
                    avg = car.get("avg_consumption")
                    if avg:
                        self._calc_mode.set("cost")
                        self._build_calc_form()
                        self._update_result_widgets("cost")
                        self.avg_var.set(str(avg))
                    self.calc_car_var.set(sel)
                    self._on_car_selected()
                    break

    def _refresh_gas_stations(self):
        self._refresh_gs_car_combo()
        self._update_gs_avg_display()

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
