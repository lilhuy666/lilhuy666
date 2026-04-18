import tkinter as tk
from tkinter import ttk

# =========================
# 🎨 Цвета
# =========================
COLORS = {
    "bg": "#f1f5f9",
    "sidebar": "#1e3a8a",
    "card": "#ffffff",
    "accent": "#3b82f6",
    "text": "#0f172a",
    "muted": "#64748b"
}

# =========================
# 🧮 Расчет
# =========================
def calculate():
    try:
        distance = float(entry_distance.get())
        fuel = float(entry_fuel.get())
        price = float(entry_price.get())

        consumption = (fuel / distance) * 100
        cost = fuel * price

        lbl_consumption.config(text=f"{consumption:.1f} л/100 км")
        lbl_cost.config(text=f"{cost:.0f} ₽")

    except:
        lbl_consumption.config(text="Ошибка")
        lbl_cost.config(text="Ошибка")

# =========================
# 🖥️ Окно
# =========================
root = tk.Tk()
root.title("CalculatCar")
root.geometry("1100x700")
root.configure(bg=COLORS["bg"])

# =========================
# 🔷 Header
# =========================
header = tk.Frame(root, bg=COLORS["accent"], height=60)
header.pack(fill="x")

tk.Label(header, text="CalculatCar", bg=COLORS["accent"],
         fg="white", font=("Arial", 20, "bold")).pack(pady=10)

# =========================
# 📦 Основной контейнер
# =========================
container = tk.Frame(root, bg=COLORS["bg"])
container.pack(fill="both", expand=True)

# =========================
# 📚 Sidebar
# =========================
sidebar = tk.Frame(container, bg=COLORS["sidebar"], width=200)
sidebar.pack(side="left", fill="y")

def menu_btn(text):
    return tk.Button(sidebar, text=text, bg=COLORS["sidebar"],
                     fg="white", relief="flat", height=2)

menu_btn("Профиль").pack(fill="x")
menu_btn("Калькулятор").pack(fill="x")
menu_btn("История").pack(fill="x")
menu_btn("О нас").pack(fill="x")

# =========================
# 🧩 Main area
# =========================
main = tk.Frame(container, bg=COLORS["bg"])
main.pack(side="left", fill="both", expand=True, padx=20, pady=20)

tk.Label(main, text="Калькулятор расхода топлива",
         bg=COLORS["bg"], fg=COLORS["text"],
         font=("Arial", 18, "bold")).pack(anchor="w", pady=10)

content = tk.Frame(main, bg=COLORS["bg"])
content.pack(fill="both", expand=True)

# =========================
# 🧾 Левая карточка (ввод)
# =========================
card_left = tk.Frame(content, bg=COLORS["card"], padx=20, pady=20)
card_left.pack(side="left", fill="y", padx=10)

def input_field(parent, label):
    tk.Label(parent, text=label, bg=COLORS["card"],
             fg=COLORS["text"]).pack(anchor="w")
    entry = tk.Entry(parent, width=25)
    entry.pack(pady=5)
    return entry

entry_distance = input_field(card_left, "Дистанция (км):")
entry_fuel = input_field(card_left, "Кол-во топлива (л):")
entry_price = input_field(card_left, "Цена топлива (₽/л):")

tk.Button(card_left, text="Рассчитать", bg=COLORS["accent"],
          fg="white", command=calculate).pack(pady=15, fill="x")

# =========================
# 📊 Правая карточка (результат)
# =========================
card_right = tk.Frame(content, bg=COLORS["card"], padx=20, pady=20)
card_right.pack(side="left", fill="y", padx=10)

tk.Label(card_right, text="Результаты",
         bg=COLORS["card"], font=("Arial", 14, "bold")).pack(anchor="w")

tk.Label(card_right, text="Расход топлива",
         bg=COLORS["card"], fg=COLORS["muted"]).pack(anchor="w", pady=(10, 0))

lbl_consumption = tk.Label(card_right, text="0.0 л/100 км",
                           bg=COLORS["card"], font=("Arial", 18, "bold"))
lbl_consumption.pack(anchor="w")

tk.Label(card_right, text="Стоимость поездки",
         bg=COLORS["card"], fg=COLORS["muted"]).pack(anchor="w", pady=(15, 0))

lbl_cost = tk.Label(card_right, text="0 ₽",
                    bg=COLORS["card"], font=("Arial", 18, "bold"))
lbl_cost.pack(anchor="w")

# =========================
# ⚡ Быстрые действия
# =========================
card_actions = tk.Frame(content, bg=COLORS["card"], padx=20, pady=20)
card_actions.pack(side="left", fill="y", padx=10)

tk.Label(card_actions, text="Быстрые действия",
         bg=COLORS["card"], font=("Arial", 12, "bold")).pack(anchor="w")

tk.Button(card_actions, text="Новый расчёт").pack(fill="x", pady=5)
tk.Button(card_actions, text="Сохранить").pack(fill="x", pady=5)
tk.Button(card_actions, text="Показать график").pack(fill="x", pady=5)

# =========================
# ▶️ Старт
# =========================
root.mainloop()
