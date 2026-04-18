import tkinter as tk
from tkinter import messagebox, ttk
import json
from datetime import datetime
import re
import matplotlib.pyplot as plt

# =====================
# ДАННЫЕ
# =====================
users = {}
current_user = None
current_car = None

# =====================
# ФАЙЛ
# =====================
def load_data():
    global users
    try:
        with open("data.json", "r", encoding="utf-8") as f:
            users = json.load(f)
    except:
        users = {}

def save_data():
    with open("data.json", "w", encoding="utf-8") as f:
        json.dump(users, f, ensure_ascii=False)

# =====================
# UI
# =====================
root = tk.Tk()
root.title("FuelCalcPro")
root.geometry("1200x800")

# =====================
# HEADER
# =====================
header = tk.Frame(root, bg="#064e3b", height=70)
header.pack(fill="x")

menu_frame = tk.Frame(root, bg="#064e3b")
menu_open = False

def toggle_menu():
    global menu_open
    if menu_open:
        menu_frame.place_forget()
    else:
        menu_frame.place(x=0, y=70, width=250, height=730)
    menu_open = not menu_open

tk.Button(header, text="☰", command=toggle_menu,
          bg="#064e3b", fg="white", font=("Arial", 18), bd=0).pack(side="left", padx=10)

tk.Label(header, text="⛽ FuelCalcPro",
         bg="#064e3b", fg="white",
         font=("Arial", 24, "bold")).pack(pady=10)

main = tk.Frame(root)
main.pack(fill="both", expand=True)

def clear():
    for w in main.winfo_children():
        w.destroy()

# =====================
# КАЛЬКУЛЯТОР
# =====================
def show_calc():
    clear()

    if not current_user:
        messagebox.showinfo("Инфо", "Для выбора авто зайдите в профиль")

    tk.Label(main, text="Калькулятор", font=("Arial", 22)).pack(pady=10)

    # выбор авто
    car_var = tk.StringVar()
    cars = []
    if current_user:
        cars = list(users[current_user]["cars"].keys())

    ttk.Combobox(main, textvariable=car_var, values=cars).pack(pady=10)

    mode = tk.StringVar(value="1")

    tk.Radiobutton(main, text="Средний расход (л/100км)",
                   variable=mode, value="1").pack()

    tk.Radiobutton(main, text="Расход и стоимость",
                   variable=mode, value="2").pack()

    frame = tk.Frame(main)
    frame.pack(pady=20)

    entries = {}

    def add_field(name):
        tk.Label(frame, text=name).pack()
        e = tk.Entry(frame)
        e.pack()
        entries[name] = e

    add_field("Топливо (л)")
    add_field("Расстояние (км)")
    add_field("Цена за литр")

    currency = tk.StringVar(value="₽")
    ttk.Combobox(main, textvariable=currency,
                 values=["₽", "$", "€"]).pack()

    def calc():
        try:
            fuel = float(entries["Топливо (л)"].get())
            dist = float(entries["Расстояние (км)"].get())
            price = float(entries["Цена за литр"].get())

            cons = (fuel / dist) * 100
            cost = fuel * price

            if current_user and car_var.get():
                car = car_var.get()
                users[current_user]["cars"][car]["consumption"] = cons
                users[current_user]["cars"][car]["history"].append(cons)
                save_data()

            messagebox.showinfo("Результат",
                                f"{cons:.2f} л/100км\n{cost:.2f} {currency.get()}")

        except:
            messagebox.showerror("Ошибка", "Проверь ввод")

    tk.Button(main, text="Рассчитать", command=calc).pack(pady=20)

# =====================
# ПРОФИЛЬ
# =====================
def show_profile():
    clear()

    if not current_user:
        show_auth()
        return

    tk.Label(main, text="Профиль", font=("Arial", 22)).pack(pady=10)

    cars = users[current_user]["cars"]

    for car, data in cars.items():
        tk.Label(main, text=f"{car} | {data.get('consumption', 0):.2f} л/100км").pack()

    new_car = tk.Entry(main)
    new_car.pack(pady=10)

    def add_car():
        cars[new_car.get()] = {"consumption": 0, "history": []}
        save_data()
        show_profile()

    tk.Button(main, text="Добавить авто", command=add_car).pack()

    def logout():
        global current_user
        current_user = None
        show_calc()

    tk.Button(main, text="Выйти", command=logout).pack(pady=20)

# =====================
# AUTH
# =====================
def show_auth():
    clear()

    tk.Label(main, text="Вход", font=("Arial", 22)).pack(pady=10)

    email = tk.Entry(main)
    password = tk.Entry(main, show="*")

    email.pack(pady=10)
    password.pack(pady=10)

    def valid_email(e):
        return re.match(r"[^@]+@[^@]+\.[^@]+", e)

    def login():
        global current_user
        if not valid_email(email.get()):
            messagebox.showerror("Ошибка", "Введите email")
            return

        if email.get() in users and users[email.get()]["password"] == password.get():
            current_user = email.get()
            show_profile()
        else:
            messagebox.showerror("Ошибка", "Неверно")

    def register():
        if not valid_email(email.get()):
            messagebox.showerror("Ошибка", "Введите email")
            return

        users[email.get()] = {
            "password": password.get(),
            "cars": {}
        }
        save_data()
        messagebox.showinfo("OK", "Создан")

    tk.Button(main, text="Войти", command=login).pack()
    tk.Button(main, text="Регистрация", command=register).pack()

# =====================
# ГРАФИК
# =====================
def show_history():
    clear()

    if not current_user:
        return

    cars = users[current_user]["cars"]

    for car, data in cars.items():
        if data["history"]:
            plt.plot(data["history"], label=car)

    plt.legend()
    plt.title("Расход топлива")
    plt.show()

# =====================
# ПРОЧЕЕ
# =====================
def show_about():
    clear()
    tk.Label(main, text="FuelCalcPro — сервис учета топлива").pack()

def show_fuel():
    clear()
    tk.Label(main, text="Заправки: Лукойл, Роснефть, Газпром").pack()

# =====================
# МЕНЮ
# =====================
def btn(text, cmd):
    tk.Button(menu_frame, text=text, command=cmd).pack(fill="x", pady=5)

btn("Калькулятор", show_calc)
btn("Профиль", show_profile)
btn("История", show_history)
btn("Заправки", show_fuel)
btn("О нас", show_about)

# =====================
# СТАРТ
# =====================
load_data()
show_calc()
root.mainloop()
