import tkinter as tk
from tkinter import messagebox, ttk
import json
from datetime import datetime
import re

# =====================
# ДАННЫЕ
# =====================
users = {}
current_user = None

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
          bg="#064e3b", fg="white", font=("Arial", 18), bd=0).pack(side="left")

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

    tk.Label(main, text="Калькулятор", font=("Arial", 22)).pack(pady=10)

    car_var = tk.StringVar()

    if current_user:
        cars = list(users[current_user]["cars"].keys())
    else:
        cars = []

    ttk.Combobox(main, textvariable=car_var, values=cars).pack(pady=10)

    fuel = tk.Entry(main)
    dist = tk.Entry(main)
    price = tk.Entry(main)

    for text, field in [("Топливо (л)", fuel), ("Расстояние (км)", dist), ("Цена", price)]:
        tk.Label(main, text=text).pack()
        field.pack()

    def calc():
        try:
            f = float(fuel.get())
            d = float(dist.get())
            p = float(price.get())

            cons = (f / d) * 100

            if current_user and car_var.get():
                car = car_var.get()
                users[current_user]["cars"][car]["history"].append(cons)
                users[current_user]["cars"][car]["consumption"] = cons
                save_data()

            messagebox.showinfo("Результат", f"{cons:.2f} л/100км\n{f*p:.2f}")

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

    tk.Label(main, text="Профиль", font=("Arial", 22)).pack()

    cars = users[current_user]["cars"]

    for car in cars:
        tk.Label(main, text=car).pack()

    new_car = tk.Entry(main)
    new_car.pack()

    def add_car():
        cars[new_car.get()] = {"history": [], "consumption": 0}
        save_data()
        show_profile()

    tk.Button(main, text="Добавить авто", command=add_car).pack()

    def logout():
        global current_user
        current_user = None
        show_calc()

    tk.Button(main, text="Выйти", command=logout).pack()

# =====================
# AUTH
# =====================
def show_auth():
    clear()

    tk.Label(main, text="Вход", font=("Arial", 22)).pack()

    email = tk.Entry(main)
    password = tk.Entry(main, show="*")

    email.pack()
    password.pack()

    def valid(e):
        return re.match(r"[^@]+@[^@]+\.[^@]+", e)

    def login():
        global current_user
        if not valid(email.get()):
            messagebox.showerror("Ошибка", "Введите email")
            return

        if email.get() in users and users[email.get()]["password"] == password.get():
            current_user = email.get()
            show_profile()
        else:
            messagebox.showerror("Ошибка", "Ошибка входа")

    def reg():
        users[email.get()] = {"password": password.get(), "cars": {}}
        save_data()

    tk.Button(main, text="Войти", command=login).pack()
    tk.Button(main, text="Регистрация", command=reg).pack()

# =====================
# ГРАФИК (Canvas)
# =====================
def show_history():
    clear()

    canvas = tk.Canvas(main, bg="white", width=800, height=400)
    canvas.pack(pady=20)

    if not current_user:
        return

    cars = users[current_user]["cars"]

    colors = ["red", "blue", "green", "orange"]

    for i, (car, data) in enumerate(cars.items()):
        points = data["history"]

        for j in range(len(points)-1):
            x1 = j * 50 + 50
            y1 = 300 - points[j]*5
            x2 = (j+1)*50 + 50
            y2 = 300 - points[j+1]*5

            canvas.create_line(x1, y1, x2, y2, fill=colors[i%4], width=2)

        canvas.create_text(700, 20 + i*20, text=car, fill=colors[i%4])

# =====================
# ПРОЧЕЕ
# =====================
def show_about():
    clear()
    tk.Label(main, text="FuelCalcPro — приложение для учета топлива").pack()

def show_fuel():
    clear()
    tk.Label(main, text="Заправки: Лукойл, Роснефть, Газпром").pack()

# =====================
# МЕНЮ
# =====================
def btn(t, c):
    tk.Button(menu_frame, text=t, command=c).pack(fill="x")

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
