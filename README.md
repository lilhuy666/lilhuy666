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

menu_visible = False  # ❗ скрыто при старте

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

main = tk.Frame(root, bg=BG)
main.pack(fill="both", expand=True)

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

    tk.Label(content, text=f"Профиль: {current_user}",
             font=("Arial", 18, "bold"), bg=BG).pack()

    def logout():
        global current_user
        current_user = None
        show_calc()

    tk.Button(content, text="Выйти",
              bg=RED, fg="white",
              command=logout).pack(pady=10)

# ===============================
# 🧮 CALC
# ===============================
def show_calc():
    clear()

    mode = tk.StringVar(value="1")

    tk.Label(content, text="Калькулятор",
             font=("Arial", 18, "bold"), bg=BG).pack()

    tk.Radiobutton(content, text="Расход по факту",
                   variable=mode, value="1",
                   bg=BG).pack(anchor="w")

    tk.Radiobutton(content, text="Расчет по среднему расходу",
                   variable=mode, value="2",
                   bg=BG).pack(anchor="w")

    frame = tk.Frame(content, bg=CARD, padx=20, pady=20)
    frame.pack(pady=20)

    entries = {}

    def build_fields():
        for w in frame.winfo_children():
            w.destroy()

        entries.clear()

        def add(name):
            tk.Label(frame, text=name, bg=CARD).pack(anchor="w")
            e = tk.Entry(frame)
            e.pack(fill="x", pady=5)
            entries[name] = e

        if mode.get() == "1":
            add("Топливо (л)")
            add("Расстояние (км)")
            add("Цена (₽)")
        else:
            add("Средний расход (л/100км)")
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
                cons = float(entries["Средний расход (л/100км)"].get())
                d = float(entries["Расстояние (км)"].get())
                p = float(entries["Цена (₽)"].get())

                f = (cons * d) / 100
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
                      font=("Arial", 20, "bold"), bg=BG)
    result.pack(pady=10)

    tk.Button(content, text="Рассчитать",
              bg=BLUE, fg="white",
              command=calc).pack()

    mode.trace("w", lambda *args: build_fields())

    build_fields()

# ===============================
# 📊 HISTORY
# ===============================
def show_history():
    clear()

    if not current_user:
        show_auth()
        return

    for h in data["users"][current_user]["history"][::-1]:
        tk.Label(content, text=f"{h['date']} | {h['result']}",
                 bg=BG).pack(anchor="w")

# ===============================
# MENU
# ===============================
def menu_btn(text, cmd):
    tk.Button(sidebar, text=text,
              bg=SIDEBAR, fg="white",
              bd=0, anchor="w",
              padx=20, pady=10,
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
