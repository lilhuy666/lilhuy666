Понял, тебе нужен минимальный, но полностью рабочий код для Visual Studio Code с базой данных. Дам два варианта: с **SQLite** (вообще без установки MySQL) и с **MySQL**.

---

## ВАРИАНТ 1 (SQLite — самый простой, не нужно ничего устанавливать)

### Шаг 1. Установка
Открой терминал в VS Code (`Ctrl+J`) и выполни:
```bash
pip install tabulate
```
*(SQLite уже встроен в Python, ничего дополнительно не нужно)*

### Шаг 2. Код (один файл `app.py`)

Создай файл **`app.py`** и вставь этот код:

```python
import sqlite3
from tabulate import tabulate

# Создаём или подключаемся к базе данных
conn = sqlite3.connect('mydatabase.db')
cursor = conn.cursor()

# Создаём таблицу
cursor.execute('''
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    phone TEXT
)
''')
conn.commit()
print("База данных и таблица готовы\n")

# Вставляем тестовые данные, если таблица пуста
cursor.execute("SELECT COUNT(*) FROM users")
if cursor.fetchone()[0] == 0:
    cursor.execute("INSERT INTO users (name, phone) VALUES ('Иван', '123-456')")
    cursor.execute("INSERT INTO users (name, phone) VALUES ('Мария', '789-012')")
    conn.commit()

# Выводим данные
cursor.execute("SELECT * FROM users")
rows = cursor.fetchall()
print(tabulate(rows, headers=["ID", "Имя", "Телефон"], tablefmt="grid"))

# Закрываем соединение
conn.close()
```

### Шаг 3. Запуск
Нажми **зелёный треугольник ▶️** в VS Code (справа вверху) или `F5`.

**Результат:**
```
База данных и таблица готовы

+----+--------+----------+
| ID | Имя    | Телефон  |
+====+========+==========+
| 1  | Иван   | 123-456  |
+----+--------+----------+
| 2  | Мария  | 789-012  |
+----+--------+----------+
```

---

## ВАРИАНТ 2 (MySQL — если у тебя уже установлен MySQL)

### Шаг 1. Установка библиотеки
В терминале VS Code:
```bash
pip install mysql-connector-python tabulate
```

### Шаг 2. Создай базу в MySQL Workbench
Открой MySQL Workbench и выполни запрос:
```sql
CREATE DATABASE IF NOT EXISTS mydb;
USE mydb;

CREATE TABLE IF NOT EXISTS users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    phone VARCHAR(20)
);

INSERT INTO users (name, phone) VALUES 
('Иван', '123-456'),
('Мария', '789-012');
```

### Шаг 3. Код `app_mysql.py`

```python
import mysql.connector
from tabulate import tabulate

# Настройки подключения (ЗАМЕНИ НА СВОИ!)
config = {
    'host': 'localhost',      # или 127.0.0.1
    'user': 'root',           # твой пользователь MySQL
    'password': '12345',      # твой пароль MySQL
    'database': 'mydb'        # имя базы данных
}

try:
    # Подключаемся
    conn = mysql.connector.connect(**config)
    cursor = conn.cursor()
    print("✅ Подключено к MySQL\n")

    # Выполняем запрос
    cursor.execute("SELECT * FROM users")
    rows = cursor.fetchall()

    # Красиво выводим
    print(tabulate(rows, headers=["ID", "Имя", "Телефон"], tablefmt="grid"))

    cursor.close()
    conn.close()
    print("\nСоединение закрыто")

except mysql.connector.Error as err:
    print(f"❌ Ошибка: {err}")
```

### Шаг 4. Запуск
Нажми **▶️** в VS Code.

---

## Как запускать в VS Code (коротко)
1. Создай файл с расширением `.py`
2. Нажми `Ctrl+S` (сохранить)
3. Нажми `F5` или зелёный треугольник ▶️ вверху справа
4. Результат появится в терминале внизу

---

Если нужен код с **возможностью ввода данных** с клавиатуры, напиши — допишу 👍
