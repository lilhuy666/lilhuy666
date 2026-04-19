import tkinter as tk
from tkinter import messagebox
import json, os
from datetime import datetime

# ================= CONFIG =================
BG = "#0b1220"
PANEL = "#111a2e"
CARD = "#162238"
ACCENT = "#4f8cff"
ACCENT2 = "#22c55e"
TEXT = "#e5e7eb"
SUB = "#94a3b8"
DANGER = "#ef4444"

DATA_FILE = "data.json"

data = {"users": {}}
current_user = None

# ================= DATA =================
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

# ================= UI =================
root = tk.Tk()
root.title("CalculatCar")
root.geometry("1200x750")
root.configure(bg=BG)

root.grid_columnconfigure(1, weight=1)
root.grid_rowconfigure(0, weight=1)

# SIDEBAR
sidebar = tk.Frame(root, bg=PANEL, width=220)
sidebar.grid(row=0, column=0, sticky="ns")

# MAIN
main = tk.Frame(root, bg=BG)
main.grid(row=0, column=1, sticky="nsew")

header = tk.Frame(main, bg=BG)
header.pack(fill="x")

user_label = tk.Label(header, bg=BG, fg=SUB)
user_label.pack(side="right", padx=15)

content = tk.Frame(main, bg=BG)
content.pack(fill="both", expand=True)

def clear():
    for w in content.winfo_children():
        w.destroy()

def update_user():
    user_label.config(text=current_user or "")

# ================= PROFILE =================
def show_profile():
    clear()

    # ЕСЛИ НЕ ВОШЕЛ → ПОКАЗАТЬ ВХОД
    if not current_user:
        box = tk.Frame(content, bg=CARD, padx=40, pady=40)
        box.pack(pady=80)

        tk.Label(box, text="Вход / Регистрация",
                 bg=CARD, fg=TEXT,
                 font=("Arial", 18, "bold")).pack(pady=10)

        email = tk.Entry(box)
        password = tk.Entry(box, show="*")

        email.pack(fill="x", pady=5)
        password.pack(fill="x", pady=5)

        def login():
            global current_user
            e, p = email.get(), password.get()

            if e in data["users"] and data["users"][e]["password"] == p:
                current_user = e
                update_user()
                show_profile()
            else:
                messagebox.showerror("Ошибка", "Неверные данные")

        def register():
            e, p = email.get(), password.get()

            if not e or not p:
                return messagebox.showerror("Ошибка", "Заполни поля")

            if e in data["users"]:
                return messagebox.showerror("Ошибка", "Уже есть")

            data["users"][e] = {"password": p, "history": []}
            save_data()
            messagebox.showinfo("OK", "Создан")

        tk.Button(box, text="Войти",
                  bg=ACCENT, fg="white",
                  command=login).pack(fill="x", pady=5)

        tk.Button(box, text="Регистрация",
                  bg=ACCENT2,
                  command=register).pack(fill="x")

        return

    # ================= ПРОФИЛЬ =================
    wrapper = tk.Frame(content, bg=BG)
    wrapper.pack(pady=20, fill="x")

    # ЛЕВАЯ КАРТОЧКА
    left = tk.Frame(wrapper, bg=CARD, padx=20, pady=20)
    left.pack(side="left", padx=20)

    tk.Label(left, text="👤",
             font=("Arial", 50),
             bg=CARD, fg=ACCENT).pack()

    tk.Label(left, text=current_user,
             bg=CARD, fg=TEXT,
             font=("Arial", 14, "bold")).pack(pady=5)

    tk.Button(left, text="Выйти",
              bg=DANGER, fg="white",
              command=logout).pack(fill="x", pady=10)

    # ПРАВАЯ ЧАСТЬ
    right = tk.Frame(wrapper, bg=CARD, padx=30, pady=30)
    right.pack(side="left", padx=10)

    tk.Label(right, text="Личная информация",
             bg=CARD, fg=TEXT,
             font=("Arial", 16, "bold")).pack(anchor="w", pady=10)

    name = tk.Entry(right)
    name.insert(0, "Имя")

    phone = tk.Entry(right)
    phone.insert(0, "+7")

    name.pack(fill="x", pady=5)
    phone.pack(fill="x", pady=5)

    tk.Label(right, text="Автомобиль",
             bg=CARD, fg=TEXT,
             font=("Arial", 14)).pack(anchor="w", pady=10)

    car = tk.Entry(right)
    car.insert(0, "Toyota Camry")

    car.pack(fill="x", pady=5)

def logout():
    global current_user
    current_user = None
    update_user()
    show_profile()

# ================= CALC =================
def show_calc():
    clear()

    tk.Label(content, text="Калькулятор",
             bg=BG, fg=TEXT,
             font=("Arial", 18)).pack(pady=20)

# ================= MENU =================
def nav(text, cmd):
    tk.Button(sidebar, text=text,
              bg=PANEL, fg=TEXT,
              bd=0, anchor="w",
              padx=20, pady=12,
              command=cmd).pack(fill="x")

nav("Профиль", show_profile)
nav("Калькулятор", show_calc)

# ================= START =================
load_data()
show_profile()
root.mainloop()
