Эта ошибка возникает из-за несовместимости версий MySQL и драйвера Python. Вот **3 способа решения** — от самого простого до самого правильного:

---

## 🔥 СПОСОБ 1 (Самый быстрый — за 1 минуту)

### Удали старую библиотеку и установи другую:

```bash
pip uninstall mysql-connector-python -y
pip install pymysql
```

### Обновлённый код `app_mysql.py`:

```python
import pymysql
from tabulate import tabulate

# Настройки подключения (замени на свои!)
config = {
    'host': 'localhost',
    'user': 'root',         # твой пользователь
    'password': '12345',    # твой пароль
    'database': 'mydb',     # имя базы данных
    'charset': 'utf8mb4'
}

try:
    # Подключаемся через PyMySQL
    conn = pymysql.connect(**config)
    cursor = conn.cursor()
    print("✅ Подключено к MySQL через PyMySQL\n")

    # Выполняем запрос
    cursor.execute("SELECT * FROM users")
    rows = cursor.fetchall()

    # Красиво выводим
    print(tabulate(rows, headers=["ID", "Имя", "Телефон"], tablefmt="grid"))

    cursor.close()
    conn.close()
    print("\n🔌 Соединение закрыто")

except Exception as err:
    print(f"❌ Ошибка: {err}")
```

**PyMySQL** автоматически поддерживает `caching_sha2_password` и работает без проблем.

---

## 🔧 СПОСОБ 2 (Исправить драйвер MySQL)

Оставь `mysql-connector-python`, но установи дополнительный пакет:

```bash
pip install mysql-connector-python dnspython
```

И в коде добавь параметр `auth_plugin`:

```python
import mysql.connector
from tabulate import tabulate

config = {
    'host': 'localhost',
    'user': 'root',
    'password': '12345',
    'database': 'mydb',
    'auth_plugin': 'mysql_native_password'  # ← вот эта строка!
}

try:
    conn = mysql.connector.connect(**config)
    cursor = conn.cursor()
    print("✅ Подключено к MySQL\n")

    cursor.execute("SELECT * FROM users")
    rows = cursor.fetchall()

    print(tabulate(rows, headers=["ID", "Имя", "Телефон"], tablefmt="grid"))

    cursor.close()
    conn.close()

except mysql.connector.Error as err:
    print(f"❌ Ошибка: {err}")
```

---

## ⚙️ СПОСОБ 3 (Изменить настройки MySQL — навсегда)

Открой **MySQL Workbench** и выполни этот запрос (замени `'12345'` на свой пароль):

```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '12345';
FLUSH PRIVILEGES;
```

После этого можешь использовать любой код с `mysql-connector-python` без ошибок.

---

## 🚀 ГОТОВЫЙ РАБОЧИЙ КОД (Прямо сейчас скопируй и вставь)

Этот код точно запустится:

```python
import pymysql
from tabulate import tabulate

# ===== НАСТРОЙКИ (поменяй на свои) =====
DB_HOST = "localhost"
DB_USER = "root"
DB_PASSWORD = "12345"     # ← твой пароль MySQL
DB_NAME = "mydb"
# =========================================

try:
    # Подключение
    conn = pymysql.connect(
        host=DB_HOST,
        user=DB_USER,
        password=DB_PASSWORD,
        database=DB_NAME,
        charset='utf8mb4',
        cursorclass=pymysql.cursors.Cursor
    )
    
    cursor = conn.cursor()
    print("✅ Успешное подключение к MySQL!\n")
    
    # Создаём таблицу (если нет)
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id INT AUTO_INCREMENT PRIMARY KEY,
            name VARCHAR(100) NOT NULL,
            phone VARCHAR(20)
        )
    """)
    
    # Добавляем тестовые данные (если таблица пуста)
    cursor.execute("SELECT COUNT(*) FROM users")
    if cursor.fetchone()[0] == 0:
        cursor.execute("INSERT INTO users (name, phone) VALUES ('Иван', '123-456')")
        cursor.execute("INSERT INTO users (name, phone) VALUES ('Мария', '789-012')")
        conn.commit()
        print("📝 Добавлены тестовые данные\n")
    
    # Выводим все записи
    cursor.execute("SELECT * FROM users")
    rows = cursor.fetchall()
    
    print(tabulate(rows, headers=["ID", "Имя", "Телефон"], tablefmt="grid"))
    
    # Закрываем
    cursor.close()
    conn.close()
    print("\n🔌 Соединение закрыто")

except Exception as e:
    print(f"❌ Ошибка: {e}")
    print("\nПроверь:")
    print("1. MySQL запущен?")
    print("2. Пароль правильный?")
    print("3. База данных 'mydb' существует?")
```

---

## 📋 Проверка перед запуском:

1. **MySQL Workbench должен быть запущен** (зелёный значок в трее или запусти MySQL сервер)
2. База данных `mydb` должна существовать (создай через Workbench если нет)
3. Пароль в коде должен совпадать с твоим

Напиши, получилось ли запустить! 👍
