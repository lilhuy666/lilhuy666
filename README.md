import tkinter as tk
from tkinter import messagebox
from datetime import datetime
import json, os

# ===================== STYLE =====================
BG = "#0b1220"
PANEL = "#111a2e"
SIDEBAR = "#162238"
CARD = "#1b2a41"
ACCENT = "#4f8cff"
GREEN = "#22c55e"
RED = "#ef4444"
TEXT = "#e5e7eb"
SUB = "#94a3b8"

FONT = ("Arial", 14)
FONT_H = ("Arial", 20, "bold")

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
root.title("CalculatCar 🚗")
root.geometry("1200x750")
root.configure(bg=BG)

root.grid_rowconfigure(0, weight=1)
root.grid_columnconfigure(0, weight=1)

# ===================== SIDEBAR FIXED =====================
sidebar = tk.Frame(root, bg=PANEL, width=300, height=750)
sidebar.place(x=-320, y=0)

menu_open = False

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

# HEADER
header = tk.Frame(main, bg=BG)
header.pack(fill="x")

tk.Button(header, text="☰",
          bg=BG, fg=TEXT,
          font=("Arial", 20),
          bd=0,
          command=toggle_menu).pack(side="left", padx=15, pady=10)

user_lbl = tk.Label(header, text="", bg=BG, fg=SUB, font=FONT)
user_lbl.pack(side="right", padx=15)

# ===================== CONTENT =====================
content = tk.Frame(main, bg=BG)
content.pack(fill="both", expand=True)

def clear():
    for w in content.winfo_children():
        w.destroy()

def card():
    f = tk.Frame(content, bg=CARD, padx=40, pady=40)
    f.pack(pady=20)
    return f

# ===================== AUTH (не стартует) =====================
def show_auth():
    clear()
    c = card()

    tk.Label(c, text="Вход / Регистрация",
             bg=CARD, fg=TEXT,
             font=FONT_H).pack(pady=10)

    e = tk.Entry(c, font=FONT)
    p = tk.Entry(c, show="*", font=FONT)

    e.pack(fill="x", pady=5)
    p.pack(fill="x", pady=5)

    def login():
        global current_user
        if e.get() in data["users"] and data["users"][e.get()]["password"] == p.get():
            current_user = e.get()
            update_user()
            show_calc()
        else:
            messagebox.showerror("Ошибка", "Неверно")

    def reg():
        if e.get() in data["users"]:
            return messagebox.showerror("Ошибка", "Уже есть")

        data["users"][e.get()] = {
            "password": p.get(),
            "cars": [],
            "selected_car": "",
            "history": [],
            "username": e.get()
        }
        save_data()
        messagebox.showinfo("OK", "Создано")

    tk.Button(c, text="Войти", bg=ACCENT, fg="white",
              font=FONT, command=login).pack(fill="x", pady=10)

    tk.Button(c, text="Регистрация", bg=GREEN,
              font=FONT, command=reg).pack(fill="x")

# ===================== PROFILE =====================
def show_profile():
    clear()

    if not current_user:
        show_auth()
        return

    u = data["users"][current_user]
    c = card()

    tk.Label(c, text="Профиль", bg=CARD, fg=TEXT, font=FONT_H).pack()

    name = tk.Entry(c, font=FONT)
    name.insert(0, u.get("username", ""))

    cars = tk.Entry(c, font=FONT)
    cars.insert(0, ",".join(u.get("cars", [])))

    tk.Label(c, text="Имя", bg=CARD, fg=SUB).pack(anchor="w")
    name.pack(fill="x")

    tk.Label(c, text="Машины", bg=CARD, fg=SUB).pack(anchor="w")
    cars.pack(fill="x")

    def save():
        u["username"] = name.get()
        u["cars"] = [x.strip() for x in cars.get().split(",") if x.strip()]
        save_data()
        update_user()

    tk.Button(c, text="Сохранить", bg=ACCENT,
              fg="white", font=FONT,
              command=save).pack(fill="x", pady=10)

# ===================== CALC (START SCREEN) =====================
def show_calc():
    clear()

    tk.Label(content, text="Калькулятор расхода",
             bg=BG, fg=TEXT, font=FONT_H).pack(pady=10)

    form = card()

    car_var = tk.StringVar()

    def build():
        for w in form.winfo_children():
            w.destroy()

        tk.Label(form, text="Машина", bg=CARD, fg=TEXT).pack(anchor="w")
        tk.OptionMenu(form, car_var, "Без авто").pack(fill="x")

        entries = {}

        def add(t):
            tk.Label(form, text=t, bg=CARD, fg=TEXT).pack(anchor="w")
            e = tk.Entry(form, font=FONT)
            e.pack(fill="x", pady=5)
            entries[t] = e

        add("Топливо")
        add("Расстояние")
        add("Цена")

        def calc():
            try:
                f = float(entries["Топливо"].get())
                d = float(entries["Расстояние"].get())
                p = float(entries["Цена"].get())

                cons = (f / d) * 100
                cost = f * p

                result.config(text=f"{cons:.1f} л/100км | {cost:.0f} ₽")

            except:
                messagebox.showerror("Ошибка", "Ввод")

        tk.Button(form, text="Рассчитать",
                  bg=ACCENT, fg="white",
                  font=FONT,
                  command=calc).pack(pady=10)

    result = tk.Label(content, text="—",
                      bg=BG, fg=ACCENT,
                      font=FONT_H)
    result.pack()

    build()

# ===================== HISTORY =====================
def show_history():
    clear()

    if not current_user:
        return

    u = data["users"][current_user]

    tk.Label(content, text="История",
             bg=BG, fg=TEXT,
             font=FONT_H).pack()

    for h in u["history"][::-1]:
        tk.Label(content,
                 text=f"{h['date']} | {h['result']}",
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
        user_lbl.config(text=data["users"][current_user]["username"])

# ===================== START FIX =====================
load_data()

# 🔥 ВАЖНО: СТАРТ СРАЗУ В КАЛЬКУЛЯТОРЕ
show_calc()

root.mainloop()
