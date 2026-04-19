import tkinter as tk
from tkinter import messagebox
from datetime import datetime
import json, os

# ===================== STYLE (LARGE UI) =====================
BG = "#0b1220"
PANEL = "#111a2e"
SIDEBAR = "#162238"
CARD = "#1b2a41"
ACCENT = "#4f8cff"
GREEN = "#22c55e"
RED = "#ef4444"
TEXT = "#e5e7eb"
SUB = "#94a3b8"

FONT_H1 = ("Arial", 22, "bold")
FONT_H2 = ("Arial", 16, "bold")
FONT = ("Arial", 14)
FONT_SMALL = ("Arial", 12)

DATA_FILE = "data.json"

# ===================== DATA =====================
data = {"users": {}}
current_user = None
calc_mode = "1"

def load_data():
    global data
    if os.path.exists(DATA_FILE):
        try:
            with open(DATA_FILE, "r", encoding="utf-8") as f:
                data = json.load(f)
            if "users" not in data:
                data = {"users": {}}
        except:
            data = {"users": {}}
    else:
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
root.grid_columnconfigure(0, weight=1)

# ===================== SIDEBAR (VISIBLE FIXED) =====================
sidebar = tk.Frame(root, bg=PANEL, width=300, height=750)
sidebar.place(x=0, y=0)   # ВСЕГДА ВИДИМ

menu_open = True

def toggle_menu():
    global menu_open
    if menu_open:
        sidebar.place(x=-320, y=0)
    else:
        sidebar.place(x=0, y=0)
    menu_open = not menu_open

# ===================== MAIN =====================
main = tk.Frame(root, bg=BG)
main.pack(fill="both", expand=True)

# ===================== HEADER =====================
header = tk.Frame(main, bg=BG)
header.pack(fill="x")

tk.Button(header, text="☰",
          bg=BG, fg=TEXT,
          font=("Arial", 20),
          bd=0,
          command=toggle_menu).pack(side="left", padx=15, pady=10)

user_label = tk.Label(header, text="",
                      bg=BG, fg=SUB,
                      font=FONT)
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
             font=FONT_H1).pack(pady=10)

    email = tk.Entry(c, font=FONT)
    password = tk.Entry(c, show="*", font=FONT)

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

        if not e or not p:
            return messagebox.showerror("Ошибка", "Заполни поля")

        if e in data["users"]:
            return messagebox.showerror("Ошибка", "Уже существует")

        data["users"][e] = {
            "password": p,
            "username": e,
            "cars": [],
            "selected_car": "",
            "history": []
        }
        save_data()
        messagebox.showinfo("OK", "Аккаунт создан")

    tk.Button(c, text="Войти",
              bg=ACCENT, fg="white",
              font=FONT,
              command=login).pack(fill="x", pady=10)

    tk.Button(c, text="Регистрация",
              bg=GREEN, fg="black",
              font=FONT,
              command=register).pack(fill="x")

# ===================== PROFILE =====================
def show_profile():
    clear()
    if not current_user:
        show_auth()
        return

    u = data["users"][current_user]
    c = card()

    tk.Label(c, text="Профиль",
             bg=CARD, fg=TEXT,
             font=FONT_H1).pack()

    username = tk.Entry(c, font=FONT)
    username.insert(0, u.get("username", ""))

    password = tk.Entry(c, show="*", font=FONT)

    cars = tk.Entry(c, font=FONT)
    cars.insert(0, ",".join(u.get("cars", [])))

    tk.Label(c, text="Username", bg=CARD, fg=SUB, font=FONT).pack(anchor="w")
    username.pack(fill="x")

    tk.Label(c, text="Новый пароль", bg=CARD, fg=SUB, font=FONT).pack(anchor="w")
    password.pack(fill="x")

    tk.Label(c, text="Машины (через запятую)", bg=CARD, fg=SUB, font=FONT).pack(anchor="w")
    cars.pack(fill="x")

    def save():
        u["username"] = username.get()

        if password.get():
            u["password"] = password.get()

        u["cars"] = [x.strip() for x in cars.get().split(",") if x.strip()]

        if u["cars"] and u["selected_car"] not in u["cars"]:
            u["selected_car"] = u["cars"][0]

        save_data()
        update_user()
        messagebox.showinfo("OK", "Сохранено")

    tk.Button(c, text="Сохранить",
              bg=ACCENT, fg="white",
              font=FONT,
              command=save).pack(fill="x", pady=10)

# ===================== CALC =====================
def show_calc():
    clear()

    if not current_user:
        show_auth()
        return

    u = data["users"][current_user]

    tk.Label(content, text="Калькулятор расхода",
             bg=BG, fg=TEXT,
             font=FONT_H1).pack(pady=10)

    mode_var = tk.StringVar(value=calc_mode)

    def change():
        global calc_mode
        calc_mode = mode_var.get()
        build()

    switch = tk.Frame(content, bg=BG)
    switch.pack(pady=10)

    tk.Radiobutton(switch,
                   text="Стоимость поездки",
                   variable=mode_var,
                   value="1",
                   command=change,
                   font=FONT,
                   bg=BG).pack(side="left", padx=10)

    tk.Radiobutton(switch,
                   text="Средний расход",
                   variable=mode_var,
                   value="2",
                   command=change,
                   font=FONT,
                   bg=BG).pack(side="left", padx=10)

    form = tk.Frame(content, bg=CARD, padx=40, pady=40)
    form.pack(pady=20)

    entries = {}

    car_var = tk.StringVar(value=u.get("selected_car", ""))

    def build():
        for w in form.winfo_children():
            w.destroy()
        entries.clear()

        tk.Label(form, text="Машина", bg=CARD, fg=TEXT, font=FONT).pack(anchor="w")
        tk.OptionMenu(form, car_var, *u.get("cars", ["Нет авто"])).pack(fill="x")

        def add(t):
            tk.Label(form, text=t, bg=CARD, fg=TEXT, font=FONT).pack(anchor="w")
            e = tk.Entry(form, font=FONT)
            e.pack(fill="x", pady=8, ipady=5)
            entries[t] = e

        if mode_var.get() == "1":
            add("Топливо (л)")
            add("Расстояние (км)")
            add("Цена (₽)")
        else:
            add("Средний расход")
            add("Расстояние (км)")
            add("Цена (₽)")

    def calc():
        try:
            if mode_var.get() == "1":
                f = float(entries["Топливо (л)"].get())
                d = float(entries["Расстояние (км)"].get())
                p = float(entries["Цена (₽)"].get())
                cons = (f / d) * 100
                cost = f * p
            else:
                cons = float(entries["Средний расход"].get())
                d = float(entries["Расстояние (км)"].get())
                p = float(entries["Цена (₽)"].get())
                f = cons * d / 100
                cost = f * p

            car = car_var.get()

            result.config(text=f"{cons:.1f} л/100км | {cost:.0f} ₽")

            u["selected_car"] = car

            u["history"].append({
                "date": datetime.now().strftime("%d.%m %H:%M"),
                "car": car,
                "result": result.cget("text")
            })

            save_data()

        except:
            messagebox.showerror("Ошибка", "Проверь ввод")

    result = tk.Label(content, text="—",
                      bg=BG, fg=ACCENT,
                      font=FONT_H1)
    result.pack()

    tk.Button(content, text="Рассчитать",
              bg=ACCENT, fg="white",
              font=FONT,
              command=calc).pack(pady=10)

    build()

# ===================== HISTORY =====================
def show_history():
    clear()

    if not current_user:
        return

    u = data["users"][current_user]

    tk.Label(content, text="История",
             bg=BG, fg=TEXT,
             font=FONT_H1).pack()

    for h in u["history"][::-1]:
        tk.Label(content,
                 text=f"{h['date']} | {h['car']} | {h['result']}",
                 bg=CARD, fg=TEXT,
                 font=FONT).pack(fill="x", padx=20, pady=5)

# ===================== MENU =====================
def nav(t, c):
    tk.Button(sidebar, text=t,
              bg=PANEL, fg=TEXT,
              font=FONT,
              bd=0,
              anchor="w",
              padx=20,
              pady=15,
              command=c).pack(fill="x")

nav("Калькулятор", show_calc)
nav("Профиль", show_profile)
nav("История", show_history)

# ===================== USER =====================
def update_user():
    if current_user:
        user_label.config(text=data["users"][current_user].get("username", current_user))
    else:
        user_label.config(text="")

# ===================== START =====================
load_data()
show_calc()
root.mainloop()
