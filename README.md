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
root.title("CalculatCar Pro 🚗")
root.geometry("1200x750")
root.configure(bg=BG)

root.grid_rowconfigure(0, weight=1)
root.grid_columnconfigure(0, weight=1)

# ===================== SIDEBAR (overlay) =====================
sidebar = tk.Frame(root, bg=PANEL, width=260, height=750)
sidebar.place(x=-260, y=0)

menu_open = False

def toggle_menu():
    global menu_open
    if menu_open:
        sidebar.place(x=-260, y=0)
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
          font=("Arial", 18),
          bd=0,
          command=toggle_menu).pack(side="left", padx=15, pady=10)

user_lbl = tk.Label(header, text="",
                    bg=BG, fg=SUB,
                    font=("Arial", 12))
user_lbl.pack(side="right", padx=15)

# CONTENT
content = tk.Frame(main, bg=BG)
content.pack(fill="both", expand=True)

def clear():
    for w in content.winfo_children():
        w.destroy()

def card():
    f = tk.Frame(content, bg=CARD, padx=35, pady=35)
    f.pack(pady=25)
    return f

# ===================== AUTH =====================
def show_auth():
    clear()
    c = card()

    tk.Label(c, text="Вход / Регистрация",
             bg=CARD, fg=TEXT,
             font=("Arial", 18, "bold")).pack(pady=10)

    email = tk.Entry(c, font=("Arial", 14))
    password = tk.Entry(c, show="*", font=("Arial", 14))

    email.pack(fill="x", pady=5)
    password.pack(fill="x", pady=5)

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

    tk.Button(c, text="Войти", bg=ACCENT, fg="white",
              command=login).pack(fill="x", pady=10)

    tk.Button(c, text="Регистрация", bg=GREEN,
              command=register).pack(fill="x")

# ===================== PROFILE =====================
def show_profile():
    clear()
    if not current_user:
        show_auth()
        return

    user = data["users"][current_user]
    c = card()

    tk.Label(c, text="Профиль",
             bg=CARD, fg=TEXT,
             font=("Arial", 18, "bold")).pack()

    username = tk.Entry(c)
    username.insert(0, user.get("username", ""))

    password = tk.Entry(c, show="*")

    tk.Label(c, text="Username", bg=CARD, fg=SUB).pack(anchor="w")
    username.pack(fill="x")

    tk.Label(c, text="Новый пароль", bg=CARD, fg=SUB).pack(anchor="w")
    password.pack(fill="x")

    tk.Label(c, text="Машины (через запятую)",
             bg=CARD, fg=SUB).pack(anchor="w")

    cars = tk.Entry(c)
    cars.insert(0, ",".join(user.get("cars", [])))
    cars.pack(fill="x")

    def save_profile():
        user["username"] = username.get()

        if password.get().strip():
            user["password"] = password.get()

        user["cars"] = [c.strip() for c in cars.get().split(",") if c.strip()]

        if user["cars"] and user["selected_car"] not in user["cars"]:
            user["selected_car"] = user["cars"][0]

        save_data()
        update_user()
        messagebox.showinfo("OK", "Сохранено")

    tk.Button(c, text="Сохранить",
              bg=ACCENT, fg="white",
              command=save_profile).pack(fill="x", pady=10)

# ===================== CALC =====================
def show_calc():
    clear()

    if not current_user:
        show_auth()
        return

    user = data["users"][current_user]

    tk.Label(content, text="Калькулятор",
             bg=BG, fg=TEXT,
             font=("Arial", 18, "bold")).pack(pady=10)

    car_var = tk.StringVar(value=user.get("selected_car", ""))

    tk.OptionMenu(content, car_var,
                  *user.get("cars", ["Нет авто"])).pack()

    form = tk.Frame(content, bg=CARD, padx=30, pady=30)
    form.pack(pady=20)

    entries = {}

    def add(t):
        tk.Label(form, text=t, bg=CARD, fg=TEXT).pack(anchor="w")
        e = tk.Entry(form)
        e.pack(fill="x", pady=5)
        entries[t] = e

    add("Топливо")
    add("Расстояние")
    add("Цена")

    result = tk.Label(content, text="—",
                      bg=BG, fg=ACCENT,
                      font=("Arial", 20, "bold"))
    result.pack()

    def calc():
        try:
            f = float(entries["Топливо"].get())
            d = float(entries["Расстояние"].get())
            p = float(entries["Цена"].get())

            cons = (f / d) * 100
            cost = f * p

            car = car_var.get()

            result.config(text=f"{cons:.1f} л/100км | {cost:.0f} ₽")

            user["selected_car"] = car

            user["history"].append({
                "date": datetime.now().strftime("%d.%m %H:%M"),
                "car": car,
                "result": result.cget("text")
            })

            save_data()

        except:
            messagebox.showerror("Ошибка", "Проверь ввод")

    tk.Button(content, text="Рассчитать",
              bg=ACCENT, fg="white",
              command=calc).pack()

# ===================== HISTORY =====================
def show_history():
    clear()

    if not current_user:
        return

    user = data["users"][current_user]

    tk.Label(content, text="История",
             bg=BG, fg=TEXT,
             font=("Arial", 18, "bold")).pack()

    def delete(i):
        del user["history"][i]
        save_data()
        show_history()

    for i, h in enumerate(user["history"][::-1]):
        frame = tk.Frame(content, bg=CARD, padx=10, pady=10)
        frame.pack(fill="x", padx=20, pady=5)

        tk.Label(frame,
                 text=f"{h['date']} | {h['car']} | {h['result']}",
                 bg=CARD, fg=TEXT).pack(side="left")

        tk.Button(frame, text="🗑",
                  bg=RED, fg="white",
                  command=lambda i=i: delete(len(user["history"]) - 1 - i)).pack(side="right")

# ===================== SETTINGS =====================
def show_settings():
    clear()
    tk.Label(content, text="Настройки",
             bg=BG, fg=TEXT,
             font=("Arial", 18, "bold")).pack(pady=40)

# ===================== MENU =====================
def nav(t, c):
    tk.Button(sidebar, text=t,
              bg=PANEL, fg=TEXT,
              bd=0, anchor="w",
              padx=20, pady=14,
              command=c).pack(fill="x")

nav("Калькулятор", show_calc)
nav("Профиль", show_profile)
nav("История", show_history)
nav("Настройки", show_settings)

# ===================== USER UPDATE =====================
def update_user():
    if current_user:
        user_lbl.config(text=data["users"][current_user].get("username", current_user))
    else:
        user_lbl.config(text="")

# ===================== START =====================
load_data()
show_calc()
root.mainloop()
