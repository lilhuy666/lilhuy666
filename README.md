import pymysql

# ============================================
# НАСТРОЙКИ - ПОМЕНЯЙ НА СВОИ ДАННЫЕ!
# ============================================
DB_CONFIG = {
    'host': 'localhost',        # адрес сервера
    'user': 'root',             # имя пользователя MySQL
    'password': '12345',        # твой пароль от MySQL
    'database': 'mydb',         # название базы данных
    'charset': 'utf8mb4'        # кодировка
}

def create_connection():
    """Создание подключения к базе данных"""
    try:
        conn = pymysql.connect(**DB_CONFIG)
        print("✅ Успешное подключение к MySQL!")
        return conn
    except Exception as e:
        print(f"❌ Ошибка подключения: {e}")
        print("\nПроверь:")
        print("1. MySQL сервер запущен?")
        print("2. Правильный пароль?")
        print("3. База данных 'mydb' существует?")
        return None

def create_database_and_table(cursor):
    """Создание базы данных и таблицы"""
    try:
        # Создаём базу данных (если не существует)
        cursor.execute("CREATE DATABASE IF NOT EXISTS mydb")
        cursor.execute("USE mydb")
        
        # Создаём таблицу users
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS users (
                id INT AUTO_INCREMENT PRIMARY KEY,
                name VARCHAR(100) NOT NULL,
                phone VARCHAR(20),
                email VARCHAR(100)
            )
        """)
        print("✅ База данных и таблица готовы")
    except Exception as e:
        print(f"❌ Ошибка создания таблицы: {e}")

def add_test_data(cursor):
    """Добавление тестовых данных"""
    try:
        # Проверяем, есть ли уже данные
        cursor.execute("SELECT COUNT(*) FROM users")
        count = cursor.fetchone()[0]
        
        if count == 0:
            users = [
                ('Иван Иванов', '+7-999-123-45-67', 'ivan@mail.ru'),
                ('Мария Петрова', '+7-999-765-43-21', 'maria@mail.ru'),
                ('Алексей Сидоров', '+7-999-555-33-22', 'alex@mail.ru')
            ]
            
            cursor.executemany(
                "INSERT INTO users (name, phone, email) VALUES (%s, %s, %s)",
                users
            )
            print("✅ Добавлены тестовые данные")
    except Exception as e:
        print(f"❌ Ошибка добавления данных: {e}")

def show_all_users(cursor):
    """Показать всех пользователей"""
    try:
        cursor.execute("SELECT * FROM users")
        users = cursor.fetchall()
        
        print("\n" + "="*70)
        print(f"{'ID':<5} {'Имя':<25} {'Телефон':<20} {'Email':<20}")
        print("="*70)
        
        for user in users:
            print(f"{user[0]:<5} {user[1]:<25} {user[2]:<20} {user[3]:<20}")
        
        print("="*70)
    except Exception as e:
        print(f"❌ Ошибка вывода данных: {e}")

def add_new_user(cursor):
    """Добавление нового пользователя с клавиатуры"""
    print("\n📝 ДОБАВЛЕНИЕ НОВОГО ПОЛЬЗОВАТЕЛЯ")
    name = input("Введите имя: ").strip()
    phone = input("Введите телефон: ").strip()
    email = input("Введите email: ").strip()
    
    if name and phone and email:
        try:
            cursor.execute(
                "INSERT INTO users (name, phone, email) VALUES (%s, %s, %s)",
                (name, phone, email)
            )
            print(f"✅ Пользователь {name} добавлен!")
        except Exception as e:
            print(f"❌ Ошибка: {e}")
    else:
        print("❌ Все поля обязательны для заполнения!")

def main():
    """Главная функция"""
    print("\n" + "="*50)
    print("ПРОГРАММА РАБОТЫ С MySQL")
    print("="*50)
    
    # Подключаемся к базе
    conn = create_connection()
    if conn is None:
        return
    
    cursor = conn.cursor()
    
    # Создаём структуру БД
    create_database_and_table(cursor)
    
    # Добавляем тестовые данные
    add_test_data(cursor)
    conn.commit()
    
    while True:
        print("\n" + "="*50)
        print("МЕНЮ:")
        print("1. Показать всех пользователей")
        print("2. Добавить нового пользователя")
        print("3. Выйти")
        print("="*50)
        
        choice = input("Выберите действие (1-3): ").strip()
        
        if choice == '1':
            show_all_users(cursor)
        elif choice == '2':
            add_new_user(cursor)
            conn.commit()
        elif choice == '3':
            print("\n👋 До свидания!")
            break
        else:
            print("❌ Неверный выбор!")
    
    # Закрываем соединение
    cursor.close()
    conn.close()
    print("🔌 Соединение закрыто")

if __name__ == "__main__":
    main()
