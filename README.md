import tkinter as tk
from tkinter import messagebox, ttk
import sqlite3
import matplotlib.pyplot as plt

# ---------------- БАЗА ----------------
conn = sqlite3.connect("fuel_app.db")
cursor = conn.cursor()

cursor.execute("""
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT,
    password TEXT
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS history (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user TEXT,
    transport TEXT,
    distance REAL,
    fuel REAL,
    price REAL,
    result REAL,
    cost REAL
)
""")

conn.commit()

current_user = None

# ---------------- СТИЛЬ ----------------
def setup_style():
    style = ttk.Style()
    style.theme_use("clam")

    style.configure("TButton",
                    font=("Segoe UI", 10),
                    padding=6)

    style.configure("TLabel",
                    font=("Segoe UI", 10))

    style.configure("Header.TLabel",
                    font=("Segoe UI", 14, "bold"),
                    foreground="#1b5e20")


# ---------------- АВТОРИЗАЦИЯ ----------------
def register():
    username = entry_login.get()
    password = entry_password.get()

    cursor.execute("SELECT * FROM users WHERE username=?", (username,))
    if cursor.fetchone():
        messagebox.showerror("Ошибка", "Пользователь уже существует")
        return

    cursor.execute("INSERT INTO users VALUES (NULL, ?, ?)", (username, password))
    conn.commit()
    messagebox.showinfo("Успех", "Регистрация успешна")


def login():
    global current_user
    username = entry_login.get()
    password = entry_password.get()

    cursor.execute("SELECT * FROM users WHERE username=? AND password=?", (username, password))
    if cursor.fetchone():
        current_user = username
        open_main()
    else:
        messagebox.showerror("Ошибка", "Неверные данные")


# ---------------- РАСЧЕТ ----------------
def calculate():
    try:
        distance = float(entry_distance.get())
        fuel = float(entry_fuel.get())
        price = float(entry_price.get())
        transport = transport_var.get()

        consumption = (fuel / distance) * 100
        cost = fuel * price

        result_label.config(
            text=f"{consumption:.2f} л/100 км | {cost:.2f} руб."
        )

        cursor.execute("""
        INSERT INTO history VALUES (NULL, ?, ?, ?, ?, ?, ?, ?)
        """, (current_user, transport, distance, fuel, price, consumption, cost))
        conn.commit()

    except:
        messagebox.showerror("Ошибка", "Проверьте ввод")


# ---------------- АНАЛИТИКА ----------------
def show_analytics():
    cursor.execute("SELECT result FROM history WHERE user=?", (current_user,))
    data = cursor.fetchall()

    if not data:
        messagebox.showinfo("Нет данных", "Нет расчетов")
        return

    values = [x[0] for x in data]
    avg = sum(values) / len(values)

    messagebox.showinfo("Аналитика", f"Средний расход: {avg:.2f} л/100 км")

    # График
    plt.figure()
    plt.plot(values)
    plt.title("История расхода топлива")
    plt.xlabel("Поездки")
    plt.ylabel("л/100 км")
    plt.show()


# ---------------- ИСТОРИЯ ----------------
def show_history():
    win = tk.Toplevel()
    win.title("История")

    tree = ttk.Treeview(win, columns=("Тип", "Расход", "Стоимость"), show="headings")
    tree.heading("Тип", text="Транспорт")
    tree.heading("Расход", text="л/100 км")
    tree.heading("Стоимость", text="руб")

    tree.pack(fill="both", expand=True)

    cursor.execute("SELECT transport, result, cost FROM history WHERE user=?", (current_user,))
    for row in cursor.fetchall():
        tree.insert("", tk.END, values=row)


# ---------------- ГЛАВНОЕ ОКНО ----------------
def open_main():
    login_window.destroy()

    app = tk.Tk()
    app.title("Fuel Analytics")
    app.geometry("420x420")
    app.configure(bg="#e8f5e9")

    setup_style()

    frame = tk.Frame(app, bg="#e8f5e9")
    frame.pack(pady=10)

    ttk.Label(frame, text=f"Пользователь: {current_user}", style="Header.TLabel").pack(pady=5)

    ttk.Label(frame, text="Тип транспорта").pack()
    global transport_var
    transport_var = ttk.Combobox(frame, values=["Легковой", "Грузовой", "Мотоцикл"])
    transport_var.pack()

    ttk.Label(frame, text="Расстояние (км)").pack()
    global entry_distance
    entry_distance = ttk.Entry(frame)
    entry_distance.pack()

    ttk.Label(frame, text="Топливо (л)").pack()
    global entry_fuel
    entry_fuel = ttk.Entry(frame)
    entry_fuel.pack()

    ttk.Label(frame, text="Цена (руб)").pack()
    global entry_price
    entry_price = ttk.Entry(frame)
    entry_price.pack()

    ttk.Button(frame, text="Рассчитать", command=calculate).pack(pady=5)
    ttk.Button(frame, text="История", command=show_history).pack()
    ttk.Button(frame, text="Аналитика", command=show_analytics).pack(pady=5)

    global result_label
    result_label = ttk.Label(frame, text="", font=("Segoe UI", 11))
    result_label.pack(pady=10)

    app.mainloop()


# ---------------- ОКНО ВХОДА ----------------
login_window = tk.Tk()
login_window.title("Вход")
login_window.geometry("300x250")
login_window.configure(bg="#c8e6c9")

setup_style()

frame = tk.Frame(login_window, bg="#c8e6c9")
frame.pack(pady=20)

ttk.Label(frame, text="Вход в систему", style="Header.TLabel").pack(pady=10)

entry_login = ttk.Entry(frame)
entry_login.pack()
entry_password = ttk.Entry(frame, show="*")
entry_password.pack()

ttk.Button(frame, text="Войти", command=login).pack(pady=5)
ttk.Button(frame, text="Регистрация", command=register).pack()

login_window.mainloop()
