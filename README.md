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

try:
    from PIL import Image

    PILLOW_AVAILABLE = True
except ImportError:
    PILLOW_AVAILABLE = False

# DATABASE CONNECTION SETTINGS
DB_CONFIG = {
    'dbname': 'fuel_calc',
    'user': 'postgres',
    'password': '12345',
    'host': 'localhost',
    'port': 5432,
}

# ADMIN SETTINGS
ADMIN_CONFIG = {
    'email': 'admin@fullcalcpro.com',
    'password': 'admin123456',
    'name': 'Administrator'
}

connection_pool = None
ALLOWED_EXTENSIONS = {'jpg', 'jpeg', 'png', 'gif', 'bmp', 'webp', 'tiff', 'ico'}
MAX_IMAGE_SIZE = (1200, 1200)

EXCHANGE_RATES = {
    '₽ RUB': 1.0,
    '$ USD': 0.011,
    '€ EUR': 0.010
}


def init_db_pool():
    """Initialize database connection pool"""
    global connection_pool
    try:
        connection_pool = pool.SimpleConnectionPool(1, 20, **DB_CONFIG)
        print("✓ Database connection pool created")
        return True
    except Exception as e:
        print(f"✗ Database connection error: {e}")
        return False


@contextmanager
def get_db_connection():
    """Context manager for database connections"""
    conn = connection_pool.getconn()
    try:
        yield conn
    except Exception:
        conn.rollback()
        raise
    finally:
        connection_pool.putconn(conn)


def db_exec(sql, params=None, fetch=False):
    """Execute a database query"""
    with get_db_connection() as conn:
        with conn.cursor() as cur:
            cur.execute(sql, params)
            if fetch:
                return cur.fetchall()
            conn.commit()


def create_tables():
    """Create database tables if they don't exist"""
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
                CREATE TABLE IF NOT EXISTS admin_logs (
                    id SERIAL PRIMARY KEY,
                    admin_email TEXT,
                    action TEXT,
                    target_email TEXT,
                    details TEXT,
                    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                );
            """)

            try:
                cur.execute("ALTER TABLE users ADD COLUMN IF NOT EXISTS created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP")
                cur.execute("ALTER TABLE users ADD COLUMN IF NOT EXISTS is_admin BOOLEAN DEFAULT FALSE")
                cur.execute("ALTER TABLE users ADD COLUMN IF NOT EXISTS is_active BOOLEAN DEFAULT TRUE")
                cur.execute("ALTER TABLE users ADD COLUMN IF NOT EXISTS last_login TIMESTAMP")
                cur.execute("ALTER TABLE users ADD COLUMN IF NOT EXISTS deleted_at TIMESTAMP")
                cur.execute("ALTER TABLE users ADD COLUMN IF NOT EXISTS deleted_by TEXT")
                print("✓ Users columns checked/added")
            except Exception as e:
                print(f"⚠ Warning adding columns: {e}")

        conn.commit()
    print("✓ Database tables checked/created")


def create_admin():
    """Create or update admin user"""
    try:
        exists = db_exec(
            "SELECT 1 FROM users WHERE email=%s AND deleted_at IS NULL",
            (ADMIN_CONFIG['email'],),
            fetch=True
        )

        if not exists:
            try:
                db_exec("""
                    INSERT INTO users (email, password, is_admin, is_active) 
                    VALUES (%s, %s, TRUE, TRUE)
                """, (ADMIN_CONFIG['email'], hash_pw(ADMIN_CONFIG['password'])))
            except:
                db_exec(
                    "INSERT INTO users (email, password) VALUES (%s, %s)",
                    (ADMIN_CONFIG['email'], hash_pw(ADMIN_CONFIG['password']))
                )
                try:
                    db_exec(
                        "UPDATE users SET is_admin=TRUE, is_active=TRUE WHERE email=%s",
                        (ADMIN_CONFIG['email'],)
                    )
                except:
                    pass

            settings_exist = db_exec(
                "SELECT 1 FROM settings WHERE email=%s",
                (ADMIN_CONFIG['email'],),
                fetch=True
            )
            if not settings_exist:
                db_exec(
                    "INSERT INTO settings (email) VALUES (%s)",
                    (ADMIN_CONFIG['email'],)
                )

            print(f"✓ Administrator created: {ADMIN_CONFIG['email']}")
        else:
            try:
                db_exec(
                    "UPDATE users SET is_admin=TRUE, is_active=TRUE, deleted_at=NULL, deleted_by=NULL WHERE email=%s",
                    (ADMIN_CONFIG['email'],)
                )
            except:
                pass

            db_exec(
                "UPDATE users SET password=%s WHERE email=%s",
                (hash_pw(ADMIN_CONFIG['password']), ADMIN_CONFIG['email'])
            )

            settings_exist = db_exec(
                "SELECT 1 FROM settings WHERE email=%s",
                (ADMIN_CONFIG['email'],),
                fetch=True
            )
            if not settings_exist:
                db_exec(
                    "INSERT INTO settings (email) VALUES (%s)",
                    (ADMIN_CONFIG['email'],)
                )
            print(f"✓ Administrator updated: {ADMIN_CONFIG['email']}")

        print(f"  Password: {ADMIN_CONFIG['password']}")
        return True
    except Exception as e:
        print(f"✗ Error creating admin: {e}")
        return False


def log_admin_action(admin_email, action, target_email=None, details=None):
    """Log admin actions"""
    try:
        db_exec("""
            INSERT INTO admin_logs (admin_email, action, target_email, details)
            VALUES (%s, %s, %s, %s)
        """, (admin_email, action, target_email, details))
    except Exception as e:
        print(f"⚠ Logging error: {e}")


def allowed_file(filename):
    """Check if file extension is allowed"""
    return '.' in filename and filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS


def process_image(image_path):
    """Process and optimize uploaded image"""
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
        print(f"Image processing error: {e}")
        return False


def save_car_photo(photo_file, old_photo_path=None):
    """Save car photo to uploads directory"""
    if not photo_file or not allowed_file(photo_file.filename):
        return None

    if old_photo_path:
        old_path = os.path.join('static', 'uploads', old_photo_path)
        if os.path.exists(old_path):
            os.remove(old_path)

    safe_name = secure_filename(photo_file.filename)
    name_part, ext = os.path.splitext(safe_name)
    hash_part = hashlib.md5(name_part.encode()).hexdigest()[:8]
    new_name = f"car_{datetime.now().strftime('%Y%m%d%H%M%S')}_{hash_part}{ext}"
    file_path = os.path.join('static', 'uploads', new_name)
    photo_file.save(file_path)
    process_image(file_path)
    return new_name


def db_create_user(email, pw_hash, is_admin=False):
    """Create a new user"""
    try:
        db_exec(
            "INSERT INTO users (email, password, is_admin) VALUES (%s,%s,%s)",
            (email, pw_hash, is_admin)
        )
    except:
        db_exec(
            "INSERT INTO users (email, password) VALUES (%s,%s)",
            (email, pw_hash)
        )
        if is_admin:
            try:
                db_exec(
                    "UPDATE users SET is_admin=TRUE WHERE email=%s",
                    (email,)
                )
            except:
                pass
    db_exec("INSERT INTO settings (email) VALUES (%s)", (email,))


def db_get_user(email):
    """Get user password hash"""
    rows = db_exec(
        "SELECT password FROM users WHERE email=%s AND deleted_at IS NULL",
        (email,),
        fetch=True
    )
    return rows[0][0] if rows else None


def db_user_exists(email):
    """Check if user exists"""
    return bool(db_exec(
        "SELECT 1 FROM users WHERE email=%s AND deleted_at IS NULL",
        (email,),
        fetch=True
    ))


def db_is_user_active(email):
    """Check if user is active"""
    try:
        rows = db_exec(
            "SELECT is_active FROM users WHERE email=%s AND deleted_at IS NULL",
            (email,),
            fetch=True
        )
        if rows and rows[0][0] is not None:
            return rows[0][0]
        return True
    except:
        return True


def db_is_user_admin(email):
    """Check if user is admin"""
    try:
        rows = db_exec(
            "SELECT is_admin FROM users WHERE email=%s AND deleted_at IS NULL",
            (email,),
            fetch=True
        )
        if rows and rows[0][0] is not None:
            return rows[0][0]
        return False
    except:
        return False


def db_is_user_deleted(email):
    """Check if user is deleted"""
    try:
        rows = db_exec(
            "SELECT deleted_at, deleted_by FROM users WHERE email=%s",
            (email,),
            fetch=True
        )
        if rows and rows[0][0] is not None:
            return True, rows[0][1]
        return False, None
    except:
        return False, None


def db_update_password(email, pw_hash):
    """Update user password"""
    db_exec(
        "UPDATE users SET password=%s WHERE email=%s AND deleted_at IS NULL",
        (pw_hash, email)
    )


def db_delete_user(email):
    """Soft delete user account"""
    db_exec("""
        UPDATE users 
        SET deleted_at = CURRENT_TIMESTAMP, 
            is_active = FALSE,
            deleted_by = NULL
        WHERE email = %s
    """, (email,))


def db_restore_user(email):
    """Restore deleted user"""
    db_exec("""
        UPDATE users 
        SET deleted_at = NULL, 
            is_active = TRUE,
            deleted_by = NULL
        WHERE email = %s
    """, (email,))


def db_get_all_users():
    """Get all non-admin users"""
    try:
        rows = db_exec("""
            SELECT u.email, u.created_at, u.is_admin, u.is_active, u.last_login,
                   u.deleted_at, u.deleted_by,
                   COUNT(DISTINCT c.id) as car_count,
                   COUNT(DISTINCT h.id) as history_count
            FROM users u
            LEFT JOIN cars c ON u.email = c.email
            LEFT JOIN history h ON u.email = h.email
            WHERE (u.is_admin = FALSE OR u.is_admin IS NULL)
              AND u.deleted_at IS NULL
            GROUP BY u.email, u.created_at, u.is_admin, u.is_active, u.last_login, u.deleted_at, u.deleted_by
            ORDER BY u.created_at DESC
        """, fetch=True)
    except:
        rows = db_exec("""
            SELECT u.email, NULL, FALSE, TRUE, NULL, NULL, NULL,
                   COUNT(DISTINCT c.id) as car_count,
                   COUNT(DISTINCT h.id) as history_count
            FROM users u
            LEFT JOIN cars c ON u.email = c.email
            LEFT JOIN history h ON u.email = h.email
            WHERE u.deleted_at IS NULL
            GROUP BY u.email
            ORDER BY u.email
        """, fetch=True)

    return [
        {
            'email': r[0],
            'created_at': r[1].strftime("%d.%m.%Y %H:%M") if r[1] else "N/A",
            'is_admin': r[2] if r[2] is not None else False,
            'is_active': r[3] if r[3] is not None else True,
            'last_login': r[4].strftime("%d.%m.%Y %H:%M") if r[4] else "Never",
            'deleted_at': r[5].strftime("%d.%m.%Y %H:%M") if r[5] else None,
            'deleted_by': r[6],
            'car_count': r[7],
            'history_count': r[8]
        } for r in rows
    ]


def db_get_deleted_users():
    """Get all deleted users"""
    try:
        rows = db_exec("""
            SELECT u.email, u.created_at, u.deleted_at,
                   COUNT(DISTINCT c.id) as car_count,
                   COUNT(DISTINCT h.id) as history_count
            FROM users u
            LEFT JOIN cars c ON u.email = c.email
            LEFT JOIN history h ON u.email = h.email
            WHERE u.deleted_at IS NOT NULL
              AND (u.is_admin = FALSE OR u.is_admin IS NULL)
            GROUP BY u.email, u.created_at, u.deleted_at
            ORDER BY u.deleted_at DESC
        """, fetch=True)
    except:
        return []

    return [
        {
            'email': r[0],
            'created_at': r[1].strftime("%d.%m.%Y %H:%M") if r[1] else "N/A",
            'deleted_at': r[2].strftime("%d.%m.%Y %H:%M") if r[2] else "N/A",
            'car_count': r[3],
            'history_count': r[4]
        } for r in rows
    ]


def db_get_user_cars(email):
    """Get user's cars"""
    return db_get_cars(email)


def db_get_user_history(email):
    """Get user's history"""
    return db_get_history(email)


def db_delete_car_photo(email, car_name):
    """Delete car photo"""
    rows = db_exec(
        "SELECT photo_path FROM cars WHERE email=%s AND name=%s",
        (email, car_name),
        fetch=True
    )
    if rows and rows[0][0]:
        fp = os.path.join('static', 'uploads', rows[0][0])
        if os.path.exists(fp):
            os.remove(fp)
    db_exec(
        "UPDATE cars SET photo_path=NULL WHERE email=%s AND name=%s",
        (email, car_name)
    )


def db_get_system_stats():
    """Get system statistics"""
    stats = {}
    try:
        stats['total_users'] = db_exec(
            "SELECT COUNT(*) FROM users WHERE (is_admin = FALSE OR is_admin IS NULL) AND deleted_at IS NULL",
            fetch=True
        )[0][0]
        stats['active_users'] = db_exec("""
            SELECT COUNT(*) FROM users 
            WHERE (is_active = TRUE OR is_active IS NULL)
            AND (is_admin = FALSE OR is_admin IS NULL)
            AND deleted_at IS NULL
        """, fetch=True)[0][0]
        stats['blocked_users'] = db_exec("""
            SELECT COUNT(*) FROM users 
            WHERE is_active = FALSE
            AND (is_admin = FALSE OR is_admin IS NULL)
            AND deleted_at IS NULL
        """, fetch=True)[0][0]
        stats['deleted_users'] = db_exec(
            "SELECT COUNT(*) FROM users WHERE deleted_at IS NOT NULL AND (is_admin = FALSE OR is_admin IS NULL)",
            fetch=True
        )[0][0]
    except:
        stats['total_users'] = db_exec(
            "SELECT COUNT(*) FROM users WHERE deleted_at IS NULL",
            fetch=True
        )[0][0]
        stats['active_users'] = stats['total_users']
        stats['blocked_users'] = 0
        stats['deleted_users'] = 0

    stats['total_cars'] = db_exec("SELECT COUNT(*) FROM cars", fetch=True)[0][0]
    stats['total_history'] = db_exec("SELECT COUNT(*) FROM history", fetch=True)[0][0]

    result = db_exec("SELECT COALESCE(SUM(distance),0) FROM history", fetch=True)[0][0]
    stats['total_distance'] = round(result, 1)

    result = db_exec("SELECT COALESCE(SUM(cost),0) FROM history", fetch=True)[0][0]
    stats['total_cost'] = round(result, 2)

    return stats


def db_block_user(email):
    """Block user"""
    try:
        db_exec(
            "UPDATE users SET is_active = FALSE WHERE email = %s AND deleted_at IS NULL",
            (email,)
        )
    except:
        pass


def db_unblock_user(email):
    """Unblock user"""
    try:
        db_exec(
            "UPDATE users SET is_active = TRUE WHERE email = %s AND deleted_at IS NULL",
            (email,)
        )
    except:
        pass


def db_get_cars(email):
    """Get user's cars list"""
    rows = db_exec(
        "SELECT name, photo_path, avg_consumption FROM cars WHERE email=%s ORDER BY id",
        (email,),
        fetch=True
    )
    return [{"name": r[0], "photo": r[1], "avg_consumption": r[2]} for r in rows]


def db_add_car(email, name, photo=None):
    """Add a car"""
    db_exec(
        "INSERT INTO cars (email,name,photo_path) VALUES (%s,%s,%s)",
        (email, name, photo)
    )


def db_update_car(email, old_name, new_name, photo=None):
    """Update car information"""
    if photo:
        db_exec(
            "UPDATE cars SET name=%s,photo_path=%s WHERE email=%s AND name=%s",
            (new_name, photo, email, old_name)
        )
    else:
        db_exec(
            "UPDATE cars SET name=%s WHERE email=%s AND name=%s",
            (new_name, email, old_name)
        )

    # Update car name in history
    db_exec(
        "UPDATE history SET car_name=%s WHERE email=%s AND car_name=%s",
        (new_name, email, old_name)
    )


def db_delete_car(email, name):
    """Delete a car"""
    # Delete photo if exists
    rows = db_exec(
        "SELECT photo_path FROM cars WHERE email=%s AND name=%s",
        (email, name),
        fetch=True
    )
    if rows and rows[0][0]:
        fp = os.path.join('static', 'uploads', rows[0][0])
        if os.path.exists(fp):
            os.remove(fp)

    # Update history - mark entries as "no car"
    no_car_text = "— Без авто —" if session.get('language', 'ru') == 'ru' else "— No car —"
    db_exec(
        "UPDATE history SET car_name=%s WHERE email=%s AND car_name=%s",
        (no_car_text, email, name)
    )

    # Delete car from cars table
    db_exec("DELETE FROM cars WHERE email=%s AND name=%s", (email, name))


def db_update_avg(email, car_name):
    """Update average consumption for car"""
    rows = db_exec(
        "SELECT AVG(consumption) FROM history WHERE email=%s AND car_name=%s AND consumption IS NOT NULL",
        (email, car_name),
        fetch=True
    )
    if rows and rows[0][0]:
        db_exec(
            "UPDATE cars SET avg_consumption=%s WHERE email=%s AND name=%s",
            (round(rows[0][0], 2), email, car_name)
        )


def db_add_history(email, entry):
    """Add history entry"""
    db_exec(
        """INSERT INTO history (email,car_name,distance,fuel,price,currency,consumption,cost,date)
           VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s)""",
        (
            email, entry['car'], entry['distance'], entry['fuel'], entry['price'],
            entry.get('currency'), entry['consumption'], entry['cost'],
            datetime.strptime(entry['date'], "%d.%m.%Y %H:%M")
        )
    )


def db_get_history(email, car_name=None):
    """Get history entries"""
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
        {
            'id': r[0], 'car': r[1], 'distance': r[2], 'fuel': r[3], 'price': r[4],
            'currency': r[5], 'consumption': r[6], 'cost': r[7], 'date': r[8], 'photo': r[9]
        }
        for r in rows
    ]


def db_clear_history(email):
    """Clear all history"""
    db_exec("DELETE FROM history WHERE email=%s", (email,))


def db_delete_history_entry(email, eid):
    """Delete specific history entry"""
    if db_exec("SELECT 1 FROM history WHERE id=%s AND email=%s", (eid, email), fetch=True):
        db_exec("DELETE FROM history WHERE email=%s AND id=%s", (email, eid))


def db_get_settings(email):
    """Get user settings"""
    rows = db_exec(
        "SELECT theme,language,currency FROM settings WHERE email=%s",
        (email,),
        fetch=True
    )
    if rows:
        return {'theme': rows[0][0], 'language': rows[0][1], 'currency': rows[0][2]}
    return {'theme': 'light', 'language': 'ru', 'currency': '₽ RUB'}


def db_update_settings(email, theme, lang, cur):
    """Update user settings"""
    db_exec(
        "UPDATE settings SET theme=%s,language=%s,currency=%s WHERE email=%s",
        (theme, lang, cur, email)
    )


def db_get_car_stats(email, car_name):
    """Get car statistics"""
    rows = db_exec(
        "SELECT SUM(distance),SUM(cost),AVG(consumption) FROM history WHERE email=%s AND car_name=%s",
        (email, car_name),
        fetch=True
    )
    if rows and rows[0][0]:
        return {
            'total_distance': round(rows[0][0], 1),
            'total_cost': round(rows[0][1], 2),
            'avg_consumption': round(rows[0][2], 2)
        }
    return {'total_distance': 0, 'total_cost': 0, 'avg_consumption': 0}


def hash_pw(pw):
    """Hash password using SHA-256"""
    return hashlib.sha256(pw.encode()).hexdigest()


def is_valid_email(e):
    """Validate email format"""
    return re.match(r"^[^@\s]+@[^@\s]+\.[^@\s]+$", e) is not None


def render_profile_cars():
    """Render profile cars section"""
    try:
        cars = db_get_cars(session['user'])
        car_stats = {c['name']: db_get_car_stats(session['user'], c['name']) for c in cars}
    except:
        cars, car_stats = [], {}

    return render_template(
        "profile_cars.html",
        cars=cars,
        car_stats=car_stats,
        lang=session.get('language', 'ru'),
        user=session['user'],
        currency_symbol=CURRENCIES.get(session.get('currency', '₽ RUB'), '₽')
    )


def tr(key):
    """Translate key based on current language"""
    lang = session.get('language', 'ru')
    return TRANSLATIONS.get(lang, TRANSLATIONS['ru']).get(key, key)


# Translation dictionaries
TRANSLATIONS = {
    "ru": {
        "app_title": "FullCalcPro",
        "calculator": "Калькулятор",
        "profile": "Профиль",
        "login_register": "Вход / Регистрация",
        "history": "История",
        "about": "О приложении",
        "settings": "Настройки",
        "admin_panel": "Админ-панель",
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
        "account_blocked": "Ваш аккаунт заблокирован администратором",
        "account_deleted": "Ваш аккаунт был удалён. Для восстановления обратитесь к администратору admin@fullcalcpro.com",
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
        "delete_account_msg": "Ваш аккаунт будет деактивирован",
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
        "password_changed": "Пароль успешно изменён!",
        "total_distance": "Общий пробег",
        "total_cost": "Общие затраты",
        "clear_form_confirm": "Очистить форму?",
        "clear_form_msg": "Вы уверены?",
        "login_required": "Войдите в систему для сохранения расчётов",
        "lock_history": "Войдите для просмотра истории",
        "admin_users": "Пользователи",
        "admin_stats": "Статистика",
        "admin_actions": "Действия",
        "admin_delete_photo": "Удалить фото",
        "admin_view_cars": "Авто",
        "admin_view_history": "История",
        "admin_total_users": "Всего пользователей",
        "admin_total_cars": "Всего авто",
        "admin_total_history": "Всего записей",
        "admin_active_users": "Активных",
        "admin_blocked_users": "Заблокировано",
        "admin_deleted_users": "Удалено",
        "block_user": "Заблокировать",
        "unblock_user": "Разблокировать",
        "deleted_users_title": "Удалённые пользователи",
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
        "created_at": "Создан",
        "active": "Активен",
        "blocked": "Заблокирован",
        "back": "Назад",
        "restore": "Восстановить",
        "cars": "Авто",
        "records": "Записей",
        "no_users": "Нет зарегистрированных пользователей",
        "no_deleted_users": "Нет удалённых пользователей",
        "no_user_cars": "У пользователя нет автомобилей",
        "empty_history": "История расчётов пуста",
        "registered": "Зарегистрирован",
        "deleted_on": "Удалён",
        "view_cars": "Авто",
        "view_history": "История",
        "delete_photo": "Удалить фото",
        "delete_car": "Удалить авто",
        "confirm_block_user": "Заблокировать пользователя?",
        "confirm_block_user_msg": "Пользователь не сможет войти в систему.",
        "confirm_delete_photo": "Удалить фото этого автомобиля?",
        "confirm_delete_car": "Удалить автомобиль",
        "price_positive": "Цена должна быть больше 0",
        "fuel_positive": "Расход топлива должен быть больше 0",
        "photo_optional": "Оставьте пустым, чтобы не менять фото",
        "account_deactivated": "Ваш аккаунт деактивирован.",
        "access_denied": "Доступ запрещён",
        "data_load_error": "Ошибка загрузки данных",
        "cannot_block_self": "Нельзя заблокировать себя",
        "restore_user_confirm": "Восстановить пользователя?",
        "restore_error": "Ошибка восстановления",
        "user_restored": "Пользователь восстановлен",
        "invalid_params": "Неверные параметры",
        "deleted_users_info": "Пользователи, которые удалили свой аккаунт. Данные сохранены в базе данных.",
    },
    "en": {
        "app_title": "FullCalcPro",
        "calculator": "Calculator",
        "profile": "Profile",
        "login_register": "Login / Register",
        "history": "History",
        "about": "About",
        "settings": "Settings",
        "admin_panel": "Admin Panel",
        "no_car": "-- No car --",
        "car": "Car",
        "avg_consumption": "Average consumption",
        "distance": "Distance",
        "fuel": "Fuel",
        "cost": "Cost",
        "calculate": "Calculate",
        "clear": "Clear",
        "fuel_consumption": "Fuel consumption",
        "trip_cost": "Trip cost",
        "distance_label": "Distance",
        "fuel_used": "Fuel used",
        "fuel_price": "Fuel price per liter",
        "avg_consumption_label": "Average consumption",
        "calc_mode_consumption": "Consumption per 100 km",
        "calc_mode_cost": "Trip cost",
        "login": "Login",
        "register": "Register",
        "create_account": "Create Account",
        "login_account": "Sign In",
        "email": "Email",
        "password": "Password",
        "repeat_password": "Repeat Password",
        "fill_all": "Fill in all fields",
        "invalid_email": "Invalid email",
        "password_short": "Password too short (min. 6 characters)",
        "passwords_mismatch": "Passwords do not match",
        "user_exists": "User already exists",
        "wrong_credentials": "Wrong email or password",
        "account_blocked": "Your account has been blocked by the administrator",
        "account_deleted": "Your account has been deleted. Contact admin@fullcalcpro.com for recovery",
        "logout": "Logout",
        "delete_account": "Delete Account",
        "change_password": "Change Password",
        "old_password": "Old Password",
        "new_password": "New Password",
        "repeat_new_password": "Repeat New Password",
        "change_password_btn": "Change Password",
        "my_cars": "My Cars",
        "add_car": "+ Add Car",
        "no_cars": "No cars added",
        "delete": "Delete",
        "edit": "Edit",
        "copy": "Copy",
        "car_name": "Car Name",
        "save": "Save",
        "cancel": "Cancel",
        "enter_name": "Enter name",
        "add_car_title": "Add Car",
        "edit_car_title": "Edit Car",
        "history_title": "Calculation History",
        "login_for_history": "Login to view history",
        "clear_all": "Clear All",
        "history_empty": "History is empty",
        "copied": "Copied!",
        "delete_confirm": "Delete?",
        "delete_car_confirm": "Delete this car?",
        "delete_history_confirm": "Delete entry?",
        "delete_history_msg": "Are you sure?",
        "clear_history_confirm": "Clear history?",
        "clear_history_msg": "Delete all history?",
        "delete_account_confirm": "Delete account?",
        "delete_account_msg": "Your account will be deactivated",
        "settings_title": "Settings",
        "currency_setting": "Currency",
        "language_setting": "Language",
        "error": "Error",
        "enter_numbers": "Enter valid numbers!",
        "distance_positive": "Distance must be greater than 0!",
        "distance_positive2": "Distance and consumption must be greater than 0!",
        "fuel_unit": "L",
        "km": "km",
        "about_title": "About",
        "version": "Version 1.0  •  2026",
        "footer": "© 2026 FullCalcPro",
        "database_error": "Database connection error",
        "old_password_wrong": "Wrong old password",
        "date": "Date",
        "password_changed": "Password successfully changed!",
        "total_distance": "Total distance",
        "total_cost": "Total cost",
        "clear_form_confirm": "Clear form?",
        "clear_form_msg": "Are you sure?",
        "login_required": "Login to save calculations",
        "lock_history": "Login to view history",
        "admin_users": "Users",
        "admin_stats": "Statistics",
        "admin_actions": "Actions",
        "admin_delete_photo": "Delete photo",
        "admin_view_cars": "Cars",
        "admin_view_history": "History",
        "admin_total_users": "Total users",
        "admin_total_cars": "Total cars",
        "admin_total_history": "Total records",
        "admin_active_users": "Active",
        "admin_blocked_users": "Blocked",
        "admin_deleted_users": "Deleted",
        "block_user": "Block",
        "unblock_user": "Unblock",
        "deleted_users_title": "Deleted Users",
        "about_intro": "FullCalcPro is a professional tool for accurate fuel consumption and trip cost tracking. The app is designed for drivers who want to control their expenses and track statistics for each car.",
        "about_features_title": "Main Features",
        "about_feature_1": "Two calculation modes — by actual fuel used or by average consumption. Choose the mode based on the data you have.",
        "about_feature_2": "Car garage — add unlimited vehicles with photos. Average consumption is automatically calculated for each car based on the entire trip history.",
        "about_feature_3": "Complete history — each calculation is saved to the database. View date, distance, fuel volume, price, consumption and total cost anytime. Filter history by car.",
        "about_feature_4": "Car statistics — total mileage, total fuel costs and average consumption over time. Clearly see which car is more economical.",
        "about_feature_5": "Multi-currency — supports ruble (₽), dollar ($) and euro (€). Switch with one click in settings.",
        "about_feature_6": "Bilingual interface — Russian and English. Switch without page reload.",
        "about_feature_7": "Data copying — copy any calculation or car statistics to clipboard with one click.",
        "about_security_title": "Security",
        "about_security_1": "Passwords are hashed using SHA-256 algorithm and never stored in plain text.",
        "about_security_2": "All data is stored in a reliable PostgreSQL database on your own server.",
        "about_security_3": "No data is shared with third parties — the app works completely locally.",
        "about_tech_title": "Technical Details",
        "about_tech_1": "Backend: Python + Flask",
        "about_tech_2": "Database: PostgreSQL",
        "about_tech_3": "Interface: modern glass design with animations",
        "about_tech_4": "Responsive layout — convenient to use on both computer and phone",
        "created_at": "Created",
        "active": "Active",
        "blocked": "Blocked",
        "back": "Back",
        "restore": "Restore",
        "cars": "Cars",
        "records": "Records",
        "no_users": "No registered users",
        "no_deleted_users": "No deleted users",
        "no_user_cars": "User has no cars",
        "empty_history": "Calculation history is empty",
        "registered": "Registered",
        "deleted_on": "Deleted",
        "view_cars": "Cars",
        "view_history": "History",
        "delete_photo": "Delete photo",
        "delete_car": "Delete car",
        "confirm_block_user": "Block user?",
        "confirm_block_user_msg": "User will not be able to log in.",
        "confirm_delete_photo": "Delete photo of this car?",
        "confirm_delete_car": "Delete car",
        "price_positive": "Price must be greater than 0",
        "fuel_positive": "Fuel must be greater than 0",
        "photo_optional": "Leave empty to keep current photo",
        "account_deactivated": "Your account has been deactivated.",
        "access_denied": "Access denied",
        "data_load_error": "Data loading error",
        "cannot_block_self": "Cannot block yourself",
        "restore_user_confirm": "Restore user?",
        "restore_error": "Restore error",
        "user_restored": "User restored",
        "invalid_params": "Invalid parameters",
        "deleted_users_info": "Users who have deleted their account. Data is preserved in the database.",
    }
}

CURRENCIES = {"₽ RUB": "₽", "$ USD": "$", "€ EUR": "€"}

# HTML TEMPLATES
TEMPLATES = {
    "base.html": '''<!DOCTYPE html>
<html lang="{{ lang }}">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ tr('app_title') }}</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
    <style>
        body {
            background-image: url('{{ url_for('static', filename='images/background.jpg') }}');
            background-size: cover;
            background-position: center;
            background-attachment: fixed;
            min-height: 100vh;
            font-family: 'Inter', sans-serif;
            font-size: 15px;
        }
        body::before {
            content: '';
            position: fixed;
            top: 0;
            left: 0;
            right: 0;
            bottom: 0;
            background: linear-gradient(135deg, rgba(20,30,48,0.85), rgba(36,59,85,0.75));
            z-index: -1;
        }
        * {
            transition: all 0.2s ease;
        }
        .navbar {
            background: rgba(255,255,255,0.15) !important;
            backdrop-filter: blur(16px);
            -webkit-backdrop-filter: blur(16px);
            border-bottom: 1px solid rgba(255,255,255,0.2);
            padding: 12px 0;
        }
        .navbar-brand {
            font-weight: 700;
            font-size: 1.5rem;
            color: white !important;
            text-decoration: none;
        }
        .logo-img {
            width: 36px;
            height: 36px;
            border-radius: 10px;
            object-fit: cover;
            margin-right: 10px;
        }
        .nav-link {
            font-weight: 500;
            padding: 8px 18px !important;
            color: rgba(255,255,255,0.85) !important;
            font-size: 0.9rem;
            border-radius: 12px;
        }
        .nav-link:hover {
            background: rgba(255,255,255,0.2);
            color: white !important;
        }
        .card, .modal-content, .list-group-item {
            background: rgba(255,255,255,0.15);
            backdrop-filter: blur(20px);
            -webkit-backdrop-filter: blur(20px);
            border: 1px solid rgba(255,255,255,0.25);
            border-radius: 20px;
            box-shadow: 0 8px 32px rgba(0,0,0,0.1);
            color: white;
        }
        .btn {
            border-radius: 12px;
            font-weight: 600;
            font-size: 0.85rem;
            padding: 8px 16px;
            white-space: nowrap;
        }
        .btn-primary {
            background: linear-gradient(135deg, #0071e3, #005bb5);
            border: none;
            color: white !important;
        }
        .btn-outline-secondary {
            border: 1px solid rgba(255,255,255,0.4);
            color: white;
            background: rgba(255,255,255,0.1);
        }
        .btn-outline-secondary:hover {
            background: rgba(255,255,255,0.25);
            color: white;
        }
        .btn-success {
            background: linear-gradient(135deg, #34c759, #28a745);
            border: none;
            color: white !important;
        }
        .btn-warning {
            background: linear-gradient(135deg, #ffcc00, #ff9500);
            border: none;
            color: #1a2a3a !important;
        }
        .btn-danger {
            background: linear-gradient(135deg, #ff3b30, #dc3545);
            border: none;
            color: white !important;
        }
        .btn-info {
            background: linear-gradient(135deg, #5ac8fa, #007aff);
            border: none;
            color: white !important;
        }
        .form-control, .form-select {
            background: rgba(255,255,255,0.15);
            color: white;
            border: 1px solid rgba(255,255,255,0.25);
            border-radius: 12px;
            padding: 10px 14px;
        }
        .form-control:focus, .form-select:focus {
            border-color: rgba(255,255,255,0.6);
            box-shadow: 0 0 0 3px rgba(0,113,227,0.3);
            background: rgba(255,255,255,0.2);
            color: white;
        }
        .form-label {
            color: white;
            font-weight: 600;
        }
        .form-select option {
            background: #1a2a3a;
            color: white;
        }
        .alert {
            border-radius: 16px;
            background: rgba(255,255,255,0.25);
            backdrop-filter: blur(12px);
            -webkit-backdrop-filter: blur(12px);
            color: white;
            border: 1px solid rgba(255,255,255,0.3);
        }
        .alert-success {
            background: rgba(52,199,89,0.3);
        }
        .alert-danger {
            background: rgba(255,59,48,0.3);
        }
        .alert-info {
            background: rgba(0,113,227,0.3);
        }
        h1, h2, h3, h4, h5, h6 {
            color: white;
            font-weight: 700;
        }
        .text-muted {
            color: rgba(255,255,255,0.7) !important;
        }
        .modal-content {
            background: rgba(30,40,50,0.95);
            backdrop-filter: blur(16px);
            -webkit-backdrop-filter: blur(16px);
            border-radius: 24px;
        }
        .modal-header {
            border-bottom: 1px solid rgba(255,255,255,0.15);
        }
        .modal-footer {
            border-top: 1px solid rgba(255,255,255,0.15);
        }
        .btn-close {
            filter: brightness(0) invert(1);
        }
        .badge {
            border-radius: 30px;
            padding: 6px 14px;
            font-weight: 600;
            font-size: 0.8rem;
        }
        .badge-success {
            background: linear-gradient(135deg, #34c759, #28a745);
            color: white;
        }
        .badge-danger {
            background: linear-gradient(135deg, #ff3b30, #dc3545);
            color: white;
        }
        .badge-warning {
            background: linear-gradient(135deg, #ffcc00, #ff9500);
            color: #1a2a3a;
        }
        .badge-info {
            background: linear-gradient(135deg, #0071e3, #005bb5);
            color: white;
        }
        footer {
            margin-top: 4rem;
            text-align: center;
            padding: 2rem;
            font-size: 0.85rem;
            color: rgba(255,255,255,0.6);
        }
        .card {
            animation: fadeInUp 0.4s ease-out;
        }
        @keyframes fadeInUp {
            from {
                opacity: 0;
                transform: translateY(20px);
            }
            to {
                opacity: 1;
                transform: translateY(0);
            }
        }
        a {
            color: rgba(255,255,255,0.9);
            text-decoration: none;
        }
        a:hover {
            color: white;
        }
        .stats-card {
            text-align: center;
            padding: 20px;
        }
        .stats-card .stats-number {
            font-size: 2.5rem;
            font-weight: 800;
        }
        .stats-card .stats-label {
            font-size: 0.9rem;
            opacity: 0.8;
        }
        .user-card {
            transition: all 0.3s ease;
        }
        .user-card:hover {
            transform: translateY(-5px);
            box-shadow: 0 12px 40px rgba(0,0,0,0.3);
        }
        .admin-header {
            background: rgba(255,255,255,0.15);
            backdrop-filter: blur(20px);
            -webkit-backdrop-filter: blur(20px);
            border: 1px solid rgba(255,255,255,0.25);
            border-radius: 20px;
            padding: 20px;
            margin-bottom: 20px;
        }
        .lock-overlay {
            background: rgba(255,255,255,0.15);
            backdrop-filter: blur(20px);
            -webkit-backdrop-filter: blur(20px);
            border: 1px solid rgba(255,255,255,0.25);
            border-radius: 20px;
            padding: 40px;
            text-align: center;
            box-shadow: 0 8px 32px rgba(0,0,0,0.1);
            color: white;
        }
        .car-photo-preview {
            max-width: 300px;
            max-height: 200px;
            border-radius: 16px;
            border: 2px solid rgba(255,255,255,0.3);
            object-fit: cover;
        }
        .mode-btn {
            flex: 1;
            text-align: center;
            padding: 12px;
            border-radius: 12px;
            border: 1px solid rgba(255,255,255,0.3);
            background: rgba(255,255,255,0.1);
            color: white;
            cursor: pointer;
            font-weight: 600;
        }
        .mode-btn.active {
            border-color: white;
            background: rgba(255,255,255,0.25);
        }
        .result-alert {
            font-size: 1.2rem;
            font-weight: 600;
            text-align: center;
        }
        .feature-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
            gap: 1.5rem;
            margin: 1.5rem 0;
        }
        .feature-card {
            background: rgba(255,255,255,0.1);
            border-radius: 16px;
            padding: 1.2rem;
            border: 1px solid rgba(255,255,255,0.15);
        }
        .feature-card i {
            font-size: 1.5rem;
            color: #0071e3;
            margin-bottom: 0.5rem;
        }
        .feature-card h6 {
            font-size: 1rem;
            margin-bottom: 0.4rem;
        }
        .feature-card p {
            font-size: 0.85rem;
            color: rgba(255,255,255,0.75);
            margin: 0;
        }
        .nav-tabs .nav-link {
            color: rgba(255,255,255,0.7) !important;
            font-weight: 600;
            border: none;
            padding: 10px 20px;
            border-radius: 12px;
            margin-right: 5px;
            background: transparent;
        }
        .nav-tabs .nav-link.active {
            color: white !important;
            background: rgba(255,255,255,0.2) !important;
            border: none;
        }
        .nav-tabs .nav-link:hover {
            color: white !important;
            background: rgba(255,255,255,0.15) !important;
        }
        .nav-tabs {
            border-bottom: 1px solid rgba(255,255,255,0.2) !important;
        }
    </style>
</head>
<body>
    <nav class="navbar navbar-expand-lg sticky-top">
        <div class="container">
            <a class="navbar-brand" href="/">
                <img src="{{ url_for('static', filename='logo.png') }}" class="logo-img" alt="Logo" onerror="this.style.display='none'">
                {{ tr('app_title') }}
            </a>
            <button class="navbar-toggler" data-bs-toggle="collapse" data-bs-target="#navbarNav" 
                    style="border:1px solid rgba(255,255,255,0.3); background:rgba(255,255,255,0.1);">
                <span class="navbar-toggler-icon" style="filter:brightness(0) invert(1)"></span>
            </button>
            <div class="collapse navbar-collapse" id="navbarNav">
                <ul class="navbar-nav ms-auto">
                    {% if user %}
                        {% if is_admin %}
                            <li class="nav-item"><a class="nav-link" href="/admin">{{ tr('admin_panel') }}</a></li>
                        {% else %}
                            <li class="nav-item"><a class="nav-link" href="/">{{ tr('calculator') }}</a></li>
                            <li class="nav-item"><a class="nav-link" href="/profile">{{ tr('profile') }}</a></li>
                            <li class="nav-item"><a class="nav-link" href="/settings">{{ tr('settings') }}</a></li>
                        {% endif %}
                    {% else %}
                        <li class="nav-item"><a class="nav-link" href="/auth">{{ tr('login_register') }}</a></li>
                        <li class="nav-item"><a class="nav-link" href="/settings">{{ tr('settings') }}</a></li>
                    {% endif %}
                    {% if not is_admin %}
                        <li class="nav-item"><a class="nav-link" href="/about">{{ tr('about') }}</a></li>
                    {% endif %}
                </ul>
            </div>
        </div>
    </nav>

    <div class="container mt-4">
        {% with messages = get_flashed_messages(with_categories=true) %}
            {% if messages %}
                {% for cat, msg in messages %}
                    <div class="alert alert-{{ cat }} alert-dismissible fade show">
                        {{ msg }}
                        <button class="btn-close" data-bs-dismiss="alert"></button>
                    </div>
                {% endfor %}
            {% endif %}
        {% endwith %}
        {% block content %}{% endblock %}
    </div>

    <div class="modal fade" id="confirmModal">
        <div class="modal-dialog modal-dialog-centered">
            <div class="modal-content">
                <div class="modal-header">
                    <h5 class="modal-title fw-bold" id="confirmModalTitle"></h5>
                    <button class="btn-close" data-bs-dismiss="modal"></button>
                </div>
                <div class="modal-body" id="confirmModalBody" style="color:white"></div>
                <div class="modal-footer">
                    <button class="btn btn-outline-secondary" data-bs-dismiss="modal">
                        {% if lang == 'ru' %}Отмена{% else %}Cancel{% endif %}
                    </button>
                    <button class="btn btn-danger" id="confirmModalBtn"></button>
                </div>
            </div>
        </div>
    </div>

    <div class="modal fade" id="userCarsModal">
        <div class="modal-dialog modal-lg">
            <div class="modal-content">
                <div class="modal-header">
                    <h5 class="modal-title fw-bold">
                        {% if lang == 'ru' %}Автомобили пользователя{% else %}User Cars{% endif %}
                    </h5>
                    <button class="btn-close" data-bs-dismiss="modal"></button>
                </div>
                <div class="modal-body" id="userCarsContent" style="color:white"></div>
            </div>
        </div>
    </div>

    <div class="modal fade" id="userHistoryModal">
        <div class="modal-dialog modal-lg">
            <div class="modal-content">
                <div class="modal-header">
                    <h5 class="modal-title fw-bold">
                        {% if lang == 'ru' %}История расчётов{% else %}Calculation History{% endif %}
                    </h5>
                    <button class="btn-close" data-bs-dismiss="modal"></button>
                </div>
                <div class="modal-body" id="userHistoryContent" style="color:white; max-height:500px; overflow-y:auto;"></div>
            </div>
        </div>
    </div>

    <footer>{{ tr('footer') }}</footer>

    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
    <script>
        function togglePassword(b) {
            let i = b.parentElement.querySelector('input'),
                ic = b.querySelector('i');
            i.type = i.type === 'password' ? 'text' : 'password';
            ic.classList.toggle('fa-eye');
            ic.classList.toggle('fa-eye-slash');
        }

        let confirmCallback = null;
        const confirmModal = new bootstrap.Modal(document.getElementById('confirmModal'));

        function showConfirm(t, m, b, c) {
            document.getElementById('confirmModalTitle').textContent = t;
            document.getElementById('confirmModalBody').textContent = m;
            document.getElementById('confirmModalBtn').textContent = b;
            confirmCallback = c;
            confirmModal.show();
        }

        document.getElementById('confirmModalBtn').addEventListener('click', () => {
            if (confirmCallback) confirmCallback();
            confirmModal.hide();
            confirmCallback = null;
        });
    </script>
    {% block scripts %}{% endblock %}
</body>
</html>''',

    "index.html": '''{% extends "base.html" %}
{% block content %}
<div style="max-width:540px; margin:0 auto">
    <h2 class="mb-4 text-center">{{ tr('calculator') }}</h2>
    {% if not user %}
        <div class="alert alert-info mb-4 text-center">
            {{ tr('login_required') }} 
            <a href="/auth" style="color:white; text-decoration:underline">{{ tr('login_register') }}</a>
        </div>
    {% endif %}
    <div class="card p-4">
        <div class="mb-3">
            <label class="form-label fw-semibold">{{ tr('car') }}</label>
            <select id="carSelect" class="form-select">
                <option value="">{{ tr('no_car') }}</option>
                {% if user %}
                    {% for car in cars %}
                        <option value="{{ car.name }}" 
                                data-photo="{% if car.photo %}{{ url_for('static', filename='uploads/' + car.photo) }}{% endif %}" 
                                data-avg="{{ car.avg_consumption or '' }}">
                            {{ car.name }}
                        </option>
                    {% endfor %}
                {% endif %}
            </select>
        </div>
        <div class="text-center mb-3">
            <img id="carPhotoPreview" src="" class="car-photo-preview" style="display:none">
        </div>
        <form id="calcForm">
            <div class="mode-selector" style="display:flex; gap:12px; margin-bottom:20px">
                <button type="button" class="mode-btn active" data-mode="consumption">{{ tr('calc_mode_consumption') }}</button>
                <button type="button" class="mode-btn" data-mode="cost">{{ tr('calc_mode_cost') }}</button>
            </div>
            <input type="hidden" name="mode" value="consumption" id="modeInput">
            <div class="mb-3">
                <label class="form-label fw-semibold">{{ tr('distance_label') }}, {{ tr('km') }}</label>
                <input type="number" step="any" name="distance" id="distance" class="form-control" required>
            </div>
            <div id="fuelField" class="mb-3">
                <label class="form-label fw-semibold">{{ tr('fuel_used') }}, {{ tr('fuel_unit') }}</label>
                <input type="number" step="any" name="fuel" id="fuel" class="form-control">
            </div>
            <div id="avgField" class="mb-3" style="display:none">
                <label class="form-label fw-semibold">{{ tr('avg_consumption_label') }}, {{ tr('fuel_unit') }}/100{{ tr('km') }}</label>
                <input type="number" step="any" name="avg_consumption" id="avgConsumption" class="form-control">
            </div>
            <div class="mb-3">
                <label class="form-label fw-semibold">{{ tr('fuel_price') }}, {{ currency_symbol }}/{{ tr('fuel_unit') }}</label>
                <input type="number" step="any" name="price" id="price" class="form-control" required>
            </div>
            <div class="d-flex justify-content-center gap-2">
                <button type="submit" class="btn btn-primary px-5" id="calculateBtn">{{ tr('calculate') }}</button>
                <button type="button" id="clearBtn" class="btn btn-outline-secondary px-4">{{ tr('clear') }}</button>
            </div>
        </form>
        <div id="result" class="mt-3 alert result-alert" style="display:none"></div>
    </div>
    <div class="mt-5">
        <h4 class="fw-bold mb-3 text-center">{{ tr('history_title') }}</h4>
        {% if user %}
            <div id="historyContainer">{% include 'history_list.html' %}</div>
        {% else %}
            <div class="lock-overlay">
                <i class="fas fa-lock fa-3x mb-3" style="color:white"></i>
                <p class="mb-3" style="color:white">{{ tr('lock_history') }}</p>
                <a href="/auth" class="btn btn-primary">{{ tr('login_register') }}</a>
            </div>
        {% endif %}
    </div>
</div>
{% endblock %}

{% block scripts %}
<script>
    let isLoggedIn = {{ 'true' if user else 'false' }},
        isCalc = false,
        currentCurrency = '{{ currency_symbol }}';

    document.querySelectorAll('.mode-btn').forEach(b => {
        b.addEventListener('click', function() {
            document.querySelectorAll('.mode-btn').forEach(x => x.classList.remove('active'));
            this.classList.add('active');
            let m = this.dataset.mode;
            document.getElementById('modeInput').value = m;
            document.getElementById('fuelField').style.display = m === 'consumption' ? 'block' : 'none';
            document.getElementById('avgField').style.display = m === 'consumption' ? 'none' : 'block';
            if (m === 'cost') {
                let o = document.getElementById('carSelect'),
                    v = o.options[o.selectedIndex].getAttribute('data-avg');
                if (v) document.getElementById('avgConsumption').value = v;
            }
        });
    });

    document.getElementById('carSelect').addEventListener('change', function() {
        let o = this.options[this.selectedIndex],
            u = o.getAttribute('data-photo'),
            img = document.getElementById('carPhotoPreview');
        img.style.display = u && u !== 'None' ? 'block' : 'none';
        if (u && u !== 'None') img.src = u;
        if (document.getElementById('modeInput').value === 'cost') {
            let v = o.getAttribute('data-avg');
            if (v) document.getElementById('avgConsumption').value = v;
        }
        if (isLoggedIn) loadHistory(this.value);
    });

    document.getElementById('calcForm').addEventListener('submit', async function(e) {
        e.preventDefault();
        if (isCalc) return;
        let btn = document.getElementById('calculateBtn');
        isCalc = true;
        btn.disabled = true;
        btn.innerHTML = '<span class="spinner-border spinner-border-sm me-1"></span>';
        let fd = new FormData(this);
        fd.append('car', isLoggedIn ? document.getElementById('carSelect').value : '{{ tr("no_car") }}');
        try {
            let r = await fetch('/calculate', { method: 'POST', body: fd });
            let d = await r.json(),
                div = document.getElementById('result');
            if (d.error) {
                div.className = 'mt-3 alert alert-danger result-alert';
                div.innerText = d.error;
            } else {
                div.className = 'mt-3 alert alert-success result-alert';
                div.innerText = d.mode === 'consumption' 
                    ? `{{ tr('fuel_consumption') }}: ${d.consumption} {{ tr('fuel_unit') }}/100{{ tr('km') }}, {{ tr('cost') }}: ${d.cost} ${d.currency}`
                    : `{{ tr('trip_cost') }}: ${d.cost} ${d.currency}`;
            }
            div.style.display = 'block';
            if (isLoggedIn) loadHistory(document.getElementById('carSelect').value);
        } catch(err) {
            console.error(err);
        } finally {
            isCalc = false;
            btn.disabled = false;
            btn.innerHTML = '{{ tr("calculate") }}';
        }
    });

    document.getElementById('clearBtn').addEventListener('click', () => {
        showConfirm(
            '{{ tr("clear_form_confirm") }}',
            '{{ tr("clear_form_msg") }}',
            '{{ tr("clear") }}',
            () => {
                document.getElementById('calcForm').reset();
                document.getElementById('result').style.display = 'none';
                document.getElementById('fuelField').style.display = 'block';
                document.getElementById('avgField').style.display = 'none';
                document.querySelectorAll('.mode-btn').forEach(b => b.classList.remove('active'));
                document.querySelector('.mode-btn[data-mode="consumption"]').classList.add('active');
                document.getElementById('modeInput').value = 'consumption';
            }
        );
    });

    {% if user %}
        async function loadHistory(car = '') {
            try {
                let r = await fetch(`/history_html?car=${encodeURIComponent(car)}`);
                document.getElementById('historyContainer').innerHTML = await r.text();
            } catch(e) {}
        }

        function clearAllHistory() {
            showConfirm(
                '{{ tr("clear_history_confirm") }}',
                '{{ tr("clear_history_msg") }}',
                '{{ tr("clear_all") }}',
                async () => {
                    await fetch('/clear_history', { method: 'POST' });
                    loadHistory(document.getElementById('carSelect').value);
                }
            );
        }

        function deleteHistoryEntry(id) {
            showConfirm(
                '{{ tr("delete_history_confirm") }}',
                '{{ tr("delete_history_msg") }}',
                '{{ tr("delete") }}',
                async () => {
                    await fetch('/delete_history/' + id, { method: 'POST' });
                    loadHistory(document.getElementById('carSelect').value);
                }
            );
        }

        function copyEntry(btn) {
            navigator.clipboard.writeText(btn.getAttribute('data-text')).then(() => {
                let o = btn.innerHTML;
                btn.innerHTML = '<i class="fas fa-check"></i>';
                setTimeout(() => btn.innerHTML = o, 1500);
            }).catch(() => alert('{{ tr("error") }}'));
        }
    {% endif %}
</script>
{% endblock %}''',

    "history_list.html": '''{% if user %}
        <div class="d-flex justify-content-end mb-3">
            <button class="btn btn-outline-danger" onclick="clearAllHistory()">{{ tr('clear_all') }}</button>
        </div>
        {% if history %}
            <div class="list-group" id="historyList">
                {% for e in history %}
                    <div class="list-group-item d-flex align-items-center p-3">
                        {% if e.photo %}
                            <img src="{{ url_for('static', filename='uploads/' + e.photo) }}" 
                                 style="width:56px; height:56px; object-fit:cover; border-radius:14px; border:2px solid rgba(255,255,255,0.3); margin-right:15px">
                        {% else %}
                            <div style="width:56px; height:56px; background:rgba(255,255,255,0.1); border-radius:14px; margin-right:15px; display:flex; align-items:center; justify-content:center">
                                <i class="fas fa-car" style="color:rgba(255,255,255,0.7)"></i>
                            </div>
                        {% endif %}
                        <div class="flex-grow-1">
                            <div class="fw-semibold mb-1">{{ e.date }} – {{ e.car or tr('no_car') }}</div>
                            <div class="small" style="color:rgba(255,255,255,0.7)">
                                <span class="me-3"><i class="fas fa-road me-1"></i>{{ e.distance }} {{ tr('km') }}</span>
                                <span class="me-3"><i class="fas fa-gas-pump me-1"></i>{{ e.fuel }} {{ tr('fuel_unit') }}</span>
                                <span class="me-3"><i class="fas fa-tachometer-alt me-1"></i>{{ e.consumption }} {{ tr('fuel_unit') }}/100{{ tr('km') }}</span>
                                <span><i class="fas fa-coins me-1"></i>{{ "%.2f"|format(e.cost) }} {{ currency_symbol }}</span>
                            </div>
                        </div>
                        <div class="ms-auto d-flex flex-column align-items-end">
                            <button class="btn btn-sm btn-outline-secondary mb-1" 
                                    onclick="copyEntry(this)" 
                                    data-text="{{ tr('date') }}: {{ e.date }}, {{ tr('car') }}: {{ e.car or tr('no_car') }}, {{ tr('fuel_consumption') }}: {{ e.consumption }} {{ tr('fuel_unit') }}/100{{ tr('km') }}, {{ tr('cost') }}: {{ '%.2f'|format(e.cost) }} {{ currency_symbol }}">
                                <i class="fas fa-copy"></i>
                            </button>
                            <button class="btn btn-sm btn-outline-danger" onclick="deleteHistoryEntry({{ e.id }})">
                                <i class="fas fa-trash"></i>
                            </button>
                        </div>
                    </div>
                {% endfor %}
            </div>
        {% else %}
            <div class="card p-4 text-center">{{ tr('history_empty') }}</div>
        {% endif %}
    {% else %}
        <div class="card p-4 text-center">{{ tr('login_for_history') }}</div>
    {% endif %}''',

    "auth.html": '''{% extends "base.html" %}
{% block content %}
<div class="row justify-content-center">
    <div class="col-md-5">
        <div class="card p-4">
            <h2 class="mb-4 text-center fw-bold">{{ tr('login_register') }}</h2>
            <ul class="nav nav-tabs mb-3" role="tablist">
                <li class="nav-item" role="presentation">
                    <button class="nav-link active" data-bs-toggle="tab" data-bs-target="#login-pane" type="button" role="tab">
                        {{ tr('login') }}
                    </button>
                </li>
                <li class="nav-item" role="presentation">
                    <button class="nav-link" data-bs-toggle="tab" data-bs-target="#register-pane" type="button" role="tab">
                        {{ tr('register') }}
                    </button>
                </li>
            </ul>
            <div class="tab-content">
                <div class="tab-pane fade show active" id="login-pane" role="tabpanel">
                    <form method="post" action="/auth">
                        <input type="hidden" name="form_type" value="login">
                        <div class="mb-3">
                            <label class="form-label fw-semibold">{{ tr('email') }}</label>
                            <input type="email" name="email" class="form-control" required>
                        </div>
                        <div class="mb-3">
                            <label class="form-label fw-semibold">{{ tr('password') }}</label>
                            <div class="input-group">
                                <input type="password" name="password" class="form-control" required>
                                <button class="btn btn-outline-secondary" type="button" onclick="togglePassword(this)" style="border-radius:0 12px 12px 0">
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
                            <label class="form-label fw-semibold">{{ tr('email') }}</label>
                            <input type="email" name="email" class="form-control" required>
                        </div>
                        <div class="mb-3">
                            <label class="form-label fw-semibold">{{ tr('password') }}</label>
                            <div class="input-group">
                                <input type="password" name="password" class="form-control" required>
                                <button class="btn btn-outline-secondary" type="button" onclick="togglePassword(this)" style="border-radius:0 12px 12px 0">
                                    <i class="fas fa-eye"></i>
                                </button>
                            </div>
                        </div>
                        <div class="mb-3">
                            <label class="form-label fw-semibold">{{ tr('repeat_password') }}</label>
                            <div class="input-group">
                                <input type="password" name="password2" class="form-control" required>
                                <button class="btn btn-outline-secondary" type="button" onclick="togglePassword(this)" style="border-radius:0 12px 12px 0">
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

    "profile.html": '''{% extends "base.html" %}
{% block content %}
<div class="d-flex justify-content-between align-items-center mb-4 flex-wrap gap-2">
    <h2 class="fw-bold">{{ tr('profile') }}</h2>
    <div class="d-flex gap-2">
        <button class="btn btn-outline-secondary" onclick="openChangePasswordModal()">{{ tr('change_password') }}</button>
        <a href="/logout" class="btn btn-outline-secondary">{{ tr('logout') }}</a>
        <form method="post" action="/delete_account" onsubmit="return confirm('{{ tr('delete_account_msg') }}')" class="m-0">
            <button class="btn btn-outline-danger">{{ tr('delete_account') }}</button>
        </form>
    </div>
</div>
<div class="card p-4 mb-4">
    <p class="mb-0"><i class="fas fa-envelope me-2"></i><strong>{{ user }}</strong></p>
</div>
<div class="card p-4" id="carsSection">{% include 'profile_cars.html' %}</div>

<div class="modal fade" id="editCarModal">
    <div class="modal-dialog modal-dialog-centered">
        <div class="modal-content">
            <div class="modal-header border-0">
                <h5 class="modal-title fw-bold">{{ tr('edit_car_title') }}</h5>
                <button class="btn-close" data-bs-dismiss="modal"></button>
            </div>
            <div class="modal-body">
                <form id="editCarForm" onsubmit="submitEditCar(event)" enctype="multipart/form-data">
                    <input type="hidden" name="old_name" id="editOldName">
                    <div class="mb-3">
                        <label class="form-label fw-semibold">{{ tr('car_name') }}</label>
                        <input type="text" name="new_name" id="editNewName" class="form-control" required>
                    </div>
                    <div class="mb-3">
                        <label class="form-label fw-semibold">{{ tr('photo_optional') }}</label>
                        <input type="file" name="photo" class="form-control" accept="image/*">
                        <small class="text-muted">{{ tr('photo_optional') }}</small>
                    </div>
                    <button type="submit" class="btn btn-primary w-100">{{ tr('save') }}</button>
                </form>
            </div>
        </div>
    </div>
</div>

<div class="modal fade" id="changePasswordModal">
    <div class="modal-dialog modal-dialog-centered">
        <div class="modal-content">
            <div class="modal-header border-0">
                <h5 class="modal-title fw-bold">{{ tr('change_password') }}</h5>
                <button class="btn-close" data-bs-dismiss="modal"></button>
            </div>
            <div class="modal-body">
                <form id="changePasswordForm" onsubmit="submitChangePassword(event)">
                    <div class="mb-3">
                        <label class="form-label fw-semibold">{{ tr('old_password') }}</label>
                        <div class="input-group">
                            <input type="password" name="old_password" class="form-control" required>
                            <button class="btn btn-outline-secondary" type="button" onclick="togglePassword(this)" style="border-radius:0 12px 12px 0">
                                <i class="fas fa-eye"></i>
                            </button>
                        </div>
                    </div>
                    <div class="mb-3">
                        <label class="form-label fw-semibold">{{ tr('new_password') }}</label>
                        <div class="input-group">
                            <input type="password" name="new_password" class="form-control" required>
                            <button class="btn btn-outline-secondary" type="button" onclick="togglePassword(this)" style="border-radius:0 12px 12px 0">
                                <i class="fas fa-eye"></i>
                            </button>
                        </div>
                    </div>
                    <div class="mb-3">
                        <label class="form-label fw-semibold">{{ tr('repeat_new_password') }}</label>
                        <div class="input-group">
                            <input type="password" name="new_password2" class="form-control" required>
                            <button class="btn btn-outline-secondary" type="button" onclick="togglePassword(this)" style="border-radius:0 12px 12px 0">
                                <i class="fas fa-eye"></i>
                            </button>
                        </div>
                    </div>
                    <button type="submit" class="btn btn-primary w-100 mt-2">{{ tr('change_password_btn') }}</button>
                </form>
            </div>
        </div>
    </div>
</div>
{% endblock %}

{% block scripts %}
<script>
    let editCarModal, passwordModal;
    document.addEventListener('DOMContentLoaded', () => {
        editCarModal = new bootstrap.Modal(document.getElementById('editCarModal'));
        passwordModal = new bootstrap.Modal(document.getElementById('changePasswordModal'));
    });

    function openChangePasswordModal() { passwordModal.show(); }

    async function submitChangePassword(e) {
        e.preventDefault();
        let fd = new FormData(e.target);
        try {
            let r = await fetch('/change_password', { method: 'POST', body: fd });
            let d = await r.json();
            alert(d.message);
            if (d.success) {
                e.target.reset();
                passwordModal.hide();
            }
        } catch(e) {
            alert('{{ tr("error") }}');
        }
    }

    async function addCar(e) {
        e.preventDefault();
        try {
            let r = await fetch('/add_car', { method: 'POST', body: new FormData(e.target) });
            document.getElementById('carsSection').innerHTML = await r.text();
        } catch(e) {
            alert('{{ tr("error") }}');
        }
    }

    function editCar(name) {
        document.getElementById('editOldName').value = name;
        document.getElementById('editNewName').value = name;
        editCarModal.show();
    }

    async function submitEditCar(e) {
        e.preventDefault();
        try {
            let r = await fetch('/edit_car', { method: 'POST', body: new FormData(e.target) });
            document.getElementById('carsSection').innerHTML = await r.text();
            editCarModal.hide();
        } catch(e) {
            alert('{{ tr("error") }}');
        }
    }

    function copyCarData(name, distance, cost, avg) {
        let text = `{{ tr('car') }}: ${name}\n{{ tr('total_distance') }}: ${distance} {{ tr('km') }}\n{{ tr('total_cost') }}: ${cost} {{ currency_symbol }}\n{{ tr('avg_consumption') }}: ${avg} {{ tr('fuel_unit') }}/100{{ tr('km') }}`;
        navigator.clipboard.writeText(text).then(() => alert('{{ tr("copied") }}')).catch(() => alert('{{ tr("error") }}'));
    }

    function deleteCar(name) {
        showConfirm(
            '{{ tr("delete_car_confirm") }}',
            '{{ tr("delete_car_confirm") }}',
            '{{ tr("delete") }}',
            async () => {
                try {
                    let r = await fetch('/delete_car/' + name, { method: 'POST' });
                    document.getElementById('carsSection').innerHTML = await r.text();
                } catch(e) {
                    alert('{{ tr("error") }}');
                }
            }
        );
    }
</script>
{% endblock %}''',

    "profile_cars.html": '''<h5 class="fw-bold mb-3">{{ tr('add_car_title') }}</h5>
<form onsubmit="addCar(event)" enctype="multipart/form-data" class="row g-3 mb-4">
    <div class="col-md-5">
        <input type="text" name="name" class="form-control" placeholder="{{ tr('car_name') }}" required>
    </div>
    <div class="col-md-5">
        <input type="file" name="photo" class="form-control" accept="image/*">
    </div>
    <div class="col-md-2">
        <button type="submit" class="btn btn-primary w-100">{{ tr('save') }}</button>
    </div>
</form>
<h5 class="fw-bold">{{ tr('my_cars') }}</h5>
<div class="row">
    {% for car in cars %}
        <div class="col-md-4 mb-4 d-flex">
            <div class="card p-3 w-100" style="border-radius:20px">
                {% if car.photo %}
                    <img src="{{ url_for('static', filename='uploads/' + car.photo) }}" 
                         style="height:160px; width:100%; object-fit:cover; border-radius:16px; margin-bottom:1rem;">
                {% else %}
                    <div class="d-flex align-items-center justify-content-center w-100 mb-3" 
                         style="height:160px; background:rgba(255,255,255,0.1); border-radius:16px;">
                        <i class="fas fa-car fa-4x" style="color:rgba(255,255,255,0.5)"></i>
                    </div>
                {% endif %}
                <div class="text-center d-flex flex-column flex-grow-1">
                    <h6 class="fw-bold">{{ car.name }}</h6>
                    {% if car.avg_consumption %}
                        <span class="badge badge-info mb-2">{{ "%.1f"|format(car.avg_consumption) }} {{ tr('fuel_unit') }}/100{{ tr('km') }}</span>
                    {% endif %}
                    <div class="mt-auto">
                        <div class="mt-2 small" style="color:rgba(255,255,255,0.7)">
                            {% set s = car_stats.get(car.name, {}) %}
                            <div>{{ tr('total_distance') }}: {{ s.total_distance }} {{ tr('km') }}</div>
                            <div>{{ tr('total_cost') }}: {{ s.total_cost }} {{ currency_symbol }}</div>
                            <div>{{ tr('avg_consumption') }}: {{ s.avg_consumption }} {{ tr('fuel_unit') }}/100{{ tr('km') }}</div>
                        </div>
                        <div class="mt-3 car-actions">
                            <button class="btn btn-sm btn-outline-secondary mb-1 w-100" onclick="editCar('{{ car.name }}')">
                                {{ tr('edit') }}
                            </button>
                            <button class="btn btn-sm btn-outline-secondary mb-1 w-100" 
                                    onclick="copyCarData('{{ car.name }}','{{ s.total_distance }}','{{ s.total_cost }}','{{ s.avg_consumption }}')">
                                {{ tr('copy') }}
                            </button>
                            <button class="btn btn-sm btn-outline-danger w-100" onclick="deleteCar('{{ car.name }}')">
                                {{ tr('delete') }}
                            </button>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    {% else %}
        <div class="col-12">
            <p class="text-muted">{{ tr('no_cars') }}</p>
        </div>
    {% endfor %}
</div>''',

    "settings.html": '''{% extends "base.html" %}
{% block content %}
<h2 class="mb-4 fw-bold">{{ tr('settings_title') }}</h2>
<div class="card p-4">
    <form id="settingsForm" onsubmit="submitSettings(event)">
        <div class="mb-4">
            <label class="form-label fw-bold">{{ tr('language_setting') }}</label>
            <select name="language" class="form-select" style="max-width:300px">
                <option value="ru" {% if settings.language=='ru' %}selected{% endif %}>Русский</option>
                <option value="en" {% if settings.language=='en' %}selected{% endif %}>English</option>
            </select>
        </div>
        <div class="mb-4">
            <label class="form-label fw-bold">{{ tr('currency_setting') }}</label>
            <select name="currency" class="form-select" style="max-width:200px">
                {% for k,s in currencies.items() %}
                    <option value="{{ k }}" {% if settings.currency==k %}selected{% endif %}>{{ s }}</option>
                {% endfor %}
            </select>
        </div>
        <button type="submit" class="btn btn-primary">{{ tr('save') }}</button>
    </form>
</div>
{% endblock %}

{% block scripts %}
<script>
    async function submitSettings(e) {
        e.preventDefault();
        try {
            let r = await fetch('/settings', { method: 'POST', body: new FormData(e.target) });
            if (r.ok) location.reload();
        } catch(e) {}
    }
</script>
{% endblock %}''',

    "about.html": '''{% extends "base.html" %}
{% block content %}
<h2 class="mb-4 fw-bold text-center">{{ tr('about_title') }}</h2>
<div class="card p-4 mb-4 text-center">
    <p class="mb-0" style="font-size:1.1rem; line-height:1.7">{{ tr('about_intro') }}</p>
</div>
<div class="card p-4 mb-4">
    <h4 class="fw-bold mb-3">{{ tr('about_features_title') }}</h4>
    <div class="feature-grid">
        <div class="feature-card">
            <i class="fas fa-exchange-alt"></i>
            <h6>{{ tr('calc_mode_consumption') }} / {{ tr('calc_mode_cost') }}</h6>
            <p>{{ tr('about_feature_1') }}</p>
        </div>
        <div class="feature-card">
            <i class="fas fa-car"></i>
            <h6>{{ tr('my_cars') }}</h6>
            <p>{{ tr('about_feature_2') }}</p>
        </div>
        <div class="feature-card">
            <i class="fas fa-history"></i>
            <h6>{{ tr('history') }}</h6>
            <p>{{ tr('about_feature_3') }}</p>
        </div>
        <div class="feature-card">
            <i class="fas fa-chart-bar"></i>
            <h6>{{ tr('total_distance') }} / {{ tr('total_cost') }}</h6>
            <p>{{ tr('about_feature_4') }}</p>
        </div>
        <div class="feature-card">
            <i class="fas fa-coins"></i>
            <h6>{{ tr('currency_setting') }}</h6>
            <p>{{ tr('about_feature_5') }}</p>
        </div>
        <div class="feature-card">
            <i class="fas fa-globe"></i>
            <h6>{{ tr('language_setting') }}</h6>
            <p>{{ tr('about_feature_6') }}</p>
        </div>
        <div class="feature-card">
            <i class="fas fa-copy"></i>
            <h6>{{ tr('copy') }}</h6>
            <p>{{ tr('about_feature_7') }}</p>
        </div>
    </div>
</div>
<div class="row">
    <div class="col-md-6 mb-4">
        <div class="card p-4 h-100">
            <h4 class="fw-bold mb-3">
                <i class="fas fa-shield-alt me-2"></i>{{ tr('about_security_title') }}
            </h4>
            <ul style="list-style:none; padding:0">
                <li class="mb-2" style="color:rgba(255,255,255,0.85)">
                    <i class="fas fa-check-circle me-2" style="color:#34c759"></i>{{ tr('about_security_1') }}
                </li>
                <li class="mb-2" style="color:rgba(255,255,255,0.85)">
                    <i class="fas fa-check-circle me-2" style="color:#34c759"></i>{{ tr('about_security_2') }}
                </li>
                <li style="color:rgba(255,255,255,0.85)">
                    <i class="fas fa-check-circle me-2" style="color:#34c759"></i>{{ tr('about_security_3') }}
                </li>
            </ul>
        </div>
    </div>
    <div class="col-md-6 mb-4">
        <div class="card p-4 h-100">
            <h4 class="fw-bold mb-3">
                <i class="fas fa-cogs me-2"></i>{{ tr('about_tech_title') }}
            </h4>
            <ul style="list-style:none; padding:0">
                <li class="mb-2" style="color:rgba(255,255,255,0.85)">
                    <i class="fas fa-server me-2" style="color:#0071e3"></i>{{ tr('about_tech_1') }}
                </li>
                <li class="mb-2" style="color:rgba(255,255,255,0.85)">
                    <i class="fas fa-database me-2" style="color:#0071e3"></i>{{ tr('about_tech_2') }}
                </li>
                <li class="mb-2" style="color:rgba(255,255,255,0.85)">
                    <i class="fas fa-paint-brush me-2" style="color:#0071e3"></i>{{ tr('about_tech_3') }}
                </li>
                <li style="color:rgba(255,255,255,0.85)">
                    <i class="fas fa-mobile-alt me-2" style="color:#0071e3"></i>{{ tr('about_tech_4') }}
                </li>
            </ul>
        </div>
    </div>
</div>
<hr style="border-color:rgba(255,255,255,0.1)">
<p class="text-muted text-center">{{ tr('version') }}</p>
{% endblock %}''',

    "admin.html": '''{% extends "base.html" %}
{% block content %}
<div class="admin-header d-flex justify-content-between align-items-center flex-wrap gap-2">
    <div>
        <h4 class="mb-0"><i class="fas fa-user-shield me-2"></i>{{ user }}</h4>
        <span class="badge badge-warning">{% if lang == 'ru' %}Администратор{% else %}Administrator{% endif %}</span>
    </div>
    <div class="d-flex gap-2">
        <a href="/admin/deleted_users" class="btn btn-outline-secondary">
            <i class="fas fa-archive me-1"></i>{{ tr('deleted_users_title') }}
        </a>
        <button class="btn btn-outline-secondary" onclick="openChangePasswordModal()">
            <i class="fas fa-key me-1"></i>{{ tr('change_password') }}
        </button>
        <a href="/logout" class="btn btn-outline-secondary">
            <i class="fas fa-sign-out-alt me-1"></i>{{ tr('logout') }}
        </a>
    </div>
</div>

<h2 class="mb-3 fw-bold">{{ tr('admin_panel') }}</h2>

<div class="row mb-4">
    <div class="col-md-3 mb-3">
        <div class="card stats-card">
            <div class="stats-number text-primary">{{ stats.total_users }}</div>
            <div class="stats-label">{{ tr('admin_total_users') }}</div>
        </div>
    </div>
    <div class="col-md-3 mb-3">
        <div class="card stats-card">
            <div class="stats-number text-success">{{ stats.active_users }}</div>
            <div class="stats-label">{{ tr('admin_active_users') }}</div>
        </div>
    </div>
    <div class="col-md-3 mb-3">
        <div class="card stats-card">
            <div class="stats-number text-danger">{{ stats.blocked_users }}</div>
            <div class="stats-label">{{ tr('admin_blocked_users') }}</div>
        </div>
    </div>
    <div class="col-md-3 mb-3">
        <div class="card stats-card">
            <div class="stats-number text-info">{{ stats.total_history }}</div>
            <div class="stats-label">{{ tr('admin_total_history') }}</div>
        </div>
    </div>
</div>

<h4 class="fw-bold mb-3">{{ tr('admin_users') }}</h4>
<div class="row">
    {% for user_item in users %}
        <div class="col-md-6 col-lg-4 mb-4">
            <div class="card p-4 user-card h-100">
                <div class="d-flex justify-content-between align-items-start mb-3">
                    <div style="flex:1; min-width:0">
                        <h5 class="mb-1 text-truncate">{{ user_item.email }}</h5>
                        <small class="text-muted">{{ tr('created_at') }}: {{ user_item.created_at }}</small>
                    </div>
                    <div style="margin-left:10px">
                        {% if user_item.is_active %}
                            <span class="badge badge-success">{{ tr('active') }}</span>
                        {% else %}
                            <span class="badge badge-danger">{{ tr('blocked') }}</span>
                        {% endif %}
                    </div>
                </div>
                <hr style="border-color:rgba(255,255,255,0.1); margin:10px 0">
                <div class="row text-center mb-3">
                    <div class="col-6">
                        <div style="font-size:1.5rem; font-weight:700">{{ user_item.car_count }}</div>
                        <small class="text-muted">{{ tr('view_cars') }}</small>
                    </div>
                    <div class="col-6">
                        <div style="font-size:1.5rem; font-weight:700">{{ user_item.history_count }}</div>
                        <small class="text-muted">{{ tr('view_history') }}</small>
                    </div>
                </div>
                <div class="d-flex flex-wrap gap-2 justify-content-center">
                    <button class="btn btn-sm btn-outline-secondary" onclick="viewUserCars('{{ user_item.email }}')">
                        <i class="fas fa-car me-1"></i>{{ tr('view_cars') }}
                    </button>
                    <button class="btn btn-sm btn-outline-secondary" onclick="viewUserHistory('{{ user_item.email }}')">
                        <i class="fas fa-history me-1"></i>{{ tr('view_history') }}
                    </button>
                    {% if user_item.is_active %}
                        <button class="btn btn-sm btn-warning" onclick="blockUser('{{ user_item.email }}')" title="{{ tr('block_user') }}">
                            <i class="fas fa-ban me-1"></i>{{ tr('block_user') }}
                        </button>
                    {% else %}
                        <button class="btn btn-sm btn-success" onclick="unblockUser('{{ user_item.email }}')" title="{{ tr('unblock_user') }}">
                            <i class="fas fa-check-circle me-1"></i>{{ tr('unblock_user') }}
                        </button>
                    {% endif %}
                </div>
            </div>
        </div>
    {% else %}
        <div class="col-12">
            <div class="card p-4 text-center">{{ tr('no_users') }}</div>
        </div>
    {% endfor %}
</div>

<div class="modal fade" id="changePasswordModal">
    <div class="modal-dialog modal-dialog-centered">
        <div class="modal-content">
            <div class="modal-header border-0">
                <h5 class="modal-title fw-bold">{{ tr('change_password') }}</h5>
                <button class="btn-close" data-bs-dismiss="modal"></button>
            </div>
            <div class="modal-body">
                <form id="changePasswordForm" onsubmit="submitChangePassword(event)">
                    <div class="mb-3">
                        <label class="form-label fw-semibold">{{ tr('old_password') }}</label>
                        <div class="input-group">
                            <input type="password" name="old_password" class="form-control" required>
                            <button class="btn btn-outline-secondary" type="button" onclick="togglePassword(this)" style="border-radius:0 12px 12px 0">
                                <i class="fas fa-eye"></i>
                            </button>
                        </div>
                    </div>
                    <div class="mb-3">
                        <label class="form-label fw-semibold">{{ tr('new_password') }}</label>
                        <div class="input-group">
                            <input type="password" name="new_password" class="form-control" required>
                            <button class="btn btn-outline-secondary" type="button" onclick="togglePassword(this)" style="border-radius:0 12px 12px 0">
                                <i class="fas fa-eye"></i>
                            </button>
                        </div>
                    </div>
                    <div class="mb-3">
                        <label class="form-label fw-semibold">{{ tr('repeat_new_password') }}</label>
                        <div class="input-group">
                            <input type="password" name="new_password2" class="form-control" required>
                            <button class="btn btn-outline-secondary" type="button" onclick="togglePassword(this)" style="border-radius:0 12px 12px 0">
                                <i class="fas fa-eye"></i>
                            </button>
                        </div>
                    </div>
                    <button type="submit" class="btn btn-primary w-100 mt-2">{{ tr('change_password_btn') }}</button>
                </form>
            </div>
        </div>
    </div>
</div>
{% endblock %}

{% block scripts %}
<script>
    const userCarsModal = new bootstrap.Modal(document.getElementById('userCarsModal'));
    const userHistoryModal = new bootstrap.Modal(document.getElementById('userHistoryModal'));
    let passwordModal;

    document.addEventListener('DOMContentLoaded', () => {
        passwordModal = new bootstrap.Modal(document.getElementById('changePasswordModal'));
    });

    function openChangePasswordModal() { passwordModal.show(); }

    async function submitChangePassword(e) {
        e.preventDefault();
        let fd = new FormData(e.target);
        try {
            let r = await fetch('/change_password', { method: 'POST', body: fd });
            let d = await r.json();
            alert(d.message);
            if (d.success) {
                e.target.reset();
                passwordModal.hide();
            }
        } catch(e) {
            alert('{{ tr("error") }}');
        }
    }

    async function viewUserCars(email) {
        try {
            let r = await fetch('/admin/user_cars/' + email);
            let html = await r.text();
            document.getElementById('userCarsContent').innerHTML = html;
            userCarsModal.show();
        } catch(e) {
            alert('{{ tr("data_load_error") }}');
        }
    }

    async function viewUserHistory(email) {
        try {
            let r = await fetch('/admin/user_history/' + email);
            let html = await r.text();
            document.getElementById('userHistoryContent').innerHTML = html;
            userHistoryModal.show();
        } catch(e) {
            alert('{{ tr("data_load_error") }}');
        }
    }

    async function blockUser(email) {
        showConfirm(
            '{{ tr("confirm_block_user") }}',
            '{{ tr("confirm_block_user_msg") }}',
            '{{ tr("block_user") }}',
            async () => {
                try {
                    let r = await fetch('/admin/block_user/' + email, { method: 'POST' });
                    if (r.ok) location.reload();
                } catch(e) {
                    alert('{{ tr("error") }}');
                }
            }
        );
    }

    async function unblockUser(email) {
        try {
            let r = await fetch('/admin/unblock_user/' + email, { method: 'POST' });
            if (r.ok) location.reload();
        } catch(e) {
            alert('{{ tr("error") }}');
        }
    }

    async function deleteCarPhoto(userEmail, carName) {
        if (confirm('{{ tr("confirm_delete_photo") }}')) {
            try {
                let r = await fetch('/admin/delete_car_photo', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
                    body: 'email=' + encodeURIComponent(userEmail) + '&car_name=' + encodeURIComponent(carName)
                });
                if (r.ok) viewUserCars(userEmail);
            } catch(e) {
                alert('{{ tr("error") }}');
            }
        }
    }

    async function deleteUserCar(userEmail, carName) {
        if (confirm('{{ tr("confirm_delete_car") }} ' + carName + '?')) {
            try {
                let r = await fetch('/admin/delete_user_car', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
                    body: 'email=' + encodeURIComponent(userEmail) + '&car_name=' + encodeURIComponent(carName)
                });
                if (r.ok) {
                    viewUserCars(userEmail);
                    setTimeout(() => location.reload(), 500);
                }
            } catch(e) {
                alert('{{ tr("error") }}');
            }
        }
    }
</script>
{% endblock %}''',

    "admin_deleted.html": '''{% extends "base.html" %}
{% block content %}
<div class="admin-header d-flex justify-content-between align-items-center flex-wrap gap-2">
    <div>
        <h4 class="mb-0"><i class="fas fa-user-shield me-2"></i>{{ user }}</h4>
        <span class="badge badge-warning">{% if lang == 'ru' %}Администратор{% else %}Administrator{% endif %}</span>
    </div>
    <div class="d-flex gap-2">
        <a href="/admin" class="btn btn-outline-secondary">
            <i class="fas fa-arrow-left me-1"></i>{{ tr('back') }}
        </a>
        <a href="/logout" class="btn btn-outline-secondary">
            <i class="fas fa-sign-out-alt me-1"></i>{{ tr('logout') }}
        </a>
    </div>
</div>

<h2 class="mb-3 fw-bold">{{ tr('deleted_users_title') }}</h2>
<p class="text-muted mb-4">{{ tr('deleted_users_info') }}</p>

<div class="row">
    {% for user_item in deleted_users %}
        <div class="col-md-6 col-lg-4 mb-4">
            <div class="card p-4 user-card h-100">
                <div class="mb-3">
                    <h5 class="mb-1 text-truncate">{{ user_item.email }}</h5>
                    <small class="text-muted">{{ tr('registered') }}: {{ user_item.created_at }}</small>
                </div>
                <hr style="border-color:rgba(255,255,255,0.1); margin:10px 0">
                <div class="mb-3">
                    <div class="mb-2">
                        <i class="fas fa-trash me-2" style="color:#ff3b30"></i>
                        {{ tr('deleted_on') }}: {{ user_item.deleted_at }}
                    </div>
                </div>
                <div class="row text-center mb-3">
                    <div class="col-6">
                        <div style="font-size:1.5rem; font-weight:700">{{ user_item.car_count }}</div>
                        <small class="text-muted">{{ tr('cars') }}</small>
                    </div>
                    <div class="col-6">
                        <div style="font-size:1.5rem; font-weight:700">{{ user_item.history_count }}</div>
                        <small class="text-muted">{{ tr('records') }}</small>
                    </div>
                </div>
                <button class="btn btn-sm btn-success w-100" onclick="restoreUser('{{ user_item.email }}')">
                    <i class="fas fa-undo me-1"></i>{{ tr('restore') }}
                </button>
            </div>
        </div>
    {% else %}
        <div class="col-12">
            <div class="card p-4 text-center">{{ tr('no_deleted_users') }}</div>
        </div>
    {% endfor %}
</div>
{% endblock %}

{% block scripts %}
<script>
    async function restoreUser(email) {
        if (confirm('{{ tr("restore_user_confirm") }}')) {
            try {
                let r = await fetch('/admin/restore_user/' + email, { method: 'POST' });
                if (r.ok) location.reload();
                else alert('{{ tr("restore_error") }}');
            } catch(e) {
                alert('{{ tr("error") }}');
            }
        }
    }
</script>
{% endblock %}''',

    "admin_user_cars.html": '''<div class="row">
    {% if cars %}
        {% for car in cars %}
            <div class="col-md-6 mb-4">
                <div class="card p-3" style="background:rgba(255,255,255,0.1)">
                    {% if car.photo %}
                        <img src="{{ url_for('static', filename='uploads/' + car.photo) }}" 
                             style="width:100%; height:200px; object-fit:cover; border-radius:12px; margin-bottom:10px">
                    {% else %}
                        <div style="width:100%; height:200px; background:rgba(255,255,255,0.1); border-radius:12px; margin-bottom:10px; display:flex; align-items:center; justify-content:center">
                            <i class="fas fa-car fa-4x" style="color:rgba(255,255,255,0.3)"></i>
                        </div>
                    {% endif %}
                    <h6 class="fw-bold text-center">{{ car.name }}</h6>
                    {% if car.avg_consumption %}
                        <div class="text-center mb-2">
                            <span class="badge badge-info">{{ "%.1f"|format(car.avg_consumption) }} {{ tr('fuel_unit') }}/100{{ tr('km') }}</span>
                        </div>
                    {% endif %}
                    <div class="d-flex justify-content-center gap-2">
                        {% if car.photo %}
                            <button class="btn btn-sm btn-outline-warning" onclick="deleteCarPhoto('{{ user_email }}', '{{ car.name }}')">
                                <i class="fas fa-image me-1"></i>{{ tr('delete_photo') }}
                            </button>
                        {% endif %}
                        <button class="btn btn-sm btn-outline-danger" onclick="deleteUserCar('{{ user_email }}', '{{ car.name }}')">
                            <i class="fas fa-trash me-1"></i>{{ tr('delete_car') }}
                        </button>
                    </div>
                </div>
            </div>
        {% endfor %}
    {% else %}
        <div class="col-12 text-center">
            <p>{{ tr('no_user_cars') }}</p>
        </div>
    {% endif %}
</div>''',

    "admin_user_history.html": '''{% if history %}
    {% for e in history %}
        <div class="card mb-3 p-3" style="background:rgba(255,255,255,0.1)">
            <div class="fw-semibold mb-2">{{ e.date }} – {{ e.car or tr('no_car') }}</div>
            <div class="row small text-muted">
                <div class="col-6"><i class="fas fa-road me-1"></i>{{ e.distance }} {{ tr('km') }}</div>
                <div class="col-6"><i class="fas fa-gas-pump me-1"></i>{{ e.fuel }} {{ tr('fuel_unit') }}</div>
                <div class="col-6"><i class="fas fa-tachometer-alt me-1"></i>{{ e.consumption }} {{ tr('fuel_unit') }}/100{{ tr('km') }}</div>
                <div class="col-6"><i class="fas fa-coins me-1"></i>{{ "%.2f"|format(e.cost) }} {{ e.currency }}</div>
            </div>
        </div>
    {% endfor %}
{% else %}
    <div class="text-center">
        <p>{{ tr('empty_history') }}</p>
    </div>
{% endif %}'''
}

# FLASK INITIALIZATION
app = Flask(__name__)
app.jinja_loader = DictLoader(TEMPLATES)
app.secret_key = os.urandom(24)

UPLOAD_FOLDER = os.path.join('static', 'uploads')
os.makedirs(UPLOAD_FOLDER, exist_ok=True)
os.makedirs(os.path.join('static', 'images'), exist_ok=True)
app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER
app.config['MAX_CONTENT_LENGTH'] = 50 * 1024 * 1024

app.add_template_global(tr, 'tr')
app.add_template_global(CURRENCIES, 'currencies')


def is_admin():
    """Check if current user is admin"""
    if 'user' not in session:
        return False
    return db_is_user_admin(session['user'])


@app.context_processor
def inject_admin():
    """Inject admin status into templates"""
    return {
        'is_admin': is_admin(),
        'lang': session.get('language', 'ru')
    }


# ROUTES
@app.route('/')
def index():
    """Main page route"""
    if 'user' in session and is_admin():
        return redirect(url_for('admin'))

    cars, history = [], []
    if 'user' in session:
        try:
            cars = db_get_cars(session['user'])
            history = db_get_history(session['user'])
        except:
            flash(tr('database_error'), 'error')

    return render_template(
        "index.html",
        cars=cars,
        history=history,
        lang=session.get('language', 'ru'),
        currency_symbol=CURRENCIES.get(session.get('currency', '₽ RUB'), '₽'),
        user=session.get('user')
    )


@app.route('/calculate', methods=['POST'])
def calculate():
    """Calculate fuel consumption and cost"""
    try:
        dist = float(request.form.get('distance', 0))
        price = float(request.form.get('price', 0))
    except:
        return jsonify({'error': tr('enter_numbers')}), 400

    if dist <= 0:
        return jsonify({'error': tr('distance_positive')}), 400

    if price <= 0:
        return jsonify({'error': tr('price_positive')}), 400

    mode = request.form.get('mode', 'consumption')
    car_name = request.form.get('car', tr('no_car')) if 'user' in session else tr('no_car')
    currency = session.get('currency', '₽ RUB') if 'user' in session else '₽ RUB'
    currency_symbol = CURRENCIES.get(currency, '₽')

    if mode == 'consumption':
        try:
            fuel = float(request.form.get('fuel', 0))
        except:
            return jsonify({'error': tr('enter_numbers')}), 400

        if fuel <= 0:
            return jsonify({'error': tr('fuel_positive')}), 400

        consumption = (fuel / dist) * 100
        cost = fuel * price
    else:
        try:
            avg_input = float(request.form.get('avg_consumption', 0))
        except:
            return jsonify({'error': tr('enter_numbers')}), 400

        if avg_input <= 0:
            return jsonify({'error': tr('distance_positive2')}), 400

        consumption = avg_input
        fuel = (avg_input / 100) * dist
        cost = ((avg_input / 100) * dist) * price

    if 'user' in session:
        try:
            db_add_history(session['user'], {
                'date': datetime.now().strftime("%d.%m.%Y %H:%M"),
                'car': car_name,
                'distance': dist,
                'fuel': round(fuel, 2),
                'price': price,
                'currency': currency_symbol,
                'consumption': round(consumption, 2),
                'cost': round(cost, 2)
            })
            if car_name not in (tr('no_car'), '-- No car --', '— Без авто —', '— No car —'):
                db_update_avg(session['user'], car_name)
        except:
            pass

    return jsonify({
        'consumption': round(consumption, 2),
        'fuel': round(fuel, 2),
        'cost': round(cost, 2),
        'mode': mode,
        'currency': currency_symbol
    })


@app.route('/history_html')
def history_html():
    """Return history HTML fragment"""
    if 'user' not in session:
        return "<div class='card p-4 text-center'>" + tr('login_for_history') + "</div>"

    return render_template(
        "history_list.html",
        history=db_get_history(session['user'], request.args.get('car', '')),
        lang=session.get('language', 'ru'),
        user=session['user'],
        currency_symbol=CURRENCIES.get(session.get('currency', '₽ RUB'), '₽')
    )


@app.route('/auth', methods=['GET', 'POST'])
def auth():
    """Authentication route"""
    if 'user' in session:
        if is_admin():
            return redirect(url_for('admin'))
        return redirect(url_for('profile'))

    if request.method == 'POST':
        email = request.form['email'].strip().lower()
        pw = request.form['password']

        if request.form.get('form_type') == 'register':
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
                try:
                    db_create_user(email, hash_pw(pw))
                    session['user'] = email
                    session['language'] = 'ru'
                    session['currency'] = '₽ RUB'
                    return redirect(url_for('profile'))
                except Exception as e:
                    flash(f'{tr("database_error")}: {str(e)}', 'error')
        else:
            if not is_valid_email(email):
                flash(tr('invalid_email'), 'error')
            else:
                is_deleted, deleted_by = db_is_user_deleted(email)
                if is_deleted:
                    flash(tr('account_deleted'), 'error')
                    return render_template("auth.html", lang=session.get('language', 'ru'))

                stored_password = db_get_user(email)
                if stored_password and stored_password == hash_pw(pw):
                    is_active = db_is_user_active(email)
                    is_user_admin = db_is_user_admin(email)

                    if not is_active and not is_user_admin:
                        flash(tr('account_blocked'), 'error')
                        return render_template("auth.html", lang=session.get('language', 'ru'))

                    session['user'] = email
                    s = db_get_settings(email)
                    session['language'] = s['language']
                    session['currency'] = s['currency']

                    if is_user_admin:
                        return redirect(url_for('admin'))
                    return redirect(url_for('profile'))

                flash(tr('wrong_credentials'), 'error')

    return render_template("auth.html", lang=session.get('language', 'ru'))


@app.route('/logout')
def logout():
    """Logout route"""
    session.clear()
    return redirect(url_for('index'))


@app.route('/profile')
def profile():
    """Profile page route"""
    if 'user' not in session:
        return redirect(url_for('auth'))

    if is_admin():
        return redirect(url_for('admin'))

    try:
        cars = db_get_cars(session['user'])
        car_stats = {c['name']: db_get_car_stats(session['user'], c['name']) for c in cars}
    except:
        cars, car_stats = [], {}

    return render_template(
        "profile.html",
        cars=cars,
        car_stats=car_stats,
        lang=session.get('language', 'ru'),
        user=session['user'],
        currency_symbol=CURRENCIES.get(session.get('currency', '₽ RUB'), '₽')
    )


@app.route('/profile_cars_html')
def profile_cars_html():
    """Return profile cars HTML fragment"""
    if 'user' not in session:
        return ''
    return render_profile_cars()


@app.route('/add_car', methods=['POST'])
def add_car():
    """Add car route"""
    if 'user' not in session:
        return redirect(url_for('auth'))

    name = request.form['name'].strip()
    if not name:
        return redirect(url_for('profile'))

    photo = request.files.get('photo')
    photo_name = save_car_photo(photo) if photo and photo.filename else None

    try:
        db_add_car(session['user'], name, photo_name)
    except:
        flash(tr('database_error'), 'error')

    return render_profile_cars()


@app.route('/edit_car', methods=['POST'])
def edit_car():
    """Edit car route"""
    if 'user' not in session:
        return redirect(url_for('auth'))

    old = request.form['old_name'].strip()
    new = request.form['new_name'].strip()

    if not new:
        return jsonify({'success': False, 'message': tr('enter_name')})

    cars = db_get_cars(session['user'])
    old_photo = next((c['photo'] for c in cars if c['name'] == old), None)
    photo = request.files.get('photo')
    photo_name = save_car_photo(photo, old_photo) if photo and photo.filename else None

    try:
        db_update_car(session['user'], old, new, photo_name)
    except:
        flash(tr('database_error'), 'error')

    return render_profile_cars()


@app.route('/delete_car/<name>', methods=['POST'])
def delete_car(name):
    """Delete car route"""
    if 'user' not in session:
        return redirect(url_for('auth'))

    try:
        db_delete_car(session['user'], name)
    except:
        flash(tr('database_error'), 'error')

    return render_profile_cars()


@app.route('/change_password', methods=['POST'])
def change_password():
    """Change password route"""
    if 'user' not in session:
        return jsonify({'success': False}), 401

    old = request.form['old_password']
    new = request.form['new_password']
    new2 = request.form['new_password2']

    if not old or not new:
        return jsonify({'success': False, 'message': tr('fill_all')})

    if len(new) < 6:
        return jsonify({'success': False, 'message': tr('password_short')})

    if new != new2:
        return jsonify({'success': False, 'message': tr('passwords_mismatch')})

    if db_get_user(session['user']) != hash_pw(old):
        return jsonify({'success': False, 'message': tr('old_password_wrong')})

    try:
        db_update_password(session['user'], hash_pw(new))
        return jsonify({'success': True, 'message': tr('password_changed')})
    except:
        return jsonify({'success': False, 'message': tr('database_error')})


@app.route('/delete_account', methods=['POST'])
def delete_account():
    """Delete account route"""
    if 'user' not in session:
        return redirect(url_for('auth'))

    try:
        db_delete_user(session['user'])
        session.clear()
        flash(tr('account_deactivated'), 'info')
    except:
        flash(tr('database_error'), 'error')

    return redirect(url_for('index'))


@app.route('/clear_history', methods=['POST'])
def clear_history():
    """Clear history route"""
    if 'user' not in session:
        return '', 401

    db_clear_history(session['user'])
    return '', 200


@app.route('/delete_history/<int:eid>', methods=['POST'])
def delete_history(eid):
    """Delete history entry route"""
    if 'user' not in session:
        return '', 401

    db_delete_history_entry(session['user'], eid)
    return '', 200


@app.route('/settings', methods=['GET', 'POST'])
def settings():
    """Settings route"""
    if request.method == 'POST':
        lang = request.form.get('language', 'ru')
        cur = request.form.get('currency', '₽ RUB')
        session['language'] = lang
        session['currency'] = cur

        if 'user' in session:
            db_update_settings(session['user'], 'light', lang, cur)

        return redirect(url_for('settings'))

    settings_data = {
        'theme': 'light',
        'language': session.get('language', 'ru'),
        'currency': session.get('currency', '₽ RUB')
    }

    if 'user' in session:
        settings_data = db_get_settings(session['user'])
        session['language'] = settings_data['language']
        session['currency'] = settings_data['currency']

    return render_template(
        "settings.html",
        settings=settings_data,
        currencies=CURRENCIES,
        lang=session.get('language', 'ru'),
        user=session.get('user')
    )


@app.route('/about')
def about():
    """About page route"""
    return render_template(
        "about.html",
        lang=session.get('language', 'ru'),
        user=session.get('user')
    )


# ADMIN ROUTES
@app.route('/admin')
def admin():
    """Admin panel route"""
    if not is_admin():
        flash(tr('access_denied'), 'error')
        return redirect(url_for('index'))

    try:
        stats = db_get_system_stats()
        users = db_get_all_users()
    except Exception as e:
        flash(f'{tr("data_load_error")}: {e}', 'error')
        stats = {
            'total_users': 0,
            'active_users': 0,
            'blocked_users': 0,
            'deleted_users': 0,
            'total_cars': 0,
            'total_history': 0
        }
        users = []

    return render_template(
        "admin.html",
        stats=stats,
        users=users,
        lang=session.get('language', 'ru'),
        user=session['user']
    )


@app.route('/admin/deleted_users')
def admin_deleted_users():
    """Admin deleted users route"""
    if not is_admin():
        flash(tr('access_denied'), 'error')
        return redirect(url_for('index'))

    deleted_users = db_get_deleted_users()
    return render_template(
        "admin_deleted.html",
        deleted_users=deleted_users,
        lang=session.get('language', 'ru'),
        user=session['user']
    )


@app.route('/admin/user_cars/<email>')
def admin_user_cars(email):
    """Admin view user cars route"""
    if not is_admin():
        return tr('access_denied'), 403

    cars = db_get_user_cars(email)
    return render_template(
        "admin_user_cars.html",
        cars=cars,
        user_email=email,
        lang=session.get('language', 'ru')
    )


@app.route('/admin/user_history/<email>')
def admin_user_history(email):
    """Admin view user history route"""
    if not is_admin():
        return tr('access_denied'), 403

    history = db_get_user_history(email)
    return render_template(
        "admin_user_history.html",
        history=history,
        lang=session.get('language', 'ru')
    )


@app.route('/admin/delete_car_photo', methods=['POST'])
def admin_delete_car_photo():
    """Admin delete car photo route"""
    if not is_admin():
        return jsonify({'error': tr('access_denied')}), 403

    email = request.form.get('email')
    car_name = request.form.get('car_name')

    if email and car_name:
        db_delete_car_photo(email, car_name)
        log_admin_action(session['user'], 'delete_car_photo', email, f'Delete car photo: {car_name}')
        return jsonify({'success': True})

    return jsonify({'error': tr('invalid_params')}), 400


@app.route('/admin/delete_user_car', methods=['POST'])
def admin_delete_user_car():
    """Admin delete user car route"""
    if not is_admin():
        return jsonify({'error': tr('access_denied')}), 403

    email = request.form.get('email')
    car_name = request.form.get('car_name')

    if email and car_name:
        db_delete_car(email, car_name)
        log_admin_action(session['user'], 'delete_car', email, f'Delete car: {car_name}')
        return jsonify({'success': True})

    return jsonify({'error': tr('invalid_params')}), 400


@app.route('/admin/block_user/<email>', methods=['POST'])
def admin_block_user(email):
    """Admin block user route"""
    if not is_admin():
        return jsonify({'error': tr('access_denied')}), 403

    if email == session['user']:
        return jsonify({'error': tr('cannot_block_self')}), 400

    try:
        db_block_user(email)
        log_admin_action(session['user'], 'block_user', email)
        return jsonify({'success': True})
    except Exception as e:
        return jsonify({'error': str(e)}), 500


@app.route('/admin/unblock_user/<email>', methods=['POST'])
def admin_unblock_user(email):
    """Admin unblock user route"""
    if not is_admin():
        return jsonify({'error': tr('access_denied')}), 403

    try:
        db_unblock_user(email)
        log_admin_action(session['user'], 'unblock_user', email)
        return jsonify({'success': True})
    except Exception as e:
        return jsonify({'error': str(e)}), 500


@app.route('/admin/restore_user/<email>', methods=['POST'])
def admin_restore_user(email):
    """Admin restore user route"""
    if not is_admin():
        return jsonify({'error': tr('access_denied')}), 403

    try:
        db_restore_user(email)
        log_admin_action(session['user'], 'restore_user', email)
        return jsonify({'success': True, 'message': tr('user_restored')})
    except Exception as e:
        return jsonify({'error': str(e)}), 500


# ENTRY POINT
if __name__ == '__main__':
    os.makedirs(os.path.join('static', 'uploads'), exist_ok=True)
    os.makedirs(os.path.join('static', 'images'), exist_ok=True)

    if init_db_pool():
        create_tables()
        if create_admin():
            print("\n" + "=" * 60)
            print("  FullCalcPro successfully started!")
            print("  Open in browser: http://127.0.0.1:5000")
            print(f"  Admin panel: http://127.0.0.1:5000/admin")
            print(f"  Admin login: {ADMIN_CONFIG['email']}")
            print(f"  Admin password: {ADMIN_CONFIG['password']}")
            print("=" * 60 + "\n")
            app.run(debug=True, host='127.0.0.1', port=5000)
        else:
            print("\n!!! ADMINISTRATOR CREATION ERROR !!!")
    else:
        print("\n!!! FAILED TO CONNECT TO POSTGRESQL !!!")
