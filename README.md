import tkinter as tk
from tkinter import messagebox
from datetime import datetime
import json
import requests

# ===============================
# 🎨 Тема
# ===============================
COLORS = {
    "primary": "#16a34a",
    "secondary": "#166534",
    "accent": "#22c55e",
    "background": "#052e16",
    "surface": "#064e3b",
    "text": "#ecfdf5",
}

# ===============================
# 💾 Данные
# ===============================
users = {}
current_user = None
history = []

# ===============================
# 💾 Работа с файлом
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
root.configure(bg=COLORS["background"])

# ===============================
# Layout
# ===============================
menu_visible = True

container = tk.Frame(root, bg=COLORS["background"])
container.pack(fill="both", expand=True)

menu_frame = tk.Frame(container, bg=COLORS["surface"], width=250)
menu_frame.pack(side="left", fill="y")

main_frame = tk.Frame(container, bg=COLORS["background"])
main_frame.pack(side="left", expand=True, fill="both")

# ===============================
def toggle_menu():
    global menu_visible
    if menu_visible:
        menu_frame.pack_forget()
    else:
        menu_frame.pack(side="left", fill="y")
    menu_visible = not menu_visible

def clear_main():
    for w in main_frame.winfo_children():
        w.destroy()

# ===============================
# 🔐 Авторизация
# ===============================
def show_auth():
    clear_main()

    tk.Label(main_frame, text="Вход / Регистрация",
             bg=COLORS["background"], fg=COLORS["text"],
             font=("Arial", 24)).pack(pady=30)

    login = tk.Entry(main_frame, width=30)
    password = tk.Entry(main_frame, width=30, show="*")

    login.pack(pady=10)
    password.pack(pady=10)

    def register():
        if login.get() in users:
            messagebox.showerror("Ошибка", "Пользователь уже есть")
        else:
            users[login.get()] = {
                "password": password.get(),
                "profile": {"name": "", "vehicle": "", "consumption": ""},
                "history": []
            }
            save_data()
            messagebox.showinfo("Успех", "Зарегистрирован!")

    def login_user():
        global current_user, history
        if login.get() in users and users[login.get()]["password"] == password.get():
            current_user = login.get()
            history = users[current_user]["history"]
            show_calc()
        else:
            messagebox.showerror("Ошибка", "Неверные данные")

    tk.Button(main_frame, text="Войти", command=login_user,
              bg=COLORS["accent"]).pack(pady=10)

    tk.Button(main_frame, text="Регистрация", command=register,
              bg=COLORS["secondary"]).pack(pady=10)

# ===============================
# 👤 Профиль
# ===============================
def show_profile():
    clear_main()

    if not current_user:
        show_auth()
        return

    profile = users[current_user]["profile"]

    tk.Label(main_frame, text="Профиль",
             bg=COLORS["background"], fg=COLORS["text"],
             font=("Arial", 24)).pack(pady=20)

    name = tk.Entry(main_frame, width=40)
    vehicle = tk.Entry(main_frame, width=40)
    consumption = tk.Entry(main_frame, width=40)

    name.insert(0, profile["name"])
    vehicle.insert(0, profile["vehicle"])
    consumption.insert(0, profile["consumption"])

    name.pack(pady=10)
    vehicle.pack(pady=10)
    consumption.pack(pady=10)

    def save():
        profile["name"] = name.get()
        profile["vehicle"] = vehicle.get()
        profile["consumption"] = consumption.get()
        save_data()
        messagebox.showinfo("Сохранено", "Профиль обновлён")

    tk.Button(main_frame, text="Сохранить",
              command=save, bg=COLORS["accent"]).pack(pady=20)

# ===============================
# 🧮 Калькулятор
# ===============================
def show_calc():
    clear_main()

    if not current_user:
        show_auth()
        return

    profile = users[current_user]["profile"]

    tk.Label(main_frame, text="Калькулятор",
             bg=COLORS["background"], fg=COLORS["text"],
             font=("Arial", 24)).pack(pady=20)

    distance = tk.Entry(main_frame, width=40)
    consumption = tk.Entry(main_frame, width=40)
    fuel_used = tk.Entry(main_frame, width=40)

    distance.pack(pady=10)
    consumption.pack(pady=10)
    fuel_used.pack(pady=10)

    consumption.insert(0, profile["consumption"])

    def calculate():
        try:
            d = float(distance.get())

            if consumption.get():
                c = float(consumption.get())
                fuel = (d / 100) * c
            else:
                fuel = float(fuel_used.get())
                c = (fuel / d) * 100

            profile["consumption"] = round(c, 2)

            record = {
                "date": datetime.now().strftime("%d.%m.%Y"),
                "distance": d,
                "fuel": round(fuel, 2),
                "consumption": round(c, 2)
            }

            history.append(record)
            users[current_user]["history"] = history
            save_data()

            messagebox.showinfo("Результат",
                                f"Расход: {c:.2f} л/100км\nТопливо: {fuel:.2f} л")

        except ValueError:
            messagebox.showerror("Ошибка", "Проверь ввод")

    tk.Button(main_frame, text="Рассчитать",
              command=calculate,
              bg=COLORS["accent"]).pack(pady=20)

# ===============================
# 📊 История
# ===============================
def show_history():
    clear_main()

    tk.Label(main_frame, text="История",
             bg=COLORS["background"], fg=COLORS["text"],
             font=("Arial", 24)).pack(pady=20)

    for i, h in enumerate(history):
        frame = tk.Frame(main_frame, bg=COLORS["surface"])
        frame.pack(fill="x", pady=5)

        tk.Label(frame,
                 text=f"{h['date']} | {h['consumption']} л/100км",
                 bg=COLORS["surface"], fg=COLORS["text"]).pack(side="left")

        def delete(index=i):
            history.pop(index)
            users[current_user]["history"] = history
            save_data()
            show_history()

        tk.Button(frame, text="Удалить",
                  command=delete).pack(side="right")

    if history:
        avg = sum(h["consumption"] for h in history) / len(history)
        tk.Label(main_frame,
                 text=f"Средний расход: {avg:.2f}",
                 bg=COLORS["background"], fg=COLORS["accent"]).pack(pady=10)

# ===============================
# ⛽ API ЦЕНЫ
# ===============================
def get_fuel_prices():
    try:
        # пример API (можно заменить)
        response = requests.get("https://api.mocki.io/v2/549a5d8b/FuelPrices")
        data = response.json()

        return {
            "Газпромнефть": data.get("gazprom", "нет данных"),
            "Роснефть": data.get("rosneft", "нет данных"),
            "Лукойл": data.get("lukoil", "нет данных"),
        }

    except:
        # fallback
        return {
            "Газпромнефть": "52.3 ₽",
            "Роснефть": "51.8 ₽",
            "Лукойл": "53.1 ₽"
        }

def show_fuel():
    clear_main()

    tk.Label(main_frame, text="Цены на топливо",
             bg=COLORS["background"], fg=COLORS["text"],
             font=("Arial", 24)).pack(pady=20)

    prices = get_fuel_prices()

    for k, v in prices.items():
        tk.Label(main_frame,
                 text=f"{k}: {v}",
                 bg=COLORS["background"], fg=COLORS["text"]).pack(pady=5)

# ===============================
# ℹ️ О нас
# ===============================
def show_about():
    clear_main()

    tk.Label(main_frame, text="О нас",
             bg=COLORS["background"], fg=COLORS["text"],
             font=("Arial", 24)).pack(pady=20)

    tk.Label(main_frame,
             text="FuelCalcPro — приложение для расчета топлива, анализа расходов и контроля затрат.",
             wraplength=600,
             bg=COLORS["background"], fg=COLORS["text"]).pack(pady=10)

# ===============================
# 📋 Меню
# ===============================
tk.Button(root, text="☰", command=toggle_menu,
          bg=COLORS["surface"], fg=COLORS["text"]).place(x=10, y=10)

tk.Button(menu_frame, text="Калькулятор", command=show_calc).pack(fill="x")
tk.Button(menu_frame, text="Профиль", command=show_profile).pack(fill="x")
tk.Button(menu_frame, text="История", command=show_history).pack(fill="x")
tk.Button(menu_frame, text="Заправки", command=show_fuel).pack(fill="x")
tk.Button(menu_frame, text="О нас", command=show_about).pack(fill="x")

# ===============================
# 🚀 Старт
# ===============================
load_data()
show_auth()
root.mainloop()
