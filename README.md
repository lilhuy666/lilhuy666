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

def card():
    f = tk.Frame(content, bg=CARD, padx=40, pady=40)
    f.pack(pady=30)
    return f

# ===================== PROFILE =====================
def show_profile():
    clear()

    if not current_user:
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
                "car": "",
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
    c = card()

    tk.Label(c, text="Профиль",
             bg=CARD, fg=TEXT,
             font=("Arial", 20, "bold")).pack(pady=10)

    # ==== ИМЯ ====
    row1 = tk.Frame(c, bg=CARD)
    row1.pack(fill="x", pady=5)

    tk.Label(row1, text="Имя", bg=CARD, fg=SUB).pack(anchor="w")

    name_entry = tk.Entry(row1, font=("Arial", 14))
    name_entry.insert(0, user.get("name", ""))
    name_entry.pack(side="left", fill="x", expand=True, padx=(0,10))

    def save_name():
        user["name"] = name_entry.get()
        save_data()
        messagebox.showinfo("OK", "Имя сохранено")

    tk.Button(row1, text="Сохранить",
              bg=ACCENT, fg="white",
              command=save_name).pack(side="right")

    # ==== АВТО ====
    row2 = tk.Frame(c, bg=CARD)
    row2.pack(fill="x", pady=5)

    tk.Label(row2, text="Автомобиль", bg=CARD, fg=SUB).pack(anchor="w")

    car_entry = tk.Entry(row2, font=("Arial", 14))
    car_entry.insert(0, user.get("car", ""))
    car_entry.pack(side="left", fill="x", expand=True, padx=(0,10))

    def save_car():
        user["car"] = car_entry.get()
        save_data()
        messagebox.showinfo("OK", "Авто сохранено")

    tk.Button(row2, text="Сохранить",
              bg=ACCENT, fg="white",
              command=save_car).pack(side="right")

    # ==== ПАРОЛЬ ====
    row3 = tk.Frame(c, bg=CARD)
    row3.pack(fill="x", pady=5)

    tk.Label(row3, text="Новый пароль", bg=CARD, fg=SUB).pack(anchor="w")

    pass1 = tk.Entry(row3, show="*", font=("Arial", 14))
    pass1.pack(fill="x", pady=5)

    row4 = tk.Frame(c, bg=CARD)
    row4.pack(fill="x", pady=5)

    tk.Label(row4, text="Повтор пароля", bg=CARD, fg=SUB).pack(anchor="w")

    pass2 = tk.Entry(row4, show="*", font=("Arial", 14))
    pass2.pack(side="left", fill="x", expand=True, padx=(0,10))

    def save_password():
        p1 = pass1.get()
        p2 = pass2.get()

        if not p1 or not p2:
            return messagebox.showerror("Ошибка", "Введите пароль")

        if p1 != p2:
            return messagebox.showerror("Ошибка", "Пароли не совпадают")

        user["password"] = p1
        save_data()
        messagebox.showinfo("OK", "Пароль изменён")

    tk.Button(row4, text="Сохранить",
              bg=ACCENT, fg="white",
              command=save_password).pack(side="right")

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

    tk.Label(content, text="Калькулятор расхода",
             bg=BG, fg=TEXT,
             font=("Arial", 20, "bold")).pack(pady=10)

    mode = tk.StringVar(value="1")

    switch = tk.Frame(content, bg=BG)
    switch.pack(pady=10)

    tk.Radiobutton(switch, text="Рассчитать стоимость поездки",
                   variable=mode, value="1",
                   indicatoron=0, width=28,
                   bg=PANEL, fg=TEXT,
                   selectcolor=ACCENT).pack(side="left", padx=5)

    tk.Radiobutton(switch, text="Средний расход на 100 км",
                   variable=mode, value="2",
                   indicatoron=0, width=28,
                   bg=PANEL, fg=TEXT,
                   selectcolor=ACCENT).pack(side="left", padx=5)

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
            e.pack(fill="x", pady=5)
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
                cons = (f / d) * 100
                cost = f * p
            else:
                cons = float(entries["Средний расход (л/100км)"].get())
                d = float(entries["Расстояние (км)"].get())
                p = float(entries["Цена за литр"].get())
                cost = cons * d / 100 * p

            result.config(text=f"{cons:.1f} л/100км | {cost:.2f} €")

            if current_user:
                h = data["users"][current_user]["history"]
                h.append({
                    "date": datetime.now().strftime("%d.%m %H:%M"),
                    "result": result.cget("text")
                })
                data["users"][current_user]["history"] = h[-50:]
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
