import tkinter as tk
from tkinter import messagebox
from datetime import datetime
import json

COLORS = {
    "bg": "#eef2f3",
    "card": "#ffffff",
    "accent": "#3b82f6",
    "text": "#111827"
}

users = {}
current_user = None
history = []

def load_data():
    global users
    try:
        with open("data.json", "r", encoding="utf-8") as f:
            users = json.load(f)
    except:
        users = {}

def save_data():
    with open("data.json", "w", encoding="utf-8") as f:
        json.dump(users, f, ensure_ascii=False, indent=2)

root = tk.Tk()
root.geometry("900x700")
root.title("FuelCalcPro")
root.configure(bg=COLORS["bg"])

main_frame = tk.Frame(root, bg=COLORS["bg"])
main_frame.pack(fill="both", expand=True)

def clear():
    for w in main_frame.winfo_children():
        w.destroy()

# ===============================
# КАЛЬКУЛЯТОР (НОВЫЙ UI)
# ===============================
def show_calc():
    clear()

    tk.Label(main_frame, text="Калькулятор расхода топлива",
             bg=COLORS["bg"], fg=COLORS["text"],
             font=("Arial", 22, "bold")).pack(pady=20)

    if not current_user:
        tk.Label(main_frame, text="Войдите в профиль",
                 bg=COLORS["bg"]).pack()
        return

    user = users[current_user]

    # карточка
    card = tk.Frame(main_frame, bg=COLORS["card"], padx=20, pady=20)
    card.pack(pady=10, ipadx=10, ipady=10)

    mode = tk.StringVar(value="avg")

    def rebuild():
        for w in fields_frame.winfo_children():
            w.destroy()

        if mode.get() == "avg":
            build_avg()
        else:
            build_cost()

    # переключатели
    tk.Radiobutton(card, text="Рассчитать расход и стоимость",
                   variable=mode, value="cost",
                   bg=COLORS["card"], command=rebuild).pack(anchor="w")

    tk.Radiobutton(card, text="Рассчитать средний расход на 100 км",
                   variable=mode, value="avg",
                   bg=COLORS["card"], command=rebuild).pack(anchor="w")

    fields_frame = tk.Frame(card, bg=COLORS["card"])
    fields_frame.pack(pady=10)

    currency = tk.StringVar(value="₽")

    def entry(label, unit=""):
        f = tk.Frame(fields_frame, bg=COLORS["card"])
        f.pack(pady=5, anchor="w")

        tk.Label(f, text=label,
                 bg=COLORS["card"]).pack(anchor="w")

        row = tk.Frame(f, bg=COLORS["card"])
        row.pack()

        e = tk.Entry(row, width=20)
        e.pack(side="left")

        tk.Label(row, text=unit, bg=COLORS["card"]).pack(side="left", padx=5)

        return e

    def build_avg():
        nonlocal fuel_e, dist_e, price_e
        fuel_e = entry("Израсходовано топлива", "литров")
        dist_e = entry("Пройденное расстояние", "км")
        price_e = entry("Цена за литр")

        tk.OptionMenu(fields_frame, currency, "₽", "$", "€").pack(pady=5)

    def build_cost():
        nonlocal cons_e, dist_e, price_e, unit
        cons_e = entry("Средний расход топлива")
        unit = tk.StringVar(value="л/100км")
        tk.OptionMenu(fields_frame, unit, "л/100км", "км/л").pack()

        dist_e = entry("Расстояние", "км")
        price_e = entry("Цена за литр")

        tk.OptionMenu(fields_frame, currency, "₽", "$", "€").pack(pady=5)

    fuel_e = dist_e = price_e = cons_e = None
    unit = tk.StringVar()

    build_avg()

    def calc():
        try:
            if mode.get() == "avg":
                f = float(fuel_e.get())
                d = float(dist_e.get())
                p = float(price_e.get())

                consumption = (f / d) * 100
                cost = f * p

            else:
                d = float(dist_e.get())
                p = float(price_e.get())
                c = float(cons_e.get())

                if unit.get() == "л/100км":
                    f = (d / 100) * c
                    consumption = c
                else:
                    f = d / c
                    consumption = 100 / c

                cost = f * p

            messagebox.showinfo("Результат",
                                f"{consumption:.2f} л/100км\n"
                                f"{cost:.2f} {currency.get()}")

        except:
            messagebox.showerror("Ошибка", "Проверь ввод")

    tk.Button(main_frame, text="РАССЧИТАТЬ",
              command=calc,
              bg=COLORS["accent"], fg="white",
              font=("Arial", 14), padx=20, pady=10).pack(pady=20)

# ===============================
# ПРОФИЛЬ (упрощённый)
# ===============================
def show_profile():
    clear()

    if not current_user:
        show_auth()
        return

    user = users[current_user]

    tk.Label(main_frame, text="Профиль",
             bg=COLORS["bg"], font=("Arial", 22)).pack(pady=20)

    name = tk.Entry(main_frame, font=("Arial", 16))
    name.insert(0, user.get("name", ""))
    name.pack(pady=10)

    car = tk.Entry(main_frame)
    car.pack()

    def add():
        user.setdefault("cars", []).append({"name": car.get()})
        save_data()
        show_profile()

    tk.Button(main_frame, text="Добавить авто", command=add).pack()

    for c in user.get("cars", []):
        tk.Label(main_frame, text=c["name"], bg=COLORS["bg"]).pack()

# ===============================
# АВТОРИЗАЦИЯ
# ===============================
def show_auth():
    clear()

    tk.Label(main_frame, text="Вход",
             font=("Arial", 24),
             bg=COLORS["bg"]).pack(pady=20)

    login = tk.Entry(main_frame, font=("Arial", 16))
    password = tk.Entry(main_frame, show="*", font=("Arial", 16))

    login.pack(pady=10)
    password.pack(pady=10)

    def login_user():
        global current_user
        if login.get() in users and users[login.get()]["password"] == password.get():
            current_user = login.get()
            show_calc()
        else:
            messagebox.showerror("Ошибка", "Неверно")

    def register():
        users[login.get()] = {"password": password.get()}
        save_data()
        messagebox.showinfo("OK", "Создан")

    tk.Button(main_frame, text="Войти", command=login_user).pack(pady=10)
    tk.Button(main_frame, text="Регистрация", command=register).pack()

# ===============================
# СТАРТ
# ===============================
load_data()
show_auth()
root.mainloop()
