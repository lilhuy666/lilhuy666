import tkinter as tk
from tkinter import messagebox
from datetime import datetime
import json
import os

# ===============================
# 🎨 COLORS
# ===============================
BG = "#eef1f5"
SIDEBAR = "#1f4aa8"
HEADER = "#2f5fb3"
CARD = "#ffffff"
TEXT = "#1f2937"
SUB = "#6b7280"
BLUE = "#3b82f6"

DATA_FILE = "data.json"

# ===============================
# 💾 DATA
# ===============================
data = {
    "profile": {
        "name": "Алексей",
        "car": "Toyota Camry"
    },
    "history": []
}

def load_data():
    global data
    if os.path.exists(DATA_FILE):
        with open(DATA_FILE, "r", encoding="utf-8") as f:
            data = json.load(f)

def save_data():
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

# ===============================
# 🪟 WINDOW
# ===============================
root = tk.Tk()
root.title("CalculatCar")
root.geometry("1200x750")
root.configure(bg=BG)

# ===============================
# HEADER
# ===============================
header = tk.Frame(root, bg=HEADER, height=60)
header.pack(fill="x")

menu_visible = True

def toggle_menu():
    global menu_visible
    if menu_visible:
        sidebar.pack_forget()
    else:
        sidebar.pack(side="left", fill="y")
    menu_visible = not menu_visible

tk.Button(header, text="☰",
          bg=HEADER, fg="white",
          font=("Arial", 18),
          bd=0,
          command=toggle_menu).pack(side="left", padx=10)

tk.Label(header, text="CalculatCar",
         bg=HEADER, fg="white",
         font=("Arial", 20, "bold")).pack(pady=10)

# ===============================
# SIDEBAR
# ===============================
sidebar = tk.Frame(root, bg=SIDEBAR, width=220)
sidebar.pack(side="left", fill="y")

main = tk.Frame(root, bg=BG)
main.pack(side="right", fill="both", expand=True)

content = tk.Frame(main, bg=BG)
content.pack(fill="both", expand=True, padx=20, pady=20)

def clear():
    for w in content.winfo_children():
        w.destroy()

# ===============================
# 🧮 КАЛЬКУЛЯТОР
# ===============================
def show_calc():
    clear()

    content.grid_columnconfigure(0, weight=2)
    content.grid_columnconfigure(1, weight=1)

    left = tk.Frame(content, bg=CARD, padx=20, pady=20)
    left.grid(row=0, column=0, sticky="nsew", padx=10, pady=10)

    tk.Label(left, text="Калькулятор",
             bg=CARD, fg=TEXT,
             font=("Arial", 18, "bold")).pack(anchor="w")

    # выбор авто
    tk.Label(left, text="Автомобиль", bg=CARD, fg=SUB).pack(anchor="w")
    car_var = tk.StringVar(value=data["profile"]["car"])
    car_entry = tk.Entry(left, textvariable=car_var)
    car_entry.pack(fill="x", pady=5)

    # выбор типа расчета
    calc_type = tk.StringVar(value="consumption")

    tk.Radiobutton(left, text="Расход (л/100км)",
                   variable=calc_type, value="consumption",
                   bg=CARD).pack(anchor="w")

    tk.Radiobutton(left, text="Стоимость поездки",
                   variable=calc_type, value="cost",
                   bg=CARD).pack(anchor="w")

    def field(text):
        tk.Label(left, text=text, bg=CARD, fg=SUB).pack(anchor="w")
        e = tk.Entry(left)
        e.pack(fill="x", pady=5)
        return e

    distance = field("Дистанция (км)")
    fuel = field("Топливо (л)")
    price = field("Цена (₽/л)")

    right = tk.Frame(content, bg=BG)
    right.grid(row=0, column=1, sticky="nsew")

    result_label = tk.Label(right, text="—",
                            bg=CARD, font=("Arial", 22, "bold"))
    result_label.pack(pady=50)

    def calc():
        try:
            d = float(distance.get())
            f = float(fuel.get())
            p = float(price.get())
            car = car_var.get()

            if calc_type.get() == "consumption":
                result = (f / d) * 100
                text = f"{result:.1f} л/100км"
            else:
                result = f * p
                text = f"{result:.0f} ₽"

            result_label.config(text=text)

            data["history"].append({
                "date": datetime.now().strftime("%d.%m.%Y %H:%M"),
                "car": car,
                "result": text
            })
            save_data()

        except:
            messagebox.showerror("Ошибка", "Проверь ввод")

    tk.Button(left, text="Рассчитать",
              bg=BLUE, fg="white",
              command=calc).pack(fill="x", pady=10)

# ===============================
# 👤 ПРОФИЛЬ
# ===============================
def show_profile():
    clear()

    card = tk.Frame(content, bg=CARD, padx=30, pady=30)
    card.pack(pady=50)

    tk.Label(card, text="Профиль",
             bg=CARD, font=("Arial", 18, "bold")).pack()

    tk.Label(card, text="Имя", bg=CARD).pack(anchor="w")
    name = tk.Entry(card)
    name.insert(0, data["profile"]["name"])
    name.pack(fill="x")

    tk.Label(card, text="Автомобиль", bg=CARD).pack(anchor="w")
    car = tk.Entry(card)
    car.insert(0, data["profile"]["car"])
    car.pack(fill="x")

    def save():
        data["profile"]["name"] = name.get()
        data["profile"]["car"] = car.get()
        save_data()
        messagebox.showinfo("OK", "Сохранено")

    tk.Button(card, text="Сохранить",
              bg=BLUE, fg="white",
              command=save).pack(pady=10)

# ===============================
# 📊 ИСТОРИЯ
# ===============================
def show_history():
    clear()

    tk.Label(content, text="История",
             bg=BG, fg=TEXT,
             font=("Arial", 18, "bold")).pack(anchor="w")

    table = tk.Frame(content, bg=CARD)
    table.pack(fill="both", expand=True, pady=10)

    def delete_item(index):
        data["history"].pop(index)
        save_data()
        show_history()

    for i, h in enumerate(data["history"][::-1]):
        real_index = len(data["history"]) - 1 - i

        row = tk.Frame(table, bg=CARD, pady=10)
        row.pack(fill="x", padx=10)

        tk.Label(row, text=h["date"],
                 bg=CARD, width=20).pack(side="left")

        tk.Label(row, text=h["car"],
                 bg=CARD, width=20).pack(side="left")

        tk.Label(row, text=h["result"],
                 bg=CARD, width=15).pack(side="left")

        tk.Button(row, text="Удалить",
                  command=lambda idx=real_index: delete_item(idx)
                  ).pack(side="right")

# ===============================
# ℹ️ О НАС
# ===============================
def show_about():
    clear()

    card = tk.Frame(content, bg=CARD, padx=30, pady=30)
    card.pack(pady=50)

    tk.Label(card, text="О программе",
             bg=CARD, font=("Arial", 18, "bold")).pack()

    tk.Label(card,
             text="CalculatCar — калькулятор расхода топлива 🚗",
             bg=CARD, fg=SUB).pack(pady=10)

# ===============================
# MENU
# ===============================
def menu_btn(text, cmd):
    tk.Button(sidebar, text=text,
              bg=SIDEBAR, fg="white",
              font=("Arial", 13),
              bd=0, anchor="w",
              padx=20, pady=12,
              command=cmd).pack(fill="x")

menu_btn("Калькулятор", show_calc)
menu_btn("Профиль", show_profile)
menu_btn("История", show_history)
menu_btn("О нас", show_about)

# ===============================
# START
# ===============================
load_data()
show_calc()
root.mainloop()
