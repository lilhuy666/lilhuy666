import tkinter as tk
from tkinter import ttk, messagebox
from datetime import datetime
import json, os, hashlib, secrets

# ===================== THEME =====================
BG = "#0f172a"
CARD = "#1e293b"
CARD2 = "#111827"
ACCENT = "#3b82f6"
ACCENT_HOVER = "#2563eb"
TEXT = "#e2e8f0"
SUB = "#94a3b8"
DANGER = "#ef4444"

FONT = ("Segoe UI", 11)
FONT_BOLD = ("Segoe UI", 12, "bold")
FONT_BIG = ("Segoe UI", 20, "bold")

DATA_FILE = "data.json"

data = {"users": {}}
current_user = None


# ===================== UTILS =====================
def hash_password(password, salt=None):
    if salt is None:
        salt = secrets.token_hex(16)
    hashed = hashlib.pbkdf2_hmac('sha256', password.encode(), salt.encode(), 100000)
    return f"{salt}:{hashed.hex()}"


def verify_password(stored, provided):
    salt, _ = stored.split(":")
    return hash_password(provided, salt) == stored


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


# ===================== APP =====================
root = tk.Tk()
root.title("CalculatCar")
root.geometry("1100x700")
root.configure(bg=BG)

style = ttk.Style()
style.theme_use("default")

# ttk styling
style.configure("TEntry",
                fieldbackground=CARD2,
                foreground=TEXT,
                borderwidth=0,
                padding=8)

# ===================== HELPERS =====================
def hover(btn):
    btn.bind("<Enter>", lambda e: btn.config(bg=ACCENT_HOVER))
    btn.bind("<Leave>", lambda e: btn.config(bg=ACCENT))


def card(parent):
    frame = tk.Frame(parent, bg=CARD, padx=25, pady=25)
    frame.pack(pady=20)
    return frame


def clear():
    for w in content.winfo_children():
        w.destroy()


# ===================== HEADER =====================
header = tk.Frame(root, bg=CARD2, height=60)
header.pack(fill="x")

title = tk.Label(header, text="🚗 CalculatCar", bg=CARD2, fg=TEXT, font=FONT_BIG)
title.pack(side="left", padx=20)

user_label = tk.Label(header, text="", bg=CARD2, fg=SUB, font=FONT)
user_label.pack(side="right", padx=20)


def update_user():
    user_label.config(text=current_user if current_user else "")


# ===================== CONTENT =====================
content = tk.Frame(root, bg=BG)
content.pack(fill="both", expand=True)


# ===================== BUTTON =====================
def button(parent, text, cmd, danger=False):
    bg = DANGER if danger else ACCENT
    b = tk.Button(parent,
                  text=text,
                  command=cmd,
                  bg=bg,
                  fg="white",
                  font=FONT_BOLD,
                  bd=0,
                  padx=10,
                  pady=8,
                  activebackground=ACCENT_HOVER)
    hover(b)
    return b


# ===================== PROFILE =====================
def show_profile():
    clear()

    if not current_user:
        c = card(content)

        tk.Label(c, text="Вход", bg=CARD, fg=TEXT, font=FONT_BIG).pack(pady=10)

        email = ttk.Entry(c)
        password = ttk.Entry(c, show="*")

        email.pack(fill="x", pady=5)
        password.pack(fill="x", pady=5)

        def login():
            global current_user
            e, p = email.get(), password.get()

            if e in data["users"] and verify_password(data["users"][e]["password"], p):
                current_user = e
                update_user()
                show_profile()
            else:
                messagebox.showerror("Ошибка", "Неверные данные")

        def register():
            e, p = email.get(), password.get()
            if not e or not p:
                return

            data["users"][e] = {
                "password": hash_password(p),
                "history": []
            }
            save_data()
            messagebox.showinfo("OK", "Аккаунт создан")

        button(c, "Войти", login).pack(fill="x", pady=10)
        button(c, "Регистрация", register).pack(fill="x")

        return

    c = card(content)

    tk.Label(c, text=f"👤 {current_user}", bg=CARD, fg=TEXT, font=FONT_BIG).pack(pady=10)

    button(c, "Выйти", logout, True).pack(fill="x")


def logout():
    global current_user
    current_user = None
    update_user()
    show_profile()


# ===================== CALCULATOR =====================
def show_calc():
    clear()

    c = card(content)

    tk.Label(c, text="Калькулятор топлива", bg=CARD, fg=TEXT, font=FONT_BIG).pack(pady=10)

    fuel = ttk.Entry(c)
    dist = ttk.Entry(c)
    price = ttk.Entry(c)

    for label, entry in [
        ("Топливо (л)", fuel),
        ("Расстояние (км)", dist),
        ("Цена за литр", price)
    ]:
        tk.Label(c, text=label, bg=CARD, fg=SUB).pack(anchor="w")
        entry.pack(fill="x", pady=5)

    result = tk.Label(c, text="—", bg=CARD, fg=ACCENT, font=("Segoe UI", 18, "bold"))
    result.pack(pady=10)

    def calc():
        try:
            f = float(fuel.get())
            d = float(dist.get())
            p = float(price.get())

            cons = (f / d) * 100
            cost = f * p

            result.config(text=f"{cons:.1f} л/100км | {cost:.2f} ₽")

        except:
            messagebox.showerror("Ошибка", "Введите числа")

    button(c, "Рассчитать", calc).pack(fill="x")


# ===================== HISTORY =====================
def show_history():
    clear()

    c = card(content)

    tk.Label(c, text="История", bg=CARD, fg=TEXT, font=FONT_BIG).pack()

    if not current_user:
        tk.Label(c, text="Войдите", bg=CARD, fg=SUB).pack()
        return

    for item in data["users"][current_user]["history"][-10:]:
        tk.Label(c,
                 text=f"{item['distance']} км | {item['total_cost']:.2f} ₽",
                 bg=CARD,
                 fg=TEXT).pack(anchor="w")


# ===================== MENU =====================
menu = tk.Frame(root, bg=CARD2, width=200)
menu.pack(side="left", fill="y")


def nav(text, cmd):
    b = tk.Button(menu,
                  text=text,
                  command=cmd,
                  bg=CARD2,
                  fg=TEXT,
                  bd=0,
                  anchor="w",
                  padx=15,
                  pady=10,
                  font=FONT)
    b.pack(fill="x")
    return b


nav("Калькулятор", show_calc)
nav("Профиль", show_profile)
nav("История", show_history)


# ===================== START =====================
load_data()
update_user()
show_calc()

root.mainloop()
