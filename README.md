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
root.title("CalculatCar")
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

title = tk.Label(header, text="CalculatCar",
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

        if not e or not p:
            return messagebox.showerror("Ошибка", "Заполни поля")

        if e in data["users"]:
            return messagebox.showerror("Ошибка", "Уже существует")

        data["users"][e] = {
            "password": p,
            "history": [],
            "name": "",
            "car": ""
        }
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

# ===================== PROFILE =====================
def show_profile():
    if not current_user:
        show_auth()
        return

    clear()
    c = card()

    user = data["users"][current_user]

    tk.Label(c, text="Профиль",
             bg=CARD, fg=TEXT,
             font=("Arial", 20, "bold")).pack()

    name_entry = tk.Entry(c, font=("Arial", 14))
    name_entry.insert(0, user.get("name", ""))

    car_entry = tk.Entry(c, font=("Arial", 14))
    car_entry.insert(0, user.get("car", ""))

    new_pass = tk.Entry(c, show="*", font=("Arial", 14))

    tk.Label(c, text="Имя", bg=CARD, fg=SUB).pack(anchor="w", pady=(10,0))
    name_entry.pack(fill="x")

    tk.Label(c, text="Авто", bg=CARD, fg=SUB).pack(anchor="w", pady=(10,0))
    car_entry.pack(fill="x")

    tk.Label(c, text="Новый пароль", bg=CARD, fg=SUB).pack(anchor="w", pady=(10,0))
    new_pass.pack(fill="x")

    def save_profile():
        user["name"] = name_entry.get().strip()
        user["car"] = car_entry.get().strip()
        if new_pass.get().strip():
            user["password"] = new_pass.get().strip()
        save_data()
        messagebox.showinfo("OK", "Сохранено")

    def logout():
        global current_user
        current_user = None
        update_user()
        show_auth()

    tk.Button(c, text="Сохранить",
              bg=ACCENT, fg="white",
              command=save_profile).pack(fill="x", pady=10)

    tk.Button(c, text="Выйти",
              bg=DANGER, fg="white",
              command=logout).pack(fill="x")

# ===================== CALC =====================
def show_calc():
    clear()

    tk.Label(content, text="Калькулятор расхода",
             bg=BG, fg=TEXT,
             font=("Arial", 20, "bold")).pack(pady=10)

    if not current_user:
        tk.Button(content, text="Войти",
                  bg=ACCENT,
                  fg="white",
                  command=show_auth).pack(pady=10)
        return

    # дальше всё как у тебя 👇

    mode = tk.StringVar(value="1")

    switch = tk.Frame(content, bg=BG)
    switch.pack(pady=10)

    tk.Radiobutton(switch, text="Поездка", variable=mode, value="1",
                   indicatoron=0, width=20, bg=PANEL, fg=TEXT,
                   selectcolor=ACCENT).pack(side="left", padx=5)

    tk.Radiobutton(switch, text="Средний расход", variable=mode, value="2",
                   indicatoron=0, width=20, bg=PANEL, fg=TEXT,
                   selectcolor=ACCENT).pack(side="left", padx=5)

    form = tk.Frame(content, bg=CARD, padx=40, pady=40)
    form.pack(pady=20)

    entries = {}

    def build():
        for w in form.winfo_children():
            w.destroy()
        entries.clear()

        def add(t):
            tk.Label(form, text=t, bg=CARD, fg=TEXT).pack(anchor="w")
            e = tk.Entry(form)
            e.pack(fill="x", pady=5)
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

        except:
            messagebox.showerror("Ошибка", "Проверь ввод")

    result = tk.Label(content, text="—",
                      bg=BG, fg=ACCENT,
                      font=("Arial", 22, "bold"))
    result.pack()

    tk.Button(content, text="Рассчитать",
              bg=ACCENT, fg="white",
              command=calc).pack(pady=10)

    mode.trace("w", lambda *a: build())
    build()

# ===================== MENU =====================
def nav(text, cmd):
    tk.Button(sidebar,
              text=text,
              bg=PANEL,
              fg=TEXT,
              bd=0,
              anchor="w",
              padx=20,
              pady=14,
              command=cmd).pack(fill="x")

nav("Калькулятор", show_calc)
nav("Профиль", show_profile)

# ===================== USER =====================
def update_user():
    if current_user:
        user_label.config(text=data["users"][current_user].get("name") or current_user)
    else:
        user_label.config(text="")

# ===================== START =====================
load_data()
show_calc()
root.mainloop()
