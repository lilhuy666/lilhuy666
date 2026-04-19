import tkinter as tk
from tkinter import messagebox
from datetime import datetime
import json
import os

# ===============================
# 🎨 COLORS
# ===============================
BG = "#eef1f5"
SIDEBAR = "#1f4aa8"
HEADER = "#2f5fb3"
CARD = "#ffffff"
TEXT = "#1f2937"
SUB = "#6b7280"
BLUE = "#3b82f6"
RED = "#ef4444"

DATA_FILE = "data.json"

# ===============================
# 💾 DATA
# ===============================
data = {"users": {}}
current_user = None

def load_data():
    global data
    if os.path.exists(DATA_FILE):
        with open(DATA_FILE, "r", encoding="utf-8") as f:
            data = json.load(f)

def save_data():
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

# ===============================
# 🪟 WINDOW
# ===============================
root = tk.Tk()
root.title("CalculatCar")
root.geometry("1200x750")
root.configure(bg=BG)

# ===============================
# HEADER
# ===============================
header = tk.Frame(root, bg=HEADER, height=60)
header.pack(fill="x")

menu_visible = True

def toggle_menu():
    global menu_visible
    if menu_visible:
        sidebar.pack_forget()
    else:
        sidebar.pack(side="left", fill="y")
    menu_visible = not menu_visible

tk.Button(header, text="☰", bg=HEADER, fg="white",
          font=("Arial", 18), bd=0,
          command=toggle_menu).pack(side="left", padx=10)

tk.Label(header, text="CalculatCar",
         bg=HEADER, fg="white",
         font=("Arial", 20, "bold")).pack(pady=10)

# ===============================
# SIDEBAR
# ===============================
sidebar = tk.Frame(root, bg=SIDEBAR, width=220)
sidebar.pack(side="left", fill="y")

main = tk.Frame(root, bg=BG)
main.pack(side="right", fill="both", expand=True)

content = tk.Frame(main, bg=BG)
content.pack(fill="both", expand=True, padx=20, pady=20)

def clear():
    for w in content.winfo_children():
        w.destroy()

# ===============================
# 🔐 AUTH
# ===============================
def show_auth():
    clear()

    card = tk.Frame(content, bg=CARD, padx=30, pady=30)
    card.pack(pady=80)

    tk.Label(card, text="Вход / Регистрация",
             bg=CARD, font=("Arial", 18, "bold")).pack(pady=10)

    login = tk.Entry(card)
    password = tk.Entry(card, show="*")

    login.pack(pady=5)
    password.pack(pady=5)

    def login_user():
        global current_user
        if login.get() in data["users"]:
            if data["users"][login.get()]["password"] == password.get():
                current_user = login.get()
                show_profile()
            else:
                messagebox.showerror("Ошибка", "Неверный пароль")
        else:
            messagebox.showerror("Ошибка", "Пользователь не найден")

    def register():
        if login.get() in data["users"]:
            messagebox.showerror("Ошибка", "Уже существует")
            return
        data["users"][login.get()] = {
            "password": password.get(),
            "profile": {"name": "", "email": "", "car": ""},
            "history": []
        }
        save_data()
        messagebox.showinfo("OK", "Аккаунт создан")

    tk.Button(card, text="Войти", bg=BLUE, fg="white",
              command=login_user).pack(pady=5, fill="x")

    tk.Button(card, text="Регистрация",
              command=register).pack(pady=5, fill="x")

# ===============================
# 👤 PROFILE
# ===============================
def show_profile():
    if not current_user:
        show_auth()
        return

    clear()
    user = data["users"][current_user]

    left = tk.Frame(content, bg=CARD, padx=20, pady=20)
    left.grid(row=0, column=0, padx=10, pady=10)

    tk.Label(left, text="👤", font=("Arial", 40), bg=CARD).pack()
    tk.Label(left, text=current_user,
             bg=CARD, font=("Arial", 14, "bold")).pack()

    def logout():
        global current_user
        current_user = None
        show_calc()

    tk.Button(left, text="Выйти",
              bg=RED, fg="white",
              command=logout).pack(pady=10, fill="x")

    right = tk.Frame(content, bg=CARD, padx=20, pady=20)
    right.grid(row=0, column=1, padx=10, pady=10)

    tk.Label(right, text="Личная информация",
             bg=CARD, font=("Arial", 16, "bold")).pack(anchor="w")

    def field(label, value):
        tk.Label(right, text=label, bg=CARD, fg=SUB).pack(anchor="w")
        e = tk.Entry(right)
        e.insert(0, value)
        e.pack(fill="x", pady=5)
        return e

    name = field("Имя", user["profile"]["name"])
    email = field("Email", user["profile"]["email"])
    car = field("Автомобиль", user["profile"]["car"])

    def save():
        user["profile"]["name"] = name.get()
        user["profile"]["email"] = email.get()
        user["profile"]["car"] = car.get()
        save_data()
        messagebox.showinfo("OK", "Сохранено")

    tk.Button(right, text="Сохранить",
              bg=BLUE, fg="white",
              command=save).pack(pady=10)

# ===============================
# 🧮 CALC
# ===============================
def show_calc():
    clear()

    left = tk.Frame(content, bg=CARD, padx=20, pady=20)
    left.pack(side="left", fill="both", expand=True, padx=10)

    tk.Label(left, text="Калькулятор",
             bg=CARD, font=("Arial", 18, "bold")).pack(anchor="w")

    def field(text):
        tk.Label(left, text=text, bg=CARD, fg=SUB).pack(anchor="w")
        e = tk.Entry(left)
        e.pack(fill="x", pady=5)
        return e

    distance = field("Дистанция")
    fuel = field("Топливо")
    price = field("Цена")

    result = tk.Label(left, text="—", bg=CARD,
                      font=("Arial", 20, "bold"))
    result.pack(pady=20)

    def calc():
        try:
            d = float(distance.get())
            f = float(fuel.get())
            p = float(price.get())

            cons = (f / d) * 100
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

    tk.Button(left, text="Рассчитать",
              bg=BLUE, fg="white",
              command=calc).pack(fill="x")

# ===============================
# 📊 HISTORY
# ===============================
def show_history():
    clear()

    if not current_user:
        messagebox.showinfo("Внимание", "Сначала войдите")
        return

    hist = data["users"][current_user]["history"]

    for i, h in enumerate(hist[::-1]):
        row = tk.Frame(content, bg=CARD, pady=10)
        row.pack(fill="x", padx=10, pady=5)

        tk.Label(row, text=h["date"], bg=CARD).pack(side="left")
        tk.Label(row, text=h["result"], bg=CARD).pack(side="left", padx=20)

# ===============================
# ⚙ SETTINGS
# ===============================
def show_settings():
    clear()

    tk.Label(content, text="Настройки",
             bg=BG, font=("Arial", 18, "bold")).pack()

# ===============================
# ℹ️ ABOUT
# ===============================
def show_about():
    clear()
    tk.Label(content, text="О программе\nCalculatCar 🚗",
             bg=BG, font=("Arial", 16)).pack(pady=50)

# ===============================
# MENU
# ===============================
def menu_btn(text, cmd):
    tk.Button(sidebar, text=text,
              bg=SIDEBAR, fg="white",
              font=("Arial", 13),
              bd=0, anchor="w",
              padx=20, pady=12,
              command=cmd).pack(fill="x")

menu_btn("Калькулятор", show_calc)
menu_btn("Профиль", show_profile)
menu_btn("История", show_history)
menu_btn("О нас", show_about)

# вниз меню
tk.Label(sidebar, bg=SIDEBAR).pack(expand=True)
menu_btn("Настройки", show_settings)

# ===============================
# START
# ===============================
load_data()
show_calc()
root.mainloop()
