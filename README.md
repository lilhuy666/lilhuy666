import tkinter as tk
from tkinter import messagebox
from datetime import datetime
import json, os

# ===================== STYLE =====================
BG = "#0b1220"
PANEL = "#111a2e"
CARD = "#162238"
ACCENT = "#4f8cff"
ACCENT2 = "#22c55e"
TEXT = "#e5e7eb"
SUB = "#94a3b8"
DANGER = "#ef4444"

DATA_FILE = "data.json"

# ===================== DATA =====================
data = {"users": {}}
current_user = None
selected_car = None

def load_data():
    global data
    if os.path.exists(DATA_FILE):
        try:
            with open(DATA_FILE, "r", encoding="utf-8") as f:
                data = json.load(f)
        except:
            data = {"users": {}}

def save_data():
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=2, ensure_ascii=False)

# ===================== WINDOW =====================
root = tk.Tk()
root.title("CalculatCar Pro 🚗")
root.geometry("1200x750")
root.configure(bg=BG)

root.grid_rowconfigure(0, weight=1)
root.grid_columnconfigure(1, weight=1)

# ===================== SIDEBAR =====================
sidebar = tk.Frame(root, bg=PANEL, width=260)
sidebar.grid(row=0, column=0, sticky="ns")
sidebar.grid_remove()

menu_open = False

def toggle_menu():
    global menu_open
    if menu_open:
        sidebar.grid_remove()
    else:
        sidebar.grid()
    menu_open = not menu_open

# ===================== MAIN =====================
main = tk.Frame(root, bg=BG)
main.grid(row=0, column=1, sticky="nsew")

# ===================== HEADER =====================
header = tk.Frame(main, bg=BG)
header.pack(fill="x")

tk.Button(header, text="☰",
          bg=BG, fg=TEXT,
          font=("Arial", 18),
          bd=0,
          command=toggle_menu).pack(side="left", padx=15, pady=10)

title = tk.Label(header, text="CalculatCar Pro",
                 bg=BG, fg=TEXT,
                 font=("Arial", 18, "bold"))
title.pack(side="left", pady=10)

user_label = tk.Label(header, text="",
                      bg=BG, fg=SUB,
                      font=("Arial", 12))
user_label.pack(side="right", padx=15)

# ===================== CONTENT =====================
content = tk.Frame(main, bg=BG)
content.pack(fill="both", expand=True)

def clear():
    for w in content.winfo_children():
        w.destroy()

def card():
    f = tk.Frame(content, bg=CARD, padx=40, pady=40)
    f.pack(pady=30)
    return f

# ===================== AUTH =====================
def show_auth():
    clear()
    c = card()

    tk.Label(c, text="Вход / Регистрация",
             bg=CARD, fg=TEXT,
             font=("Arial", 20, "bold")).pack(pady=10)

    email = tk.Entry(c, font=("Arial", 14))
    password = tk.Entry(c, show="*", font=("Arial", 14))

    email.pack(fill="x", pady=8)
    password.pack(fill="x", pady=8)

    def login():
        global current_user
        e = email.get().strip()
        p = password.get().strip()

        if e in data["users"] and data["users"][e]["password"] == p:
            current_user = e
            update_user()
            show_calc()
        else:
            messagebox.showerror("Ошибка", "Неверные данные")

    def register():
        e = email.get().strip()
        p = password.get().strip()

        if e in data["users"]:
            return messagebox.showerror("Ошибка", "Уже существует")

        data["users"][e] = {
            "password": p,
            "history": [],
            "cars": {}
        }
        save_data()
        messagebox.showinfo("OK", "Аккаунт создан")

    tk.Button(c, text="Войти", bg=ACCENT, fg="white",
              command=login).pack(fill="x", pady=10)

    tk.Button(c, text="Регистрация", bg=ACCENT2,
              command=register).pack(fill="x")

# ===================== PROFILE =====================
def show_profile():
    global selected_car

    if not current_user:
        show_auth()
        return

    clear()
    c = card()
    user = data["users"][current_user]

    tk.Label(c, text="Профиль",
             bg=CARD, fg=TEXT,
             font=("Arial", 20, "bold")).pack()

    # авто список
    tk.Label(c, text="Ваши авто:", bg=CARD, fg=SUB).pack(anchor="w")

    car_var = tk.StringVar(value=selected_car)

    for car, info in user["cars"].items():
        txt = f"{car} | {info.get('cons','—')} л/100км"
        tk.Radiobutton(c, text=txt, variable=car_var,
                       value=car, bg=CARD, fg=TEXT,
                       selectcolor=ACCENT).pack(anchor="w")

    def set_car():
        global selected_car
        selected_car = car_var.get()

    tk.Button(c, text="Выбрать авто",
              bg=ACCENT,
              command=set_car).pack(fill="x", pady=5)

    # добавить авто
    new_car = tk.Entry(c)
    new_car.pack(fill="x", pady=10)

    def add_car():
        name = new_car.get().strip()
        if not name:
            return
        user["cars"][name] = {"cons": None}
        save_data()
        show_profile()

    tk.Button(c, text="Добавить авто",
              bg=ACCENT2,
              command=add_car).pack(fill="x")

    # пароль
    new_pass = tk.Entry(c, show="*")
    new_pass.pack(fill="x", pady=10)

    def change_pass():
        if new_pass.get():
            user["password"] = new_pass.get()
            save_data()
            messagebox.showinfo("OK", "Пароль изменён")

    tk.Button(c, text="Сменить пароль",
              command=change_pass).pack(fill="x")

    def logout():
        global current_user
        current_user = None
        update_user()
        show_calc()

    tk.Button(c, text="Выйти",
              bg=DANGER,
              fg="white",
              command=logout).pack(fill="x", pady=10)

# ===================== CALC =====================
def show_calc():
    clear()

    tk.Label(content, text="Калькулятор",
             bg=BG, fg=TEXT,
             font=("Arial", 20)).pack(pady=10)

    mode = tk.StringVar(value="1")

    form = tk.Frame(content, bg=CARD, padx=30, pady=30)
    form.pack()

    entries = {}

    def add(t):
        tk.Label(form, text=t, bg=CARD, fg=TEXT).pack(anchor="w")
        e = tk.Entry(form)
        e.pack(fill="x")
        entries[t] = e

    def build():
        for w in form.winfo_children():
            w.destroy()
        entries.clear()

        if mode.get() == "1":
            add("Топливо (л)")
            add("Расстояние (км)")
            add("Цена за литр")
        else:
            add("Средний расход (л/100км)")
            add("Расстояние (км)")
            add("Цена за литр")

    def calc():
        try:
            if mode.get() == "1":
                f = float(entries["Топливо (л)"].get())
                d = float(entries["Расстояние (км)"].get())
                p = float(entries["Цена за литр"].get())
                cons = (f / d) * 100
            else:
                cons = float(entries["Средний расход (л/100км)"].get())
                d = float(entries["Расстояние (км)"].get())
                p = float(entries["Цена за литр"].get())
                f = cons * d / 100

            cost = f * p
            result.config(text=f"{cons:.1f} л/100км | {cost:.0f} ₽")

            if current_user:
                user = data["users"][current_user]

                # история
                user["history"].append({
                    "date": datetime.now().strftime("%d.%m %H:%M"),
                    "result": result.cget("text")
                })

                # сохранить расход в авто
                if selected_car and selected_car in user["cars"]:
                    user["cars"][selected_car]["cons"] = round(cons, 1)

                save_data()

        except:
            messagebox.showerror("Ошибка", "Проверь ввод")

    result = tk.Label(content, text="—",
                      bg=BG, fg=ACCENT,
                      font=("Arial", 20))
    result.pack()

    tk.Button(content, text="Рассчитать",
              bg=ACCENT,
              command=calc).pack(pady=10)

    mode.trace("w", lambda *a: build())
    build()

# ===================== HISTORY =====================
def show_history():
    clear()
    tk.Label(content, text="История",
             bg=BG, fg=TEXT,
             font=("Arial", 20)).pack()

    if not current_user:
        return

    for h in data["users"][current_user]["history"][::-1]:
        tk.Label(content,
                 text=f"{h['date']} | {h['result']}",
                 bg=CARD, fg=TEXT).pack(fill="x", padx=20, pady=5)

# ===================== SETTINGS =====================
def show_settings():
    clear()
    tk.Label(content, text="Настройки",
             bg=BG, fg=TEXT,
             font=("Arial", 20)).pack(pady=50)

# ===================== ABOUT =====================
def show_about():
    clear()
    tk.Label(content,
             text="CalculatCar Pro 🚗\nЛучший калькулятор топлива",
             bg=BG, fg=TEXT,
             font=("Arial", 16)).pack(pady=60)

# ===================== MENU =====================
def nav(text, cmd):
    tk.Button(sidebar, text=text,
              bg=PANEL, fg=TEXT,
              bd=0, anchor="w",
              command=cmd).pack(fill="x", pady=2)

nav("Калькулятор", show_calc)
nav("Профиль", show_profile)
nav("История", show_history)
nav("Настройки", show_settings)
nav("О программе", show_about)

# ===================== USER =====================
def update_user():
    user_label.config(text=current_user if current_user else "")

# ===================== START =====================
load_data()
show_calc()
root.mainloop()
