import tkinter as tk
from tkinter import messagebox
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

DATA_FILE = "data.json"

data = {"users": {}}
current_user = None

# ===============================
# 💾 LOAD / SAVE
# ===============================
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
root.geometry("900x600")
root.configure(bg=BG)

# ===============================
# CLEAR
# ===============================
def clear():
    for w in root.winfo_children():
        w.destroy()

# ===============================
# 🔐 REGISTER WINDOW
# ===============================
def open_register():
    reg = tk.Toplevel(root)
    reg.title("Регистрация")
    reg.geometry("300x300")

    tk.Label(reg, text="Логин").pack()
    login = tk.Entry(reg)
    login.pack()

    tk.Label(reg, text="Почта").pack()
    email = tk.Entry(reg)
    email.pack()

    tk.Label(reg, text="Пароль").pack()
    password = tk.Entry(reg, show="*")
    password.pack()

    tk.Label(reg, text="Повторите пароль").pack()
    password2 = tk.Entry(reg, show="*")
    password2.pack()

    def register():
        if password.get() != password2.get():
            messagebox.showerror("Ошибка", "Пароли не совпадают")
            return

        if login.get() in data["users"]:
            messagebox.showerror("Ошибка", "Пользователь уже есть")
            return

        data["users"][login.get()] = {
            "email": email.get(),
            "password": password.get()
        }

        save_data()
        messagebox.showinfo("OK", "Аккаунт создан")
        reg.destroy()

    tk.Button(reg, text="Создать аккаунт",
              command=register).pack(pady=10)

# ===============================
# 🔐 LOGIN SCREEN
# ===============================
def show_login():
    clear()

    card = tk.Frame(root, bg=CARD, padx=30, pady=30)
    card.pack(pady=100)

    tk.Label(card, text="Вход",
             bg=CARD, font=("Arial", 18, "bold")).pack(pady=10)

    tk.Label(card, text="Логин", bg=CARD).pack(anchor="w")
    login = tk.Entry(card)
    login.pack()

    tk.Label(card, text="Пароль", bg=CARD).pack(anchor="w")
    password = tk.Entry(card, show="*")
    password.pack()

    def login_user():
        global current_user
        if login.get() in data["users"]:
            if data["users"][login.get()]["password"] == password.get():
                current_user = login.get()
                show_profile()
            else:
                messagebox.showerror("Ошибка", "Неверный пароль")
        else:
            messagebox.showerror("Ошибка", "Нет такого пользователя")

    tk.Button(card, text="Войти",
              bg=BLUE, fg="white",
              command=login_user).pack(pady=10, fill="x")

    tk.Button(card, text="Регистрация",
              command=open_register).pack(fill="x")

# ===============================
# 👤 PROFILE
# ===============================
def show_profile():
    clear()

    tk.Label(root, text=f"Вы вошли как: {current_user}",
             bg=BG, font=("Arial", 16)).pack(pady=50)

    def logout():
        global current_user
        current_user = None
        show_login()

    tk.Button(root, text="Выйти",
              command=logout).pack()

# ===============================
# START
# ===============================
load_data()
show_login()
root.mainloop()
