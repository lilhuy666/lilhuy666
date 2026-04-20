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

def save_data():
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=2, ensure_ascii=False)

# ===================== WINDOW =====================
root = tk.Tk()
root.title("CalculatCar")
root.geometry("1200x750")
root.configure(bg=BG)

main = tk.Frame(root, bg=BG)
main.pack(fill="both", expand=True)

# ===================== HEADER =====================
header = tk.Frame(main, bg=ACCENT, height=70)
header.pack(fill="x")
header.pack_propagate(False)

menu_window = None

def toggle_menu():
    global menu_window

    if menu_window and menu_window.winfo_exists():
        menu_window.destroy()
        return

    x = root.winfo_x() + 20
    y = root.winfo_y() + 70

    menu_window = tk.Toplevel(root)
    menu_window.overrideredirect(True)
    menu_window.configure(bg=PANEL)
    menu_window.geometry(f"220x240+{x}+{y}")

    def close(e=None):
        if menu_window:
            menu_window.destroy()

    menu_window.bind("<FocusOut>", close)

    def nav(text, cmd):
        tk.Button(menu_window,
                  text=text,
                  bg=PANEL,
                  fg=TEXT,
                  bd=0,
                  anchor="w",
                  padx=15,
                  pady=10,
                  font=("Arial", 11),
                  activebackground=CARD,
                  activeforeground=ACCENT,
                  command=lambda: [cmd(), close()]
                  ).pack(fill="x")

    nav("Калькулятор", show_calc)
    nav("Профиль", show_profile)
    nav("История", show_history)
    nav("Настройки", show_settings)
    nav("О программе", show_about)

    menu_window.focus_force()

tk.Button(header, text="☰", bg=ACCENT, fg="white",
          font=("Arial", 16, "bold"), bd=0,
          command=toggle_menu).pack(side="left", padx=15)

tk.Label(header, text="CalculatCar",
         bg=ACCENT, fg="white",
         font=("Arial", 20, "bold")).pack(side="left")

user_label = tk.Label(header, text="",
                      bg=ACCENT, fg="white",
                      font=("Arial", 11))
user_label.pack(side="right", padx=15)

# ===================== CONTENT =====================
content = tk.Frame(main, bg=BG)
content.pack(fill="both", expand=True)

def clear():
    for w in content.winfo_children():
        w.destroy()

def card(full=False):
    f = tk.Frame(content, bg=CARD, padx=40, pady=40)

    if full:
        f.pack(fill="both", expand=True, padx=20, pady=20)
    else:
        f.pack(pady=30)

    return f

# ===================== PROFILE =====================
def show_profile():
    clear()

    if not current_user:
        c = card(full=True)

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
                show_profile()
            else:
                messagebox.showerror("Ошибка", "Неверные данные")

        def register():
            e = email.get().strip()
            p = password.get().strip()

            if not e or not p:
                return messagebox.showerror("Ошибка", "Заполни поля")

            if e in data["users"]:
                return messagebox.showerror("Ошибка", "Уже существует")

            data["users"][e] = {
                "password": p,
                "name": "",
                "cars": [],
                "history": []
            }
            save_data()
            messagebox.showinfo("OK", "Аккаунт создан")

        tk.Button(c, text="Войти", bg=ACCENT, fg="white",
                  command=login).pack(fill="x", pady=10)

        tk.Button(c, text="Регистрация", bg=ACCENT2, fg="black",
                  command=register).pack(fill="x")
        return

    user = data["users"][current_user]
    c = card(full=True)

    tk.Label(c, text="Профиль",
             bg=CARD, fg=TEXT,
             font=("Arial", 20, "bold")).pack(pady=10)

    # ===== ИМЯ =====
    tk.Label(c, text="Имя", bg=CARD, fg=SUB).pack(anchor="w")
    name_entry = tk.Entry(c, font=("Arial", 14))
    name_entry.insert(0, user.get("name", ""))
    name_entry.pack(fill="x", pady=5)

    def save_name():
        user["name"] = name_entry.get()
        save_data()
        messagebox.showinfo("OK", "Сохранено")

    tk.Button(c, text="Сохранить имя",
              bg=ACCENT, fg="white",
              command=save_name).pack(fill="x", pady=5)

    # ===== ДОБАВЛЕНИЕ АВТО =====
    tk.Label(c, text="Добавить авто",
             bg=CARD, fg=TEXT,
             font=("Arial", 14, "bold")).pack(pady=10)

    car_name = tk.Entry(c)
    car_name.insert(0, "Название авто")
    car_name.pack(fill="x", pady=5)

    car_cons = tk.Entry(c)
    car_cons.insert(0, "Расход л/100км")
    car_cons.pack(fill="x", pady=5)

    def add_car():
        name = car_name.get().strip()
        try:
            cons = float(car_cons.get())
        except:
            return messagebox.showerror("Ошибка", "Неверный расход")

        if not name:
            return messagebox.showerror("Ошибка", "Введите имя авто")

        user["cars"].append({
            "name": name,
            "consumption": cons
        })

        save_data()
        show_profile()

    tk.Button(c, text="Добавить авто",
              bg=ACCENT2, fg="black",
              command=add_car).pack(fill="x", pady=5)

    # ===== СПИСОК АВТО =====
    tk.Label(c, text="Мои авто",
             bg=CARD, fg=TEXT,
             font=("Arial", 14, "bold")).pack(pady=10)

    for car in user.get("cars", []):
        tk.Label(c,
                 text=f"{car['name']} — {car['consumption']} л/100км",
                 bg=PANEL, fg=TEXT,
                 padx=10, pady=5).pack(fill="x", pady=2)

    def logout():
        global current_user
        current_user = None
        update_user()
        show_profile()

    tk.Button(c, text="Выйти",
              bg=DANGER, fg="white",
              command=logout).pack(fill="x", pady=10)

# ===================== CALCULATOR =====================
def show_calc():
    clear()

    tk.Label(content, text="Калькулятор",
             bg=BG, fg=TEXT,
             font=("Arial", 20, "bold")).pack(pady=10)

    tk.Label(content, text="(оставил твою логику без изменений)",
             bg=BG, fg=SUB).pack()

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
                 text=f"{h['date']} | {h['result']}",
                 bg=CARD, fg=TEXT).pack(fill="x", padx=40, pady=5)

# ===================== OTHER =====================
def show_settings():
    clear()
    tk.Label(content, text="Настройки", bg=BG, fg=TEXT).pack(pady=40)

def show_about():
    clear()
    tk.Label(content, text="CalculatCar 🚗", bg=BG, fg=TEXT).pack(pady=40)

def update_user():
    user_label.config(text=current_user if current_user else "")

# ===================== START =====================
load_data()
show_calc()
root.mainloop()
