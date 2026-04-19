import tkinter as tk
from tkinter import messagebox
from datetime import datetime
import json

# ===============================
# 🎨 Тема
# ===============================
COLORS = {
    "bg": "#052e16",
    "card": "#064e3b",
    "accent": "#22c55e",
    "text": "#ecfdf5"
}

# ===============================
# 💾 Данные
# ===============================
users = {}
current_user = None
history = []

# ===============================
# 📂 Файл
# ===============================
def load_data():
    global users
    try:
        with open("data.json", "r", encoding="utf-8") as f:
            users = json.load(f)
    except:
        users = {}

def save_data():
    with open("data.json", "w", encoding="utf-8") as f:
        json.dump(users, f, ensure_ascii=False)

# ===============================
# 🖥️ Окно
# ===============================
root = tk.Tk()
root.title("FuelCalcPro")
root.geometry("1200x800")
root.configure(bg=COLORS["bg"])

# ===============================
# HEADER
# ===============================
header = tk.Frame(root, bg=COLORS["card"], height=80)
header.pack(fill="x")

menu_open = False

def toggle_menu():
    global menu_open
    if menu_open:
        menu_frame.place_forget()
    else:
        menu_frame.place(x=0, y=80, width=250, height=720)
    menu_open = not menu_open

tk.Button(header, text="☰", command=toggle_menu,
          bg=COLORS["card"], fg=COLORS["text"],
          font=("Arial", 18), bd=0).pack(side="left", padx=10)

tk.Label(header, text="⛽ FuelCalcPro",
         bg=COLORS["card"], fg=COLORS["text"],
         font=("Arial", 26, "bold")).pack(pady=15)

# ===============================
# MAIN
# ===============================
main_frame = tk.Frame(root, bg=COLORS["bg"])
main_frame.pack(fill="both", expand=True)

menu_frame = tk.Frame(root, bg=COLORS["card"])

def clear():
    for w in main_frame.winfo_children():
        w.destroy()

# ===============================
# 🧮 КАЛЬКУЛЯТОР
# ===============================
def show_calc():
    clear()

    tk.Label(main_frame, text="Калькулятор топлива",
             bg=COLORS["bg"], fg=COLORS["text"],
             font=("Arial", 22)).pack(pady=20)

    frame = tk.Frame(main_frame, bg=COLORS["card"], padx=30, pady=30)
    frame.pack(pady=20)

    def field(label):
        tk.Label(frame, text=label,
                 bg=COLORS["card"], fg=COLORS["text"],
                 font=("Arial", 14)).pack(anchor="w", pady=5)
        entry = tk.Entry(frame, width=30, font=("Arial", 14))
        entry.pack(pady=5)
        return entry

    distance = field("Расстояние (км)")
    consumption = field("Расход (л/100 км)")
    fuel_used = field("Потраченное топливо (л)")

    def calc():
        try:
            d = float(distance.get())

            if consumption.get():
                c = float(consumption.get())
                fuel = (d / 100) * c
            else:
                fuel = float(fuel_used.get())
                c = (fuel / d) * 100

            if current_user:
                users[current_user]["profile"]["consumption"] = round(c, 2)

            record = {
                "date": datetime.now().strftime("%d.%m.%Y"),
                "consumption": round(c, 2)
            }

            history.append(record)

            if current_user:
                users[current_user]["history"] = history
                save_data()

            messagebox.showinfo("Результат",
                                f"{c:.2f} л/100км\n{fuel:.2f} л")

        except:
            messagebox.showerror("Ошибка", "Проверь ввод")

    tk.Button(main_frame, text="Рассчитать",
              command=calc,
              bg=COLORS["accent"],
              font=("Arial", 16)).pack(pady=20)

# ===============================
# 👤 ПРОФИЛЬ
# ===============================
def show_profile():
    clear()

    if not current_user:
        show_auth()
        return

    profile = users[current_user]["profile"]

    tk.Label(main_frame, text="Профиль",
             bg=COLORS["bg"], fg=COLORS["text"],
             font=("Arial", 22)).pack(pady=20)

    name = tk.Entry(main_frame, font=("Arial", 14))
    vehicle = tk.Entry(main_frame, font=("Arial", 14))

    name.insert(0, profile["name"])
    vehicle.insert(0, profile["vehicle"])

    name.pack(pady=10)
    vehicle.pack(pady=10)

    def save():
        profile["name"] = name.get()
        profile["vehicle"] = vehicle.get()
        save_data()
        messagebox.showinfo("OK", "Сохранено")

    def logout():
        global current_user
        current_user = None
        show_calc()

    tk.Button(main_frame, text="Сохранить", command=save,
              bg=COLORS["accent"]).pack(pady=10)

    tk.Button(main_frame, text="Выйти", command=logout,
              bg="red", fg="white").pack(pady=10)

# ===============================
# 🔐 АВТОРИЗАЦИЯ
# ===============================
def show_auth():
    clear()

    tk.Label(main_frame, text="Вход",
             bg=COLORS["bg"], fg=COLORS["text"],
             font=("Arial", 22)).pack(pady=20)

    login = tk.Entry(main_frame, font=("Arial", 14))
    password = tk.Entry(main_frame, show="*", font=("Arial", 14))

    login.pack(pady=10)
    password.pack(pady=10)

    def login_user():
        global current_user, history
        if login.get() in users and users[login.get()]["password"] == password.get():
            current_user = login.get()
            history = users[current_user]["history"]
            show_profile()
        else:
            messagebox.showerror("Ошибка", "Неверно")

    def register():
        users[login.get()] = {
            "password": password.get(),
            "profile": {"name": "", "vehicle": "", "consumption": ""},
            "history": []
        }
        save_data()
        messagebox.showinfo("OK", "Создан")

    tk.Button(main_frame, text="Войти", command=login_user,
              bg=COLORS["accent"]).pack(pady=10)

    tk.Button(main_frame, text="Регистрация", command=register,
              bg=COLORS["card"]).pack(pady=10)

# ===============================
# 📊 ИСТОРИЯ
# ===============================
def show_history():
    clear()

    tk.Label(main_frame, text="История",
             bg=COLORS["bg"], fg=COLORS["text"],
             font=("Arial", 22)).pack(pady=20)

    for i, h in enumerate(history):
        frame = tk.Frame(main_frame, bg=COLORS["card"])
        frame.pack(fill="x", pady=5, padx=50)

        tk.Label(frame,
                 text=f"{h['date']} | {h['consumption']} л/100км",
                 bg=COLORS["card"], fg=COLORS["text"],
                 font=("Arial", 14)).pack(side="left", padx=10)

        def delete(index=i):
            history.pop(index)
            if current_user:
                users[current_user]["history"] = history
                save_data()
            show_history()

        tk.Button(frame, text="Удалить",
                  command=delete).pack(side="right")

# ===============================
# 📋 МЕНЮ
# ===============================
def menu_button(text, command):
    tk.Button(menu_frame, text=text, command=command,
              bg=COLORS["card"], fg=COLORS["text"],
              font=("Arial", 14), bd=0).pack(fill="x", pady=10)

menu_button("Калькулятор", show_calc)
menu_button("Профиль", show_profile)
menu_button("История", show_history)

# ===============================
# 🚀 СТАРТ
# ===============================
load_data()
show_calc()
root.mainloop()
