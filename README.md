```python
import tkinter as tk
from tkinter import messagebox
from datetime import datetime
import json, os

# ===================== STYLE =====================
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

# ===================== MAIN =====================
main = tk.Frame(root, bg=BG)
main.grid(row=0, column=0, sticky="nsew")

# ===================== HEADER =====================
header = tk.Frame(main, bg=ACCENT, height=70)
header.pack(fill="x")
header.pack_propagate(False)

menu_open = False

# ===================== DROPDOWN MENU =====================
dropdown = tk.Frame(main, bg=PANEL)
dropdown.pack(fill="x")
dropdown.pack_forget()

def toggle_menu():
    global menu_open
    if menu_open:
        dropdown.pack_forget()
    else:
        dropdown.pack(fill="x")
    menu_open = not menu_open

tk.Button(header,
          text="☰",
          bg=ACCENT,
          fg="white",
          font=("Arial", 16, "bold"),
          bd=0,
          activebackground=ACCENT,
          command=toggle_menu).pack(side="left", padx=15)

title = tk.Label(header,
                 text="CalculatCar Pro 🚗",
                 bg=ACCENT,
                 fg="white",
                 font=("Arial", 20, "bold"))
title.pack(side="left")

user_label = tk.Label(header,
                      text="",
                      bg=ACCENT,
                      fg="white",
                      font=("Arial", 11))
user_label.pack(side="right", padx=15)

# ===================== CONTENT =====================
content = tk.Frame(main, bg=BG)
content.pack(fill="both", expand=True)

def clear():
    for w in content.winfo_children():
        w.destroy()

def card():
    f = tk.Frame(content, bg=CARD, padx=40, pady=40)
    f.pack(pady=30)
    return f

# ===================== MENU =====================
def nav(text, cmd):
    tk.Button(dropdown,
              text=text,
              bg=PANEL,
              fg=TEXT,
              bd=0,
              anchor="w",
              padx=20,
              pady=12,
              font=("Arial", 12),
              activebackground=CARD,
              activeforeground=ACCENT,
              command=lambda: [cmd(), toggle_menu()]
              ).pack(fill="x")

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

        data["users"][e] = {"password": p, "history": []}
        save_data()
        messagebox.showinfo("OK", "Аккаунт создан")

    tk.Button(c, text="Войти",
              bg=ACCENT, fg="white",
              font=("Arial", 12, "bold"),
              command=login).pack(fill="x", pady=10)

    tk.Button(c, text="Регистрация",
              bg=ACCENT2, fg="black",
              font=("Arial", 12, "bold"),
              command=register).pack(fill="x")

# ===================== PROFILE =====================
def show_profile():
    clear()
    c = card()

    tk.Label(c, text="Профиль",
             bg=CARD, fg=TEXT,
             font=("Arial", 20, "bold")).pack()

    tk.Label(c, text=current_user or "",
             bg=CARD, fg=SUB,
             font=("Arial", 14)).pack(pady=10)

    def logout():
        global current_user
        current_user = None
        update_user()
        show_auth()

    tk.Button(c, text="Выйти",
              bg=DANGER, fg="white",
              command=logout).pack(fill="x", pady=10)

# ===================== CALC =====================
def show_calc():
    clear()

    tk.Label(content, text="Калькулятор расхода",
             bg=BG, fg=TEXT,
             font=("Arial", 20, "bold")).pack(pady=10)

    mode = tk.StringVar(value="1")

    switch = tk.Frame(content, bg=BG)
    switch.pack(pady=10)

    tk.Radiobutton(switch, text="Рассчитать стоимость поездки",
                   variable=mode, value="1", indicatoron=0,
                   width=28, bg=PANEL, fg=TEXT,
                   selectcolor=ACCENT,
                   font=("Arial", 11)).pack(side="left", padx=5)

    tk.Radiobutton(switch, text="Средний расход на 100 км",
                   variable=mode, value="2", indicatoron=0,
                   width=28, bg=PANEL, fg=TEXT,
                   selectcolor=ACCENT,
                   font=("Arial", 11)).pack(side="left", padx=5)

    form = tk.Frame(content, bg=CARD, padx=40, pady=40)
    form.pack(pady=20)

    entries = {}

    def build():
        for w in form.winfo_children():
            w.destroy()
        entries.clear()

        def add(t):
            tk.Label(form, text=t, bg=CARD, fg=TEXT,
                     font=("Arial", 12)).pack(anchor="w")
            e = tk.Entry(form, font=("Arial", 14))
            e.pack(fill="x", pady=8, ipady=4)
            entries[t] = e

        if mode.get() == "1":
            add("Топливо (л)")
            add("Расстояние (км)")
            add("Цена за литр")
        else:
            add("Средний расход (л/100км)")
            add("Расстояние (км)")
            add("Цена за литр")

    def calc():
        try:
            if mode.get() == "1":
                f = float(entries["Топливо (л)"].get())
                d = float(entries["Расстояние (км)"].get())
                p = float(entries["Цена за литр"].get())
                if d == 0:
                    raise ValueError
                cons = (f / d) * 100
                cost = f * p
            else:
                cons = float(entries["Средний расход (л/100км)"].get())
                d = float(entries["Расстояние (км)"].get())
                p = float(entries["Цена за литр"].get())
                f = cons * d / 100
                cost = f * p

            result.config(text=f"{cons:.1f} л/100км | {cost:.2f} €")

            if current_user:
                history = data["users"][current_user]["history"]
                history.append({
                    "date": datetime.now().strftime("%d.%m %H:%M"),
                    "result": result.cget("text")
                })
                data["users"][current_user]["history"] = history[-50:]
                save_data()

        except:
            messagebox.showerror("Ошибка", "Проверь ввод")

    result = tk.Label(content, text="—",
                      bg=BG, fg=ACCENT,
                      font=("Arial", 22, "bold"))
    result.pack()

    tk.Button(content, text="Рассчитать",
              bg=ACCENT, fg="white",
              font=("Arial", 14, "bold"),
              command=calc).pack(pady=10)

    mode.trace("w", lambda *a: build())
    build()

# ===================== HISTORY =====================
def show_history():
    clear()

    tk.Label(content, text="История",
             bg=BG, fg=TEXT,
             font=("Arial", 20, "bold")).pack(pady=10)

    if not current_user:
        return

    for h in data["users"][current_user]["history"][::-1]:
        tk.Label(content,
                 text=f"{h['date']}  |  {h['result']}",
                 bg=CARD, fg=TEXT,
                 font=("Arial", 12)).pack(fill="x", padx=40, pady=5)

# ===================== SETTINGS =====================
def show_settings():
    clear()
    tk.Label(content, text="Настройки",
             bg=BG, fg=TEXT,
             font=("Arial", 20, "bold")).pack(pady=40)

# ===================== ABOUT =====================
def show_about():
    clear()
    tk.Label(content,
             text="CalculatCar Pro 🚗\nКрасивое приложение",
             bg=BG, fg=TEXT,
             font=("Arial", 16)).pack(pady=60)

# ===================== MENU BUTTONS =====================
nav("Калькулятор", show_calc)
nav("Профиль", show_profile)
nav("История", show_history)
nav("Настройки", show_settings)
nav("О программе", show_about)

# ===================== USER =====================
def update_user():
    user_label.config(text=current_user if current_user else "")

# ===================== START =====================
load_data()
show_calc()
root.mainloop()
```
