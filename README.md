import tkinter as tk
from tkinter import messagebox
import json, os
from datetime import datetime

# =========================
# 🎨 THEME (modern dark + soft)
# =========================
BG = "#0b1220"
PANEL = "#111a2e"
CARD = "#162238"
ACCENT = "#4f8cff"
ACCENT2 = "#22c55e"
TEXT = "#e5e7eb"
SUB = "#94a3b8"
DANGER = "#ef4444"

DATA_FILE = "data.json"

# =========================
# 💾 DATA SAFE
# =========================
data = {"users": {}}
current_user = None

def load_data():
    global data
    if not os.path.exists(DATA_FILE):
        data = {"users": {}}
        return

    try:
        with open(DATA_FILE, "r", encoding="utf-8") as f:
            data = json.load(f)

        if "users" not in data:
            data = {"users": {}}

    except:
        data = {"users": {}}

def save_data():
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=2, ensure_ascii=False)

# =========================
# 🪟 WINDOW
# =========================
root = tk.Tk()
root.title("CalculatCar Pro 🚗")
root.geometry("1200x750")
root.configure(bg=BG)

root.grid_rowconfigure(0, weight=1)
root.grid_columnconfigure(1, weight=1)

# =========================
# SIDEBAR
# =========================
sidebar = tk.Frame(root, bg=PANEL, width=260)
sidebar.grid(row=0, column=0, sticky="ns")

# =========================
# MAIN
# =========================
main = tk.Frame(root, bg=BG)
main.grid(row=0, column=1, sticky="nsew")

# =========================
# HEADER
# =========================
header = tk.Frame(main, bg=BG, height=70)
header.pack(fill="x")

title = tk.Label(header, text="🚗 CalculatCar Pro",
                 bg=BG, fg=TEXT,
                 font=("Arial", 20, "bold"))
title.pack(side="left", padx=20, pady=15)

user_lbl = tk.Label(header, text="",
                    bg=BG, fg=SUB,
                    font=("Arial", 12))
user_lbl.pack(side="right", padx=20)

# =========================
# CONTENT
# =========================
content = tk.Frame(main, bg=BG)
content.pack(fill="both", expand=True)

def clear():
    for w in content.winfo_children():
        w.destroy()

def big_card():
    f = tk.Frame(content, bg=CARD, padx=40, pady=40)
    f.pack(pady=40, ipadx=20, ipady=20)
    return f

# =========================
# NAV BUTTON
# =========================
def nav(text, cmd):
    tk.Button(sidebar, text=text,
              bg=PANEL, fg=TEXT,
              font=("Arial", 12),
              bd=0, anchor="w",
              padx=20, pady=15,
              activebackground=CARD,
              activeforeground=ACCENT,
              command=cmd).pack(fill="x")

# =========================
# AUTH
# =========================
def show_auth():
    clear()
    c = big_card()

    tk.Label(c, text="Вход / Регистрация",
             bg=CARD, fg=TEXT,
             font=("Arial", 22, "bold")).pack(pady=10)

    email = tk.Entry(c, font=("Arial", 14))
    password = tk.Entry(c, show="*", font=("Arial", 14))

    email.pack(fill="x", pady=10)
    password.pack(fill="x", pady=10)

    def login():
        global current_user
        e = email.get().strip()
        p = password.get().strip()

        if e in data["users"] and data["users"][e]["password"] == p:
            current_user = e
            update_user()
            show_calc()
        else:
            messagebox.showerror("Ошибка", "Неверный логин или пароль")

    def register():
        e = email.get().strip()
        p = password.get().strip()

        if e == "" or p == "":
            return messagebox.showerror("Ошибка", "Заполни поля")

        if e in data["users"]:
            return messagebox.showerror("Ошибка", "Уже существует")

        data["users"][e] = {"password": p, "history": []}
        save_data()
        messagebox.showinfo("OK", "Аккаунт создан")

    tk.Button(c, text="Войти",
              bg=ACCENT, fg="white",
              font=("Arial", 12, "bold"),
              command=login).pack(fill="x", pady=10)

    tk.Button(c, text="Регистрация",
              bg=ACCENT2, fg="black",
              font=("Arial", 12, "bold"),
              command=register).pack(fill="x")

# =========================
# PROFILE
# =========================
def show_profile():
    clear()
    c = big_card()

    tk.Label(c, text="Профиль",
             bg=CARD, fg=TEXT,
             font=("Arial", 22, "bold")).pack()

    tk.Label(c, text=current_user or "-",
             bg=CARD, fg=SUB,
             font=("Arial", 14)).pack(pady=10)

    def logout():
        global current_user
        current_user = None
        show_auth()

    tk.Button(c, text="Выйти",
              bg=DANGER, fg="white",
              command=logout).pack(fill="x", pady=10)

# =========================
# CALC
# =========================
def show_calc():
    clear()

    tk.Label(content, text="Калькулятор расхода",
             bg=BG, fg=TEXT,
             font=("Arial", 22, "bold")).pack(pady=10)

    mode = tk.StringVar(value="1")

    switch = tk.Frame(content, bg=BG)
    switch.pack()

    tk.Radiobutton(switch, text="Фактический расход",
                   variable=mode, value="1",
                   bg=BG, fg=TEXT).pack(side="left", padx=20)

    tk.Radiobutton(switch, text="По среднему расходу",
                   variable=mode, value="2",
                   bg=BG, fg=TEXT).pack(side="left", padx=20)

    form = tk.Frame(content, bg=CARD, padx=40, pady=40)
    form.pack(pady=30)

    entries = {}

    def build():
        for w in form.winfo_children():
            w.destroy()
        entries.clear()

        def add(t):
            tk.Label(form, text=t, bg=CARD, fg=TEXT,
                     font=("Arial", 12)).pack(anchor="w")
            e = tk.Entry(form, font=("Arial", 14))
            e.pack(fill="x", pady=10, ipady=5)
            entries[t] = e

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
                cost = f * p
            else:
                cons = float(entries["Средний расход (л/100км)"].get())
                d = float(entries["Расстояние (км)"].get())
                p = float(entries["Цена за литр"].get())
                f = cons * d / 100
                cost = f * p

            result.config(text=f"{cons:.1f} л/100км | {cost:.0f} ₽")

            if current_user:
                data["users"][current_user]["history"].append({
                    "date": datetime.now().strftime("%d.%m %H:%M"),
                    "result": result.cget("text")
                })
                save_data()

        except:
            messagebox.showerror("Ошибка", "Проверь ввод")

    result = tk.Label(content, text="—",
                      bg=BG, fg=ACCENT,
                      font=("Arial", 24, "bold"))
    result.pack(pady=10)

    tk.Button(content, text="Рассчитать",
              bg=ACCENT, fg="white",
              font=("Arial", 14, "bold"),
              command=calc).pack(pady=10)

    mode.trace("w", lambda *a: build())
    build()

# =========================
# HISTORY
# =========================
def show_history():
    clear()

    tk.Label(content, text="История",
             bg=BG, fg=TEXT,
             font=("Arial", 22, "bold")).pack(pady=10)

    if not current_user:
        return

    for h in data["users"][current_user]["history"][::-1]:
        tk.Label(content, text=f"{h['date']}  |  {h['result']}",
                 bg=CARD, fg=TEXT,
                 font=("Arial", 12)).pack(fill="x", padx=40, pady=5)

# =========================
# SETTINGS
# =========================
def show_settings():
    clear()
    tk.Label(content, text="Настройки",
             bg=BG, fg=TEXT,
             font=("Arial", 22, "bold")).pack(pady=40)

# =========================
# ABOUT
# =========================
def show_about():
    clear()
    tk.Label(content,
             text="CalculatCar Pro 🚗\nКрасивое приложение для расчёта расхода",
             bg=BG, fg=TEXT,
             font=("Arial", 16)).pack(pady=60)

# =========================
# NAV MENU
# =========================
nav("Калькулятор", show_calc)
nav("Профиль", show_profile)
nav("История", show_history)
nav("Настройки", show_settings)
nav("О программе", show_about)
nav("Вход", show_auth)

# =========================
# USER UPDATE
# =========================
def update_user():
    user_lbl.config(text=current_user if current_user else "")

# =========================
# START
# =========================
load_data()
show_calc()
root.mainloop()
