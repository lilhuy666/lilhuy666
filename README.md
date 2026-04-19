import tkinter as tk
from tkinter import messagebox
from datetime import datetime
import json, os
from PIL import Image, ImageTk  # <-- ДОБАВЛЕНО

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

# ===================== MAIN =====================
main = tk.Frame(root)
main.pack(fill="both", expand=True)

# ===== BACKGROUND IMAGE =====
bg_image = Image.open("background.jpg")
bg_image = bg_image.resize((1200, 750))
bg_photo = ImageTk.PhotoImage(bg_image)

bg_label = tk.Label(main, image=bg_photo)
bg_label.place(x=0, y=0, relwidth=1, relheight=1)

# ===================== LAYOUT =====================
main.grid_rowconfigure(0, weight=1)
main.grid_columnconfigure(1, weight=1)

# ===================== SIDEBAR =====================
sidebar = tk.Frame(main, bg=PANEL, width=260)
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

# ===================== CONTENT AREA =====================
content_frame = tk.Frame(main, bg=BG)
content_frame.grid(row=0, column=1, sticky="nsew")

# ===================== HEADER =====================
header = tk.Frame(content_frame, bg=BG)
header.pack(fill="x")

tk.Button(header, text="☰",
          bg=BG, fg=TEXT,
          font=("Arial", 18),
          bd=0,
          command=toggle_menu).pack(side="left", padx=15, pady=10)

title = tk.Label(header, text="CalculatCar Pro",
                 bg=BG, fg=TEXT,
                 font=("Arial", 18, "bold"))
title.pack(side="left", pady=10)

user_label = tk.Label(header, text="",
                      bg=BG, fg=SUB,
                      font=("Arial", 12))
user_label.pack(side="right", padx=15)

# ===================== CONTENT =====================
content = tk.Frame(content_frame, bg=BG)
content.pack(fill="both", expand=True)

# поднимаем поверх фона
content_frame.lift()
sidebar.lift()

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

    tk.Button(c, text="Войти", bg=ACCENT, fg="white",
              font=("Arial", 12, "bold"),
              command=login).pack(fill="x", pady=10)

    tk.Button(c, text="Регистрация", bg=ACCENT2, fg="black",
              font=("Arial", 12, "bold"),
              command=register).pack(fill="x")

# ===================== CALC =====================
def show_calc():
    clear()

    tk.Label(content, text="Калькулятор расхода",
             bg=BG, fg=TEXT,
             font=("Arial", 20, "bold")).pack(pady=10)

    mode = tk.StringVar(value="1")

    form = tk.Frame(content, bg=CARD, padx=40, pady=40)
    form.pack(pady=20)

    entries = {}

    def build():
        for w in form.winfo_children():
            w.destroy()
        entries.clear()

        def add(t):
            tk.Label(form, text=t, bg=CARD, fg=TEXT).pack(anchor="w")
            e = tk.Entry(form)
            e.pack(fill="x", pady=8)
            entries[t] = e

        add("Топливо (л)")
        add("Расстояние (км)")
        add("Цена за литр")

    def calc():
        try:
            f = float(entries["Топливо (л)"].get())
            d = float(entries["Расстояние (км)"].get())
            p = float(entries["Цена за литр"].get())

            cons = (f / d) * 100
            cost = f * p

            result.config(text=f"{cons:.1f} л/100км | {cost:.0f} ₽")
        except:
            messagebox.showerror("Ошибка", "Проверь ввод")

    result = tk.Label(content, text="—",
                      bg=BG, fg=ACCENT,
                      font=("Arial", 22, "bold"))
    result.pack()

    tk.Button(content, text="Рассчитать",
              bg=ACCENT, fg="white",
              command=calc).pack(pady=10)

    build()

# ===================== MENU =====================
def nav(text, cmd):
    tk.Button(sidebar, text=text,
              bg=PANEL, fg=TEXT,
              bd=0, anchor="w",
              padx=20, pady=14,
              command=cmd).pack(fill="x")

nav("Калькулятор", show_calc)
nav("Профиль", show_auth)

# ===================== USER =====================
def update_user():
    user_label.config(text=current_user if current_user else "")

# ===================== START =====================
load_data()
show_calc()
root.mainloop()
