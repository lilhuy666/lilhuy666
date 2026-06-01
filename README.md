# ============================================
# ВСЕ НЕОБХОДИМЫЕ ИМПОРТЫ
# ============================================
import pymysql
import sys
import os
from datetime import datetime

# ============================================
# НАСТРОЙКИ ПОДКЛЮЧЕНИЯ (ИЗМЕНИ НА СВОИ!)
# ============================================
DB_CONFIG = {
    'host': 'localhost',        # IP адрес или localhost
    'user': 'root',             # имя пользователя MySQL
    'password': '12345',        # твой пароль MySQL (ОБЯЗАТЕЛЬНО ПОМЕНЯЙ!)
    'database': 'mydb',         # название базы данных
    'charset': 'utf8mb4',       # кодировка
    'cursorclass': pymysql.cursors.Cursor
}

# ============================================
# ФУНКЦИИ ДЛЯ РАБОТЫ С БАЗОЙ ДАННЫХ
# ============================================

def clear_screen():
    """Очистка экрана"""
    os.system('cls' if os.name == 'nt' else 'clear')

def print_header(text):
    """Красивый заголовок"""
    print("\n" + "="*60)
    print(f"  {text}")
    print("="*60)

def create_connection():
    """
    Создание подключения к MySQL
    Возвращает объект соединения или None при ошибке
    """
    try:
        print("🔄 Подключение к MySQL...")
        conn = pymysql.connect(**DB_CONFIG)
        print("✅ Успешное подключение к MySQL!")
        return conn
    except pymysql.Error as e:
        print(f"❌ Ошибка подключения: {e}")
        print("\n📋 ВОЗМОЖНЫЕ ПРИЧИНЫ:")
        print("1. MySQL сервер не запущен")
        print("2. Неправильный пароль")
        print("3. База данных не существует")
        print("4. Порт занят другой программой")
        return None

def create_database(cursor):
    """
    Создание базы данных, если её нет
    """
    try:
        cursor.execute("CREATE DATABASE IF NOT EXISTS mydb")
        cursor.execute("USE mydb")
        print("✅ База данных 'mydb' готова к работе")
        return True
    except pymysql.Error as e:
        print(f"❌ Ошибка создания базы данных: {e}")
        return False

def create_tables(cursor):
    """
    Создание всех необходимых таблиц
    """
    try:
        # Таблица пользователей
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS users (
                id INT AUTO_INCREMENT PRIMARY KEY,
                name VARCHAR(100) NOT NULL,
                phone VARCHAR(20),
                email VARCHAR(100),
                age INT,
                city VARCHAR(50),
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        
        # Таблица заказов (для демонстрации связей)
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS orders (
                id INT AUTO_INCREMENT PRIMARY KEY,
                user_id INT,
                product VARCHAR(100),
                price DECIMAL(10, 2),
                order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
            )
        """)
        
        print("✅ Таблицы созданы успешно")
        return True
    except pymysql.Error as e:
        print(f"❌ Ошибка создания таблиц: {e}")
        return False

def insert_test_data(cursor):
    """
    Добавление тестовых данных
    """
    try:
        # Проверяем, есть ли данные в таблице users
        cursor.execute("SELECT COUNT(*) FROM users")
        user_count = cursor.fetchone()[0]
        
        if user_count == 0:
            users = [
                ('Иван Иванов', '+7-999-123-45-67', 'ivan@mail.ru', 25, 'Москва'),
                ('Мария Петрова', '+7-999-765-43-21', 'maria@mail.ru', 30, 'Санкт-Петербург'),
                ('Алексей Сидоров', '+7-999-555-33-22', 'alex@mail.ru', 35, 'Казань'),
                ('Елена Смирнова', '+7-999-111-22-33', 'elena@mail.ru', 28, 'Новосибирск'),
                ('Дмитрий Козлов', '+7-999-444-55-66', 'dmitry@mail.ru', 32, 'Екатеринбург')
            ]
            
            cursor.executemany(
                "INSERT INTO users (name, phone, email, age, city) VALUES (%s, %s, %s, %s, %s)",
                users
            )
            print("✅ Добавлены тестовые пользователи")
        else:
            print(f"ℹ️ В таблице уже есть {user_count} пользователей")
        
        # Проверяем, есть ли заказы
        cursor.execute("SELECT COUNT(*) FROM orders")
        order_count = cursor.fetchone()[0]
        
        if order_count == 0 and user_count > 0:
            orders = [
                (1, 'Ноутбук', 55000.00),
                (1, 'Мышь', 2500.00),
                (2, 'Монитор', 15000.00),
                (3, 'Клавиатура', 5000.00),
                (4, 'Наушники', 8000.00)
            ]
            
            cursor.executemany(
                "INSERT INTO orders (user_id, product, price) VALUES (%s, %s, %s)",
                orders
            )
            print("✅ Добавлены тестовые заказы")
            
    except pymysql.Error as e:
        print(f"❌ Ошибка добавления данных: {e}")

def show_all_users(cursor):
    """
    Показать всех пользователей в красивой таблице
    """
    try:
        cursor.execute("""
            SELECT id, name, phone, email, age, city, 
                   DATE_FORMAT(created_at, '%d.%m.%Y %H:%i') as created
            FROM users 
            ORDER BY id
        """)
        users = cursor.fetchall()
        
        if not users:
            print("\n📋 База данных пуста")
            return
        
        print_header("СПИСОК ПОЛЬЗОВАТЕЛЕЙ")
        
        # Заголовок таблицы
        header = f"{'ID':<5} {'Имя':<20} {'Телефон':<18} {'Email':<22} {'Возраст':<8} {'Город':<15} {'Создан'}"
        print(header)
        print("-" * len(header))
        
        # Данные
        for user in users:
            print(f"{user[0]:<5} {user[1]:<20} {user[2]:<18} {user[3]:<22} "
                  f"{user[4]:<8} {user[5]:<15} {user[6]}")
            
        print(f"\n📊 Всего пользователей: {len(users)}")
        
    except pymysql.Error as e:
        print(f"❌ Ошибка вывода данных: {e}")

def show_user_orders(cursor):
    """
    Показать заказы пользователей
    """
    try:
        cursor.execute("""
            SELECT u.name, o.product, o.price, 
                   DATE_FORMAT(o.order_date, '%d.%m.%Y') as date
            FROM users u
            JOIN orders o ON u.id = o.user_id
            ORDER BY o.order_date DESC
        """)
        orders = cursor.fetchall()
        
        if not orders:
            print("\n📋 Нет заказов")
            return
        
        print_header("ЗАКАЗЫ ПОЛЬЗОВАТЕЛЕЙ")
        
        header = f"{'Пользователь':<20} {'Товар':<15} {'Цена':<12} {'Дата'}"
        print(header)
        print("-" * len(header))
        
        total = 0
        for order in orders:
            print(f"{order[0]:<20} {order[1]:<15} {order[2]:<12.2f} {order[3]}")
            total += order[2]
            
        print(f"\n💰 Общая сумма заказов: {total:.2f} руб.")
        
    except pymysql.Error as e:
        print(f"❌ Ошибка вывода заказов: {e}")

def add_new_user(cursor):
    """
    Добавление нового пользователя через консоль
    """
    print_header("ДОБАВЛЕНИЕ НОВОГО ПОЛЬЗОВАТЕЛЯ")
    
    try:
        name = input("👤 Введите имя: ").strip()
        if not name:
            print("❌ Имя не может быть пустым!")
            return
        
        phone = input("📱 Введите телефон: ").strip()
        email = input("📧 Введите email: ").strip()
        
        if not email:
            print("❌ Email не может быть пустым!")
            return
        
        age_input = input("🎂 Введите возраст (или Enter для пропуска): ").strip()
        age = int(age_input) if age_input else None
        
        city = input("🏙️ Введите город (или Enter для пропуска): ").strip()
        city = city if city else None
        
        cursor.execute(
            """INSERT INTO users (name, phone, email, age, city) 
               VALUES (%s, %s, %s, %s, %s)""",
            (name, phone, email, age, city)
        )
        
        print(f"✅ Пользователь '{name}' успешно добавлен!")
        
    except ValueError:
        print("❌ Возраст должен быть числом!")
    except pymysql.Error as e:
        if "Duplicate" in str(e):
            print("❌ Пользователь с таким email уже существует!")
        else:
            print(f"❌ Ошибка: {e}")

def update_user(cursor):
    """
    Обновление данных пользователя
    """
    print_header("ОБНОВЛЕНИЕ ДАННЫХ ПОЛЬЗОВАТЕЛЯ")
    
    try:
        user_id = input("Введите ID пользователя: ").strip()
        if not user_id:
            print("❌ ID не может быть пустым!")
            return
        
        # Проверяем существование пользователя
        cursor.execute("SELECT * FROM users WHERE id = %s", (int(user_id),))
        user = cursor.fetchone()
        
        if not user:
            print(f"❌ Пользователь с ID {user_id} не найден!")
            return
        
        print(f"\nТекущие данные: {user[1]} | {user[2]} | {user[3]} | {user[4]} | {user[5]}")
        
        print("\nВведите новые данные (Enter - оставить без изменений):")
        
        name = input(f"👤 Имя [{user[1]}]: ").strip()
        phone = input(f"📱 Телефон [{user[2]}]: ").strip()
        email = input(f"📧 Email [{user[3]}]: ").strip()
        age_input = input(f"🎂 Возраст [{user[4]}]: ").strip()
        city = input(f"🏙️ Город [{user[5]}]: ").strip()
        
        # Формируем запрос на обновление
        updates = []
        values = []
        
        if name:
            updates.append("name = %s")
            values.append(name)
        if phone:
            updates.append("phone = %s")
            values.append(phone)
        if email:
            updates.append("email = %s")
            values.append(email)
        if age_input:
            updates.append("age = %s")
            values.append(int(age_input))
        if city:
            updates.append("city = %s")
            values.append(city)
        
        if updates:
            values.append(int(user_id))
            query = f"UPDATE users SET {', '.join(updates)} WHERE id = %s"
            cursor.execute(query, values)
            print("✅ Данные обновлены успешно!")
        else:
            print("ℹ️ Нет данных для обновления")
            
    except ValueError:
        print("❌ Некорректный ввод!")
    except pymysql.Error as e:
        print(f"❌ Ошибка: {e}")

def delete_user(cursor):
    """
    Удаление пользователя
    """
    print_header("УДАЛЕНИЕ ПОЛЬЗОВАТЕЛЯ")
    
    try:
        user_id = input("Введите ID пользователя для удаления: ").strip()
        if not user_id:
            print("❌ ID не может быть пустым!")
            return
        
        # Проверяем существование
        cursor.execute("SELECT name FROM users WHERE id = %s", (int(user_id),))
        user = cursor.fetchone()
        
        if not user:
            print(f"❌ Пользователь с ID {user_id} не найден!")
            return
        
        # Подтверждение
        confirm = input(f"⚠️ Вы уверены, что хотите удалить '{user[0]}'? (да/нет): ").strip().lower()
        
        if confirm in ['да', 'yes', 'y']:
            cursor.execute("DELETE FROM users WHERE id = %s", (int(user_id),))
            print(f"✅ Пользователь '{user[0]}' удален!")
        else:
            print("ℹ️ Удаление отменено")
            
    except ValueError:
        print("❌ Некорректный ID!")
    except pymysql.Error as e:
        print(f"❌ Ошибка: {e}")

def show_statistics(cursor):
    """
    Показать статистику
    """
    print_header("СТАТИСТИКА")
    
    try:
        # Количество пользователей
        cursor.execute("SELECT COUNT(*) FROM users")
        user_count = cursor.fetchone()[0]
        
        # Средний возраст
        cursor.execute("SELECT AVG(age) FROM users WHERE age IS NOT NULL")
        avg_age = cursor.fetchone()[0]
        
        # Количество городов
        cursor.execute("SELECT COUNT(DISTINCT city) FROM users WHERE city IS NOT NULL")
        city_count = cursor.fetchone()[0]
        
        # Самый популярный город
        cursor.execute("""
            SELECT city, COUNT(*) as cnt 
            FROM users 
            WHERE city IS NOT NULL 
            GROUP BY city 
            ORDER BY cnt DESC 
            LIMIT 1
        """)
        top_city = cursor.fetchone()
        
        print(f"👥 Всего пользователей: {user_count}")
        print(f"🎂 Средний возраст: {avg_age:.1f} лет" if avg_age else "🎂 Средний возраст: нет данных")
        print(f"🏙️ Количество городов: {city_count}")
        if top_city:
            print(f"🌟 Самый популярный город: {top_city[0]} ({top_city[1]} чел.)")
        
        # Статистика заказов
        cursor.execute("SELECT COUNT(*), SUM(price), AVG(price) FROM orders")
        orders_stats = cursor.fetchone()
        
        if orders_stats[0] > 0:
            print(f"\n📦 Всего заказов: {orders_stats[0]}")
            print(f"💰 Общая сумма: {orders_stats[1]:.2f} руб.")
            print(f"💵 Средний чек: {orders_stats[2]:.2f} руб.")
        
    except pymysql.Error as e:
        print(f"❌ Ошибка: {e}")

def show_menu():
    """
    Показать главное меню
    """
    print_header("ГЛАВНОЕ МЕНЮ")
    print("1. 📋 Показать всех пользователей")
    print("2. 👤 Добавить нового пользователя")
    print("3. ✏️ Обновить данные пользователя")
    print("4. ❌ Удалить пользователя")
    print("5. 📦 Показать заказы")
    print("6. 📊 Статистика")
    print("7. ℹ️ О программе")
    print("0. 🚪 Выход")
    print("-" * 60)

def show_about():
    """
    Информация о программе
    """
    print_header("О ПРОГРАММЕ")
    print("📌 Программа для работы с MySQL в Python")
    print("📌 Версия: 1.0")
    print("📌 База данных: MySQL")
    print("📌 Драйвер: PyMySQL")
    print("\n📋 Возможности:")
    print("   • Просмотр пользователей и заказов")
    print("   • Добавление, редактирование, удаление")
    print("   • Статистика и аналитика")
    print("   • Связанные таблицы (users ↔ orders)")

def main():
    """
    Главная функция программы
    """
    clear_screen()
    print_header("ПРОГРАММА РАБОТЫ С MySQL")
    print("Запуск приложения...")
    print(f"Время запуска: {datetime.now().strftime('%d.%m.%Y %H:%M:%S')}")
    
    # 1. Подключаемся к базе данных
    conn = create_connection()
    if conn is None:
        print("\n❌ Не удалось подключиться к базе данных.")
        print("Проверьте настройки в DB_CONFIG и запустите MySQL сервер.")
        sys.exit(1)
    
    cursor = conn.cursor()
    
    # 2. Создаем базу данных и таблицы
    if not create_database(cursor):
        cursor.close()
        conn.close()
        sys.exit(1)
    
    if not create_tables(cursor):
        cursor.close()
        conn.close()
        sys.exit(1)
    
    # 3. Добавляем тестовые данные
    insert_test_data(cursor)
    conn.commit()
    
    # 4. Основной цикл программы
    while True:
        try:
            show_menu()
            choice = input("\n👉 Выберите действие: ").strip()
            
            if choice == '1':
                show_all_users(cursor)
                
            elif choice == '2':
                add_new_user(cursor)
                conn.commit()
                
            elif choice == '3':
                update_user(cursor)
                conn.commit()
                
            elif choice == '4':
                delete_user(cursor)
                conn.commit()
                
            elif choice == '5':
                show_user_orders(cursor)
                
            elif choice == '6':
                show_statistics(cursor)
                
            elif choice == '7':
                show_about()
                
            elif choice == '0':
                print("\n👋 До свидания!")
                break
                
            else:
                print("❌ Неверный выбор! Введите число от 0 до 7")
            
            input("\nНажмите Enter для продолжения...")
            clear_screen()
            
        except KeyboardInterrupt:
            print("\n\n⚠️ Прерывание работы...")
            break
        except Exception as e:
            print(f"❌ Непредвиденная ошибка: {e}")
    
    # 5. Закрываем соединение
    cursor.close()
    conn.close()
    print("\n🔌 Соединение с базой данных закрыто")
    print("✨ Программа завершена")

# ============================================
# ТОЧКА ВХОДА
# ============================================
if __name__ == "__main__":
    main()
