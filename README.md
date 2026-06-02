import tkinter as tk
from tkinter import messagebox
import mysql.connector

try:
    conn = mysql.connector.connect(
        host="localhost",
        user="root",
        password="1234",   # <-- ВСТАВЬ СВОЙ ПАРОЛЬ MYSQL
        database="shop_db"
    )

    cursor = conn.cursor(dictionary=True)

except mysql.connector.Error as err:
    print("Ошибка подключения:", err)
    exit()

def show_products():

    products_window = tk.Toplevel(root)
    products_window.title("Список товаров")
    products_window.geometry("500x300")

    try:
        cursor.execute("SELECT * FROM Products")
        products = cursor.fetchall()

        for idx, product in enumerate(products):

            text = f"{product['name']} | Цена: {product['price']} руб | Остаток: {product['count']}"

            tk.Label(
                products_window,
                text=text,
                font=("Arial", 12)
            ).pack(anchor="w", padx=10, pady=5)

    except mysql.connector.Error as err:
        messagebox.showerror("Ошибка", str(err))

def login():

    user = entry_user.get()
    pwd = entry_pass.get()

    try:

        query = "SELECT * FROM Users WHERE login=%s AND password=%s"

        cursor.execute(query, (user, pwd))

        result = cursor.fetchone()

        if result:

            role = result['role']

            messagebox.showinfo(
                "Успех",
                f"Вход выполнен как: {role}"
            )

            show_products()

        else:
            messagebox.showerror(
                "Ошибка",
                "Неверный логин или пароль"
            )

    except mysql.connector.Error as err:
        messagebox.showerror("Ошибка БД", str(err))

root = tk.Tk()

root.title("Авторизация")
root.geometry("300x200")

# Логин
tk.Label(root, text="Логин").pack(pady=5)

entry_user = tk.Entry(root)
entry_user.pack()

# Пароль
tk.Label(root, text="Пароль").pack(pady=5)

entry_pass = tk.Entry(root, show="*")
entry_pass.pack()

# Кнопка входа
tk.Button(
    root,
    text="Войти",
    command=login
).pack(pady=10)

# Гостевой вход
tk.Button(
    root,
    text="Войти как гость",
    command=show_products
).pack()

# Запуск приложения
root.mainloop()




DROP DATABASE IF EXISTS shop_db;
CREATE DATABASE shop_db;

USE shop_db;

CREATE TABLE Users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    login VARCHAR(50),
    password VARCHAR(50),
    role VARCHAR(20)
);

INSERT INTO Users(login,password,role)
VALUES
('admin','123','admin'),
('client','123','client');

CREATE TABLE Products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2),
    count INT
);

INSERT INTO Products(name,price,count)
VALUES
('Кроссовки',2500,10),
('Ботинки',4000,5),
('Сандалии',1500,20);


