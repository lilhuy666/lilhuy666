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

sidebar = tk.Frame(root, bg=PANEL, width=220)
sidebar.grid(row=0, column=0, sticky="ns")

main = tk.Frame(root, bg=BG)
main.grid(row=0, column=1, sticky="nsew")

content = tk.Frame(main, bg=BG)
content.pack(fill="both", expand=True)

def clear():
    for w in content.winfo_children():
        w.destroy()

# ================= PROFILE =================
def show_profile():
    clear()

    if not current_user:
        box = tk.Frame(content, bg=CARD, padx=40, pady=40)
        box.pack(pady=100)

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
                show_profile()
            else:
                messagebox.showerror("Ошибка", "Неверно")

        def register():
            e, p = email.get(), password.get()
            if not e or not p:
                return

            data["users"][e] = {
                "password": p,
                "history": [],
                "profile": {"name": "", "phone": "", "car": ""}
            }
            save_data()
            messagebox.showinfo("OK", "Создан")

        tk.Button(box, text="Войти", bg=ACCENT, fg="white",
                  command=login).pack(fill="x", pady=5)

        tk.Button(box, text="Регистрация",
                  bg=ACCENT2, command=register).pack(fill="x")
        return

    user = data["users"][current_user]
    prof = user["profile"]

    wrap = tk.Frame(content, bg=BG)
    wrap.pack(pady=20)

    # LEFT
    left = tk.Frame(wrap, bg=CARD, padx=20, pady=20)
    left.pack(side="left", padx=20)

    tk.Label(left, text="👤", font=("Arial", 50),
             bg=CARD, fg=ACCENT).pack()

    tk.Label(left, text=current_user,
             bg=CARD, fg=TEXT).pack(pady=5)

    def logout():
        global current_user
        current_user = None
        show_profile()

    tk.Button(left, text="Выйти",
              bg=DANGER, fg="white",
              command=logout).pack(fill="x", pady=10)

    # RIGHT
    right = tk.Frame(wrap, bg=CARD, padx=30, pady=30)
    right.pack(side="left")

    tk.Label(right, text="Личная информация",
             bg=CARD, fg=TEXT).pack(anchor="w")

    name = tk.Entry(right)
    name.insert(0, prof["name"])

    phone = tk.Entry(right)
    phone.insert(0, prof["phone"])

    car = tk.Entry(right)
    car.insert(0, prof["car"])

    name.pack(fill="x", pady=5)
    phone.pack(fill="x", pady=5)
    car.pack(fill="x", pady=5)

    def save_profile():
        prof["name"] = name.get()
        prof["phone"] = phone.get()
        prof["car"] = car.get()
        save_data()
        messagebox.showinfo("OK", "Сохранено")

    tk.Button(right, text="Сохранить",
              bg=ACCENT, fg="white",
              command=save_profile).pack(pady=10)

# ================= CALC =================
def show_calc():
    clear()

    tk.Label(content, text="Калькулятор",
             bg=BG, fg=TEXT).pack(pady=10)

    f = tk.Entry(content)
    d = tk.Entry(content)
    p = tk.Entry(content)

    f.pack()
    d.pack()
    p.pack()

    result = tk.Label(content, text="", bg=BG, fg=ACCENT)
    result.pack()

    def calc():
        try:
            fuel = float(f.get())
            dist = float(d.get())
            price = float(p.get())

            cons = (fuel / dist) * 100
            cost = fuel * price

            txt = f"{cons:.1f} | {cost:.0f}"
            result.config(text=txt)

            if current_user:
                data["users"][current_user]["history"].append(txt)
                save_data()

        except:
            messagebox.showerror("Ошибка", "Ошибка ввода")

    tk.Button(content, text="Рассчитать",
              command=calc).pack()

# ================= HISTORY =================
def show_history():
    clear()

    if not current_user:
        return

    for h in data["users"][current_user]["history"]:
        tk.Label(content, text=h,
                 bg=CARD, fg=TEXT).pack(fill="x", padx=20, pady=5)

# ================= MENU =================
def nav(text, cmd):
    tk.Button(sidebar, text=text,
              bg=PANEL, fg=TEXT,
              bd=0, anchor="w",
              padx=20, pady=12,
              command=cmd).pack(fill="x")

nav("Профиль", show_profile)
nav("Калькулятор", show_calc)
nav("История", show_history)

# ================= START =================
load_data()
show_profile()
root.mainloop()
