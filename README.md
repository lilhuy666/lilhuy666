import tkinter as tk
from tkinter import messagebox
import json, os, hashlib
from datetime import datetime

# ========= DATA =========
DATA_FILE = "data.json"

data = {
    "users": {},
    "settings": {"currency": "€", "precision": 2}
}
current_user = None

def load_data():
    global data
    if os.path.exists(DATA_FILE):
        with open(DATA_FILE, "r", encoding="utf-8") as f:
            data = json.load(f)

def save_data():
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=2, ensure_ascii=False)

def hash_password(p):
    return hashlib.sha256(p.encode()).hexdigest()

# ========= UI COLORS =========
BG = "#f2f2f7"
CARD = "#ffffff"
TEXT = "#1c1c1e"
SUB = "#8e8e93"
ACCENT = "#007aff"
DANGER = "#ff3b30"

# ========= WINDOW =========
root = tk.Tk()
root.title("CalculatCar")
root.geometry("900x650")
root.configure(bg=BG)

# ========= LAYOUT =========
header = tk.Frame(root, bg=BG)
header.pack(fill="x", pady=10)

content = tk.Frame(root, bg=BG)
content.pack(fill="both", expand=True, padx=20)

def clear():
    for w in content.winfo_children():
        w.destroy()

def card():
    f = tk.Frame(content, bg=CARD, bd=0)
    f.pack(fill="x", pady=10)
    return f

# ========= HEADER =========
title = tk.Label(header, text="🚗 CalculatCar",
                 bg=BG, fg=TEXT,
                 font=("Arial", 20, "bold"))
title.pack(side="left", padx=20)

user_label = tk.Label(header, text="", bg=BG, fg=SUB)
user_label.pack(side="right", padx=20)

def update_user():
    user_label.config(text=current_user if current_user else "")

# ========= MENU =========
menu = tk.Frame(root, bg=CARD)
menu.pack(fill="x")

def nav_button(text, cmd):
    tk.Button(menu, text=text,
              bg=CARD, fg=TEXT,
              bd=0, padx=15, pady=10,
              command=cmd).pack(side="left")

# ========= CALCULATOR =========
def show_calc():
    clear()

    c = card()

    tk.Label(c, text="⛽ Калькулятор",
             bg=CARD, fg=TEXT,
             font=("Arial", 16, "bold")).pack(pady=10)

    mode = tk.StringVar(value="1")
    entries = {}

    switch = tk.Frame(c, bg=CARD)
    switch.pack(pady=5)

    tk.Radiobutton(switch, text="Стоимость",
                   variable=mode, value="1",
                   bg=CARD).pack(side="left", padx=5)

    tk.Radiobutton(switch, text="Расход",
                   variable=mode, value="2",
                   bg=CARD).pack(side="left", padx=5)

    form = tk.Frame(c, bg=CARD)
    form.pack(pady=10)

    result = tk.Label(c, text="—", bg=CARD,
                      fg=ACCENT, font=("Arial", 16, "bold"))
    result.pack(pady=10)

    def build():
        for w in form.winfo_children():
            w.destroy()
        entries.clear()

        def add(name):
            tk.Label(form, text=name, bg=CARD).pack(anchor="w")
            e = tk.Entry(form)
            e.pack(fill="x", pady=5)
            entries[name] = e

        if mode.get() == "1":
            add("Топливо")
            add("Расстояние")
            add("Цена")
        else:
            add("Расход")
            add("Расстояние")
            add("Цена")

    def calc():
        try:
            precision = data["settings"]["precision"]
            currency = data["settings"]["currency"]

            if mode.get() == "1":
                f = float(entries["Топливо"].get())
                d = float(entries["Расстояние"].get())
                p = float(entries["Цена"].get())
                cons = (f / d) * 100
                cost = f * p
            else:
                cons = float(entries["Расход"].get())
                d = float(entries["Расстояние"].get())
                p = float(entries["Цена"].get())
                cost = cons * d / 100 * p

            result.config(
                text=f"{round(cons,precision)} л/100км | {round(cost,precision)} {currency}"
            )

            if current_user:
                data["users"][current_user]["history"].append({
                    "date": datetime.now().strftime("%d.%m %H:%M"),
                    "result": result.cget("text")
                })
                data["users"][current_user]["history"] = data["users"][current_user]["history"][-50:]
                save_data()

        except:
            messagebox.showerror("Ошибка", "Проверь данные")

    tk.Button(c, text="Рассчитать",
              bg=ACCENT, fg="white",
              command=calc).pack()

    mode.trace_add("write", lambda *a: build())
    build()

# ========= PROFILE =========
def show_profile():
    clear()

    c = card()

    if not current_user:
        tk.Label(c, text="👤 Вход",
                 bg=CARD, font=("Arial", 16, "bold")).pack(pady=10)

        email = tk.Entry(c)
        password = tk.Entry(c, show="*")

        email.pack(fill="x", pady=5)
        password.pack(fill="x", pady=5)

        def login():
            global current_user
            e = email.get()
            p = password.get()

            if e in data["users"] and data["users"][e]["password"] == hash_password(p):
                current_user = e
                update_user()
                show_profile()
            else:
                messagebox.showerror("Ошибка", "Неверные данные")

        def register():
            e = email.get()
            p = password.get()

            if e in data["users"]:
                return messagebox.showerror("Ошибка", "Уже есть")

            data["users"][e] = {"password": hash_password(p), "history": []}
            save_data()
            messagebox.showinfo("OK", "Создано")

        tk.Button(c, text="Войти", bg=ACCENT, fg="white", command=login).pack(pady=5)
        tk.Button(c, text="Регистрация", command=register).pack()

    else:
        tk.Label(c, text=f"👤 {current_user}",
                 bg=CARD, font=("Arial", 16)).pack(pady=10)

        def logout():
            global current_user
            current_user = None
            update_user()
            show_profile()

        tk.Button(c, text="Выйти", bg=DANGER, fg="white",
                  command=logout).pack()

# ========= HISTORY =========
def show_history():
    clear()

    c = card()

    tk.Label(c, text="📊 История",
             bg=CARD, font=("Arial", 16, "bold")).pack(pady=10)

    if not current_user:
        tk.Label(c, text="Войдите", bg=CARD).pack()
        return

    history = data["users"][current_user]["history"]

    if not history:
        tk.Label(c, text="Пусто", bg=CARD).pack()
        return

    for h in reversed(history):
        tk.Label(c,
                 text=f"{h['date']} | {h['result']}",
                 bg=CARD).pack(anchor="w")

    def clear_hist():
        data["users"][current_user]["history"] = []
        save_data()
        show_history()

    tk.Button(c, text="Очистить",
              bg=DANGER, fg="white",
              command=clear_hist).pack(pady=10)

# ========= SETTINGS =========
def show_settings():
    clear()

    c = card()

    tk.Label(c, text="⚙️ Настройки",
             bg=CARD, font=("Arial", 16, "bold")).pack(pady=10)

    currency = tk.StringVar(value=data["settings"]["currency"])
    precision = tk.IntVar(value=data["settings"]["precision"])

    tk.Label(c, text="Валюта", bg=CARD).pack()
    tk.Entry(c, textvariable=currency).pack()

    tk.Label(c, text="Точность", bg=CARD).pack()
    tk.Entry(c, textvariable=precision).pack()

    def save():
        data["settings"]["currency"] = currency.get()
        data["settings"]["precision"] = precision.get()
        save_data()
        messagebox.showinfo("OK", "Сохранено")

    tk.Button(c, text="Сохранить",
              bg=ACCENT, fg="white",
              command=save).pack(pady=10)

# ========= ABOUT =========
def show_about():
    clear()

    c = card()

    tk.Label(c, text="ℹ️ О программе",
             bg=CARD, font=("Arial", 16, "bold")).pack(pady=10)

    tk.Label(c,
             text="CalculatCar\nКалькулятор топлива\n\nВерсия 2.0",
             bg=CARD).pack(pady=10)

# ========= MENU BUTTONS =========
nav_button("⛽", show_calc)
nav_button("👤", show_profile)
nav_button("📊", show_history)
nav_button("⚙️", show_settings)
nav_button("ℹ️", show_about)

# ========= START =========
load_data()
update_user()
show_calc()

root.mainloop()
