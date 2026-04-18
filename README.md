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
        json.dump(users, f, ensure_ascii=False, indent=2)

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

    tk.Label(main_frame, text="Калькулятор",
             bg=COLORS["bg"], fg=COLORS["text"],
             font=("Arial", 22)).pack(pady=10)

    if not current_user:
        tk.Label(main_frame, text="Сначала войдите в профиль",
                 bg=COLORS["bg"], fg="red").pack()
        return

    user = users[current_user]

    # выбор авто
    tk.Label(main_frame, text="Выберите авто",
             bg=COLORS["bg"], fg=COLORS["text"]).pack()

    car_var = tk.StringVar()
    car_var.set(user["cars"][0]["name"] if user["cars"] else "")

    cars = [c["name"] for c in user["cars"]]
    tk.OptionMenu(main_frame, car_var, *cars).pack(pady=5)

    # выбор режима
    mode = tk.StringVar(value="1")

    tk.Radiobutton(main_frame, text="Средний расход",
                   variable=mode, value="1",
                   bg=COLORS["bg"], fg=COLORS["text"]).pack()

    tk.Radiobutton(main_frame, text="Расход и стоимость",
                   variable=mode, value="2",
                   bg=COLORS["bg"], fg=COLORS["text"]).pack()

    frame = tk.Frame(main_frame, bg=COLORS["card"], padx=20, pady=20)
    frame.pack(pady=20)

    def field(label):
        tk.Label(frame, text=label,
                 bg=COLORS["card"], fg=COLORS["text"]).pack()
        e = tk.Entry(frame)
        e.pack(pady=5)
        return e

    # поля
    fuel = field("Топливо (л)")
    distance = field("Расстояние (км)")
    price = field("Цена за литр")

    unit = tk.StringVar(value="л/100км")
    tk.OptionMenu(frame, unit, "л/100км", "км/л").pack(pady=5)

    currency = tk.StringVar(value="€")
    tk.OptionMenu(frame, currency, "€", "$", "₽").pack(pady=5)

    def calc():
        try:
            d = float(distance.get())
            p = float(price.get())
            f = float(fuel.get())

            car = next(c for c in user["cars"] if c["name"] == car_var.get())

            if mode.get() == "1":
                consumption = (f / d) * 100
                cost = f * p
            else:
                if unit.get() == "л/100км":
                    consumption = float(car.get("consumption", 0))
                    f = (d / 100) * consumption
                else:
                    km_per_l = float(car.get("consumption", 1))
                    f = d / km_per_l
                    consumption = 100 / km_per_l

                cost = f * p

            car["consumption"] = round(consumption, 2)

            record = {
                "date": datetime.now().strftime("%d.%m.%Y"),
                "car": car["name"],
                "consumption": round(consumption, 2)
            }

            history.append(record)
            user["history"] = history
            save_data()

            messagebox.showinfo("Результат",
                                f"{consumption:.2f} л/100км\n"
                                f"{f:.2f} л\n"
                                f"{cost:.2f} {currency.get()}")

        except Exception as e:
            messagebox.showerror("Ошибка", "Проверь ввод")

    tk.Button(main_frame, text="Рассчитать",
              command=calc,
              bg=COLORS["accent"]).pack(pady=10)

# ===============================
# 👤 ПРОФИЛЬ
# ===============================
def show_profile():
    clear()

    if not current_user:
        show_auth()
        return

    user = users[current_user]

    tk.Label(main_frame, text="Профиль",
             bg=COLORS["bg"], fg=COLORS["text"],
             font=("Arial", 22)).pack(pady=10)

    name_entry = tk.Entry(main_frame, font=("Arial", 16), width=30)
    name_entry.insert(0, user["name"])
    name_entry.pack(pady=10)

    tk.Label(main_frame, text="Автомобили",
             bg=COLORS["bg"], fg=COLORS["text"]).pack()

    car_entry = tk.Entry(main_frame, font=("Arial", 14))
    car_entry.pack(pady=5)

    def add_car():
        name = car_entry.get()
        if name:
            user["cars"].append({"name": name, "consumption": 0})
            save_data()
            show_profile()

    tk.Button(main_frame, text="Добавить авто",
              command=add_car,
              bg=COLORS["accent"]).pack(pady=5)

    for car in user["cars"]:
        tk.Label(main_frame,
                 text=f"{car['name']} | {car.get('consumption', 0)} л/100км",
                 bg=COLORS["bg"], fg=COLORS["text"]).pack()

    def auto_save(*args):
        user["name"] = name_entry.get()
        save_data()

    name_entry.bind("<KeyRelease>", auto_save)

    tk.Button(main_frame, text="Выйти",
              command=logout,
              bg="red", fg="white").pack(pady=10)

def logout():
    global current_user
    current_user = None
    show_calc()

# ===============================
# 🔐 АВТОРИЗАЦИЯ
# ===============================
def show_auth():
    clear()

    tk.Label(main_frame, text="Вход",
             bg=COLORS["bg"], fg=COLORS["text"],
             font=("Arial", 26)).pack(pady=20)

    login = tk.Entry(main_frame, font=("Arial", 18), width=30)
    password = tk.Entry(main_frame, show="*", font=("Arial", 18), width=30)

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
            "name": "",
            "cars": [],
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

    for h in history:
        tk.Label(main_frame,
                 text=f"{h['date']} | {h['car']} | {h['consumption']} л/100км",
                 bg=COLORS["bg"], fg=COLORS["text"]).pack()

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
