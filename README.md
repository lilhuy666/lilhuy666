import tkinter as tk
from tkinter import messagebox
from datetime import datetime
import json
import os

# ===============================
# 🎨 STYLE
# ===============================
BG = "#eef2f7"
SIDEBAR = "#1e293b"
HEADER = "#334155"
CARD = "#ffffff"
BLUE = "#3b82f6"
RED = "#ef4444"
TEXT = "#0f172a"
SUB = "#64748b"

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
        json.dump(data, f, indent=2, ensure_ascii=False)

# ===============================
# 🪟 WINDOW
# ===============================
root = tk.Tk()
root.title("CalculatCar 🚗")
root.geometry("1200x750")
root.configure(bg=BG)

# ===============================
# LAYOUT (ВАЖНО)
# ===============================
root.grid_rowconfigure(1, weight=1)
root.grid_columnconfigure(1, weight=1)

# HEADER
header = tk.Frame(root, bg=HEADER, height=60)
header.grid(row=0, column=0, columnspan=2, sticky="nsew")

# SIDEBAR
sidebar = tk.Frame(root, bg=SIDEBAR, width=220)
sidebar.grid(row=1, column=0, sticky="nsew")

# CONTENT
content = tk.Frame(root, bg=BG)
content.grid(row=1, column=1, sticky="nsew")

# скрываем меню при старте
sidebar.grid_remove()
menu_visible = False

def toggle_menu():
    global menu_visible
    if menu_visible:
        sidebar.grid_remove()
    else:
        sidebar.grid()
    menu_visible = not menu_visible

# HEADER UI
tk.Button(header, text="☰", bg=HEADER, fg="white",
          font=("Arial", 18), bd=0,
          command=toggle_menu).pack(side="left", padx=10)

tk.Label(header, text="CalculatCar",
         bg=HEADER, fg="white",
         font=("Arial", 18, "bold")).pack(pady=10)

# ===============================
# HELPERS
# ===============================
def clear():
    for w in content.winfo_children():
        w.destroy()

def card():
    frame = tk.Frame(content, bg=CARD, padx=25, pady=25)
    frame.pack(pady=40)
    return frame

# ===============================
# AUTH
# ===============================
def show_auth():
    clear()

    c = card()

    tk.Label(c, text="Вход / Регистрация",
             font=("Arial", 18, "bold"),
             bg=CARD).pack(pady=10)

    email = tk.Entry(c, font=("Arial", 12))
    password = tk.Entry(c, show="*", font=("Arial", 12))

    email.pack(pady=5, fill="x")
    password.pack(pady=5, fill="x")

    def login():
        global current_user
        e = email.get().strip()
        p = password.get().strip()

        if not e or not p:
            messagebox.showerror("Ошибка", "Заполни все поля")
            return

        if e in data["users"] and data["users"][e]["password"] == p:
            current_user = e
            show_profile()
        else:
            messagebox.showerror("Ошибка", "Неверные данные")

    def register():
        e = email.get().strip()
        p = password.get().strip()

        if not e or not p:
            messagebox.showerror("Ошибка", "Заполни все поля")
            return

        if e in data["users"]:
            messagebox.showerror("Ошибка", "Уже существует")
            return

        data["users"][e] = {
            "password": p,
            "history": []
        }
        save_data()
        messagebox.showinfo("OK", "Аккаунт создан")

    tk.Button(c, text="Войти",
              bg=BLUE, fg="white",
              command=login).pack(fill="x", pady=5)

    tk.Button(c, text="Регистрация",
              command=register).pack(fill="x")

# ===============================
# PROFILE
# ===============================
def show_profile():
    if not current_user:
        show_auth()
        return

    clear()

    c = card()

    tk.Label(c, text="Профиль",
             font=("Arial", 18, "bold"),
             bg=CARD).pack()

    tk.Label(c, text=current_user,
             bg=CARD, fg=SUB).pack(pady=5)

    def logout():
        global current_user
        current_user = None
        show_calc()

    tk.Button(c, text="Выйти",
              bg=RED, fg="white",
              command=logout).pack(pady=10)

# ===============================
# CALCULATOR
# ===============================
def show_calc():
    clear()

    mode = tk.StringVar(value="1")

    tk.Label(content, text="Калькулятор",
             font=("Arial", 20, "bold"),
             bg=BG).pack(pady=10)

    switch = tk.Frame(content, bg=BG)
    switch.pack()

    tk.Radiobutton(switch, text="По факту",
                   variable=mode, value="1",
                   bg=BG).pack(side="left", padx=10)

    tk.Radiobutton(switch, text="По среднему",
                   variable=mode, value="2",
                   bg=BG).pack(side="left", padx=10)

    c = tk.Frame(content, bg=CARD, padx=20, pady=20)
    c.pack(pady=20)

    entries = {}

    def build():
        for w in c.winfo_children():
            w.destroy()
        entries.clear()

        def add(label):
            tk.Label(c, text=label, bg=CARD).pack(anchor="w")
            e = tk.Entry(c)
            e.pack(fill="x", pady=5)
            entries[label] = e

        if mode.get() == "1":
            add("Топливо (л)")
            add("Расстояние (км)")
            add("Цена (₽)")
        else:
            add("Средний расход")
            add("Расстояние (км)")
            add("Цена (₽)")

    def calc():
        try:
            if mode.get() == "1":
                f = float(entries["Топливо (л)"].get())
                d = float(entries["Расстояние (км)"].get())
                p = float(entries["Цена (₽)"].get())
                cons = (f / d) * 100
                cost = f * p
            else:
                cons = float(entries["Средний расход"].get())
                d = float(entries["Расстояние (км)"].get())
                p = float(entries["Цена (₽)"].get())
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
                      font=("Arial", 20, "bold"),
                      bg=BG)
    result.pack(pady=10)

    tk.Button(content, text="Рассчитать",
              bg=BLUE, fg="white",
              command=calc).pack()

    mode.trace("w", lambda *args: build())
    build()

# ===============================
# HISTORY
# ===============================
def show_history():
    clear()

    if not current_user:
        show_auth()
        return

    for h in data["users"][current_user]["history"][::-1]:
        tk.Label(content,
                 text=f"{h['date']}  |  {h['result']}",
                 bg=BG).pack(anchor="w")

# ===============================
# MENU
# ===============================
def menu_btn(text, cmd):
    tk.Button(sidebar, text=text,
              bg=SIDEBAR, fg="white",
              bd=0, anchor="w",
              padx=20, pady=12,
              command=cmd).pack(fill="x")

menu_btn("Калькулятор", show_calc)
menu_btn("Профиль", show_profile)
menu_btn("История", show_history)
menu_btn("Вход", show_auth)

# ===============================
# START
# ===============================
load_data()
show_calc()
root.mainloop()
