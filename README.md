import tkinter as tk
from tkinter import messagebox
from datetime import datetime
import json

# ===============================
# 🎨 ТЕМА
# ===============================
COLORS = {
    "bg": "#022c22",
    "card": "#064e3b",
    "accent": "#22c55e",
    "text": "#ecfdf5"
}

# ===============================
# 💾 ДАННЫЕ
# ===============================
users = {}
current_user = None
history = []

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
# 🖥️ ОКНО
# ===============================
root = tk.Tk()
root.title("FuelCalcPro")
root.geometry("1100x750")
root.configure(bg=COLORS["bg"])

# ===============================
# HEADER
# ===============================
header = tk.Frame(root, bg=COLORS["card"], height=70)
header.pack(fill="x")

menu_open = False

def toggle_menu():
    global menu_open
    if menu_open:
        menu_frame.place_forget()
    else:
        menu_frame.place(x=0, y=70, width=250, height=680)
    menu_open = not menu_open

tk.Button(header, text="☰", command=toggle_menu,
          bg=COLORS["card"], fg=COLORS["text"],
          font=("Arial", 18), bd=0).pack(side="left", padx=10)

tk.Label(header, text="⛽ FuelCalcPro",
         bg=COLORS["card"], fg=COLORS["text"],
         font=("Arial", 22, "bold")).pack(pady=10)

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

    tk.Label(main_frame, text="Калькулятор расхода топлива",
             bg=COLORS["bg"], fg=COLORS["text"],
             font=("Arial", 22)).pack(pady=20)

    card = tk.Frame(main_frame, bg=COLORS["card"], padx=30, pady=30)
    card.pack(pady=10)

    mode = tk.StringVar(value="avg")
    currency = tk.StringVar(value="₽")
    unit = tk.StringVar(value="л/100км")

    fields = {}

    def create_field(parent, label, unit_text=""):
        frame = tk.Frame(parent, bg=COLORS["card"])
        frame.pack(pady=8, anchor="w")

        tk.Label(frame, text=label,
                 bg=COLORS["card"], fg=COLORS["text"]).pack(anchor="w")

        row = tk.Frame(frame, bg=COLORS["card"])
        row.pack()

        entry = tk.Entry(row, width=25)
        entry.pack(side="left")

        tk.Label(row, text=unit_text,
                 bg=COLORS["card"], fg="#a7f3d0").pack(side="left", padx=5)

        return entry

    def build():
        for w in fields_frame.winfo_children():
            w.destroy()

        fields.clear()

        if mode.get() == "avg":
            fields["fuel"] = create_field(fields_frame, "Израсходовано топлива", "л")
            fields["distance"] = create_field(fields_frame, "Пройденное расстояние", "км")
            fields["price"] = create_field(fields_frame, "Цена за литр")

        else:
            fields["consumption"] = create_field(fields_frame, "Средний расход топлива")
            tk.OptionMenu(fields_frame, unit, "л/100км", "км/л").pack(pady=5)

            fields["distance"] = create_field(fields_frame, "Расстояние", "км")
            fields["price"] = create_field(fields_frame, "Цена за литр")

        tk.OptionMenu(fields_frame, currency, "₽", "$", "€").pack(pady=10)

    tk.Radiobutton(card, text="Рассчитать расход и стоимость",
                   variable=mode, value="cost",
                   command=build,
                   bg=COLORS["card"], fg=COLORS["text"]).pack(anchor="w")

    tk.Radiobutton(card, text="Рассчитать средний расход на 100 км",
                   variable=mode, value="avg",
                   command=build,
                   bg=COLORS["card"], fg=COLORS["text"]).pack(anchor="w")

    fields_frame = tk.Frame(card, bg=COLORS["card"])
    fields_frame.pack(pady=10)

    build()

    def calc():
        try:
            if mode.get() == "avg":
                f = float(fields["fuel"].get())
                d = float(fields["distance"].get())
                p = float(fields["price"].get())

                consumption = (f / d) * 100
                cost = f * p

            else:
                d = float(fields["distance"].get())
                p = float(fields["price"].get())
                c = float(fields["consumption"].get())

                if unit.get() == "л/100км":
                    f = (d / 100) * c
                    consumption = c
                else:
                    f = d / c
                    consumption = 100 / c

                cost = f * p

            # 👉 ВАЖНО: datetime используется полноценно
            now = datetime.now()
            record = {
                "datetime": now.strftime("%d.%m.%Y %H:%M"),
                "consumption": round(consumption, 2)
            }

            history.append(record)

            if current_user:
                users[current_user]["history"] = history
                save_data()

            messagebox.showinfo("Результат",
                                f"{consumption:.2f} л/100км\n"
                                f"{cost:.2f} {currency.get()}")

        except:
            messagebox.showerror("Ошибка", "Проверь ввод")

    tk.Button(main_frame, text="РАССЧИТАТЬ",
              command=calc,
              bg=COLORS["accent"], fg="black",
              font=("Arial", 14), padx=20, pady=10).pack(pady=20)

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
                 text=f"{h['datetime']} | {h['consumption']} л/100км",
                 bg=COLORS["bg"], fg=COLORS["text"]).pack()

# ===============================
# 👤 ПРОФИЛЬ
# ===============================
def show_profile():
    clear()

    if not current_user:
        show_auth()
        return

    tk.Label(main_frame, text="Профиль",
             bg=COLORS["bg"], fg=COLORS["text"],
             font=("Arial", 22)).pack(pady=20)

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
             font=("Arial", 24)).pack(pady=20)

    login = tk.Entry(main_frame)
    password = tk.Entry(main_frame, show="*")

    login.pack(pady=10)
    password.pack(pady=10)

    def login_user():
        global current_user, history
        if login.get() in users and users[login.get()]["password"] == password.get():
            current_user = login.get()
            history = users[current_user].get("history", [])
            show_calc()
        else:
            messagebox.showerror("Ошибка", "Неверно")

    def register():
        users[login.get()] = {"password": password.get(), "history": []}
        save_data()
        messagebox.showinfo("OK", "Создан")

    tk.Button(main_frame, text="Войти", command=login_user).pack()
    tk.Button(main_frame, text="Регистрация", command=register).pack()

# ===============================
# 📋 МЕНЮ
# ===============================
def menu_button(text, command):
    tk.Button(menu_frame, text=text, command=command,
              bg=COLORS["card"], fg=COLORS["text"],
              font=("Arial", 14), bd=0).pack(fill="x", pady=10)

menu_button("Калькулятор", show_calc)
menu_button("История", show_history)
menu_button("Профиль", show_profile)

# ===============================
# 🚀 СТАРТ
# ===============================
load_data()
show_calc()
root.mainloop()
