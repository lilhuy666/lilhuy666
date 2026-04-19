import tkinter as tk
from tkinter import messagebox
import json, os
from datetime import datetime

# ===================== STYLE =====================
BG = "#f1f5f9"
HEADER = "#0f172a"
SIDEBAR = "#1e293b"
CARD = "#ffffff"
ACCENT = "#2563eb"
TEXT = "#0f172a"
SUB = "#64748b"

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
        except:
            data = {"users": {}}

def save_data():
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=2, ensure_ascii=False)

# ===================== WINDOW =====================
root = tk.Tk()
root.title("CalculatCar 🚗")
root.geometry("1200x750")
root.configure(bg=BG)

root.grid_rowconfigure(1, weight=1)
root.grid_columnconfigure(1, weight=1)

# ===================== HEADER =====================
header = tk.Frame(root, bg=HEADER, height=60)
header.grid(row=0, column=0, columnspan=2, sticky="nsew")

# ===================== SIDEBAR =====================
sidebar = tk.Frame(root, bg=SIDEBAR, width=220)
sidebar.grid(row=1, column=0, sticky="nsew")
sidebar.grid_remove()

# ===================== CONTENT =====================
content = tk.Frame(root, bg=BG)
content.grid(row=1, column=1, sticky="nsew")

# ===================== MENU =====================
menu_visible = False

def toggle_menu():
    global menu_visible
    if menu_visible:
        sidebar.grid_remove()
    else:
        sidebar.grid()
    menu_visible = not menu_visible

tk.Button(header, text="☰", bg=HEADER, fg="white",
          font=("Arial", 18), bd=0,
          command=toggle_menu).pack(side="left", padx=10)

title = tk.Label(header, text="CalculatCar",
                 bg=HEADER, fg="white",
                 font=("Arial", 18, "bold"))
title.pack(side="left", padx=10)

profile_btn = tk.Button(header, text="👤",
                        bg=HEADER, fg="white",
                        font=("Arial", 16),
                        bd=0)
profile_btn.pack(side="right", padx=10)

# ===================== HELPERS =====================
def clear():
    for w in content.winfo_children():
        w.destroy()

def card(width=400):
    f = tk.Frame(content, bg=CARD, padx=25, pady=25)
    f.pack(pady=30)
    return f

# ===================== AUTH =====================
def show_auth():
    clear()
    c = card()

    tk.Label(c, text="Авторизация",
             font=("Arial", 18, "bold"),
             bg=CARD).pack(pady=10)

    email = tk.Entry(c, font=("Arial", 12))
    password = tk.Entry(c, show="*", font=("Arial", 12))

    email.pack(fill="x", pady=5)
    password.pack(fill="x", pady=5)

    def login():
        global current_user
        e = email.get().strip()
        p = password.get().strip()

        if not e or not p:
            messagebox.showerror("Ошибка", "Заполни поля")
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
            messagebox.showerror("Ошибка", "Заполни поля")
            return

        if e in data["users"]:
            messagebox.showerror("Ошибка", "Уже есть")
            return

        data["users"][e] = {"password": p, "history": []}
        save_data()
        messagebox.showinfo("OK", "Создан аккаунт")

    tk.Button(c, text="Войти", bg=ACCENT, fg="white",
              command=login).pack(fill="x", pady=5)

    tk.Button(c, text="Регистрация",
              command=register).pack(fill="x")

profile_btn.config(command=show_auth)

# ===================== PROFILE =====================
def show_profile():
    clear()
    if not current_user:
        show_auth()
        return

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
              bg="#ef4444", fg="white",
              command=logout).pack(pady=10)

profile_btn.config(command=show_profile)

# ===================== CALC =====================
def show_calc():
    clear()

    mode = tk.StringVar(value="1")

    tk.Label(content, text="Калькулятор",
             font=("Arial", 20, "bold"),
             bg=BG).pack(pady=10)

    switch = tk.Frame(content, bg=BG)
    switch.pack()

    tk.Radiobutton(switch, text="По факту",
                   variable=mode, value="1", bg=BG).pack(side="left", padx=10)

    tk.Radiobutton(switch, text="По среднему",
                   variable=mode, value="2", bg=BG).pack(side="left", padx=10)

    form = tk.Frame(content, bg=CARD, padx=20, pady=20)
    form.pack(pady=20)

    entries = {}

    def build():
        for w in form.winfo_children():
            w.destroy()
        entries.clear()

        def add(name):
            tk.Label(form, text=name, bg=CARD).pack(anchor="w")
            e = tk.Entry(form)
            e.pack(fill="x", pady=5)
            entries[name] = e

        if mode.get() == "1":
            add("Топливо")
            add("Расстояние")
            add("Цена")
        else:
            add("Средний расход")
            add("Расстояние")
            add("Цена")

    def calc():
        try:
            if mode.get() == "1":
                f = float(entries["Топливо"].get())
                d = float(entries["Расстояние"].get())
                p = float(entries["Цена"].get())
                cons = (f/d)*100
                cost = f*p
            else:
                cons = float(entries["Средний расход"].get())
                d = float(entries["Расстояние"].get())
                p = float(entries["Цена"].get())
                f = cons*d/100
                cost = f*p

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
                      font=("Arial", 22, "bold"), bg=BG)
    result.pack()

    tk.Button(content, text="Рассчитать",
              bg=ACCENT, fg="white",
              command=calc).pack(pady=10)

    mode.trace("w", lambda *args: build())
    build()

# ===================== HISTORY =====================
def show_history():
    clear()
    if not current_user:
        show_auth()
        return

    tk.Label(content, text="История",
             font=("Arial", 18, "bold"),
             bg=BG).pack()

    for h in data["users"][current_user]["history"][::-1]:
        tk.Label(content,
                 text=f"{h['date']} | {h['result']}",
                 bg=BG).pack(anchor="w")

# ===================== SETTINGS =====================
def show_settings():
    clear()
    tk.Label(content, text="Настройки (пока пусто)",
             font=("Arial", 18),
             bg=BG).pack(pady=50)

# ===================== ABOUT =====================
def show_about():
    clear()
    tk.Label(content,
             text="CalculatCar 🚗\nВерсия 2.0\nСделано красиво 😎",
             font=("Arial", 16),
             bg=BG).pack(pady=50)

# ===================== MENU BUTTONS =====================
def menu_btn(text, cmd):
    tk.Button(sidebar, text=text,
              bg=SIDEBAR, fg="white",
              bd=0, anchor="w",
              padx=20, pady=12,
              command=cmd).pack(fill="x")

menu_btn("Калькулятор", show_calc)
menu_btn("Профиль", show_profile)
menu_btn("История", show_history)
menu_btn("Настройки", show_settings)
menu_btn("О нас", show_about)

# ===================== START =====================
load_data()
show_calc()
root.mainloop()
