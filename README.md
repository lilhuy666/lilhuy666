import tkinter as tk
from tkinter import messagebox
from datetime import datetime
import json
import os

# ===================== CONFIG =====================
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


def save_data():
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=2, ensure_ascii=False)


# ===================== UI BASE =====================
root = tk.Tk()
root.title("CalculatCar Pro 🚗")
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

header = tk.Frame(main, bg=BG)
header.pack(fill="x")

tk.Button(header, text="☰", bg=BG, fg=TEXT,
          font=("Arial", 18), bd=0,
          command=toggle_menu).pack(side="left", padx=15, pady=10)

tk.Label(header, text="CalculatCar Pro",
         bg=BG, fg=TEXT,
         font=("Arial", 18, "bold")).pack(side="left")

user_label = tk.Label(header, bg=BG, fg=SUB, font=("Arial", 12))
user_label.pack(side="right", padx=15)

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
        e, p = email.get().strip(), password.get().strip()

        if e in data["users"] and data["users"][e]["password"] == p:
            current_user = e
            update_user()
            show_calc()
        else:
            messagebox.showerror("Ошибка", "Неверные данные")

    def register():
        e, p = email.get().strip(), password.get().strip()

        if not e or not p:
            return messagebox.showerror("Ошибка", "Заполни поля")

        if e in data["users"]:
            return messagebox.showerror("Ошибка", "Уже существует")

        data["users"][e] = {"password": p, "history": []}
        save_data()
        messagebox.showinfo("OK", "Аккаунт создан")

    tk.Button(c, text="Войти", bg=ACCENT, fg="white",
              command=login).pack(fill="x", pady=10)

    tk.Button(c, text="Регистрация", bg=ACCENT2, fg="black",
              command=register).pack(fill="x")


# ===================== PROFILE =====================
def show_profile():
    clear()
    c = card()

    tk.Label(c, text="Профиль",
             bg=CARD, fg=TEXT,
             font=("Arial", 20, "bold")).pack()

    tk.Label(c, text=current_user or "",
             bg=CARD, fg=SUB).pack(pady=10)

    def logout():
        global current_user
        current_user = None
        update_user()
        show_auth()

    tk.Button(c, text="Выйти",
              bg=DANGER, fg="white",
              command=logout).pack(fill="x", pady=10)


# ===================== CALCULATOR =====================
def show_calc():
    clear()

    tk.Label(content, text="Калькулятор расхода",
             bg=BG, fg=TEXT,
             font=("Arial", 20, "bold")).pack(pady=10)

    mode = tk.StringVar(value="1")

    switch = tk.Frame(content, bg=BG)
    switch.pack(pady=10)

    tk.Radiobutton(switch, text="Стоимость поездки",
                   variable=mode, value="1",
                   indicatoron=0, width=25,
                   bg=PANEL, fg=TEXT,
                   selectcolor=ACCENT).pack(side="left", padx=5)

    tk.Radiobutton(switch, text="Расход на 100 км",
                   variable=mode, value="2",
                   indicatoron=0, width=25,
                   bg=PANEL, fg=TEXT,
                   selectcolor=ACCENT).pack(side="left", padx=5)

    form = tk.Frame(content, bg=CARD, padx=40, pady=40)
    form.pack(pady=20)

    entries = {}

    def build():
        for w in form.winfo_children():
            w.destroy()
        entries.clear()

        def add(label):
            tk.Label(form, text=label, bg=CARD, fg=TEXT).pack(anchor="w")
            e = tk.Entry(form)
            e.pack(fill="x", pady=5)
            entries[label] = e

        if mode.get() == "1":
            add("Топливо")
            add("Расстояние")
            add("Цена")
        else:
            add("Расход")
            add("Расстояние")
            add("Цена")

    def calc():
        try:
            if mode.get() == "1":
                f = float(entries["Топливо"].get())
                d = float(entries["Расстояние"].get())
                p = float(entries["Цена"].get())
                cons = (f / d) * 100
            else:
                cons = float(entries["Расход"].get())
                d = float(entries["Расстояние"].get())
                p = float(entries["Цена"].get())
                f = cons * d / 100

            cost = f * p
            text = f"{cons:.1f} л/100км | {cost:.0f} ₽"
            result.config(text=text)

            if current_user:
                data["users"][current_user]["history"].append({
                    "date": datetime.now().strftime("%d.%m %H:%M"),
                    "result": text
                })
                save_data()

        except:
            messagebox.showerror("Ошибка", "Проверь ввод")

    result = tk.Label(content, text="—",
                      bg=BG, fg=ACCENT,
                      font=("Arial", 20, "bold"))
    result.pack()

    tk.Button(content, text="Рассчитать",
              bg=ACCENT, fg="white",
              command=calc).pack(pady=10)

    mode.trace("w", lambda *args: build())
    build()


# ===================== HISTORY =====================
def show_history():
    clear()

    tk.Label(content, text="История",
             bg=BG, fg=TEXT,
             font=("Arial", 20, "bold")).pack(pady=10)

    if not current_user:
        return

    for h in reversed(data["users"][current_user]["history"]):
        tk.Label(content,
                 text=f"{h['date']} | {h['result']}",
                 bg=CARD, fg=TEXT).pack(fill="x", padx=40, pady=5)


# ===================== OTHER =====================
def show_settings():
    clear()
    tk.Label(content, text="Настройки",
             bg=BG, fg=TEXT).pack(pady=40)


def show_about():
    clear()
    tk.Label(content,
             text="CalculatCar Pro 🚗\nПриложение для расчёта расхода",
             bg=BG, fg=TEXT).pack(pady=60)


def update_user():
    user_label.config(text=current_user or "")


# ===================== MENU =====================
def nav(text, cmd):
    tk.Button(sidebar, text=text,
              bg=PANEL, fg=TEXT,
              bd=0, anchor="w",
              padx=20, pady=12,
              command=cmd).pack(fill="x")


nav("Калькулятор", show_calc)
nav("Профиль", show_profile)
nav("История", show_history)
nav("Настройки", show_settings)
nav("О программе", show_about)

# ===================== START =====================
load_data()
show_calc()
root.mainloop()
