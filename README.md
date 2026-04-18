import tkinter as tk
from tkinter import ttk

# ======================
# 🎨 Цвета
# ======================
BG = "#eef2f7"
BLUE = "#2f6fed"
DARK = "#1e293b"
CARD = "#ffffff"
MUTED = "#94a3b8"

# ======================
# 🧮 Расчет
# ======================
def calculate():
    try:
        d = float(distance.get())
        f = float(fuel.get())
        p = float(price.get())

        cons = (f / d) * 100
        cost = f * p

        result_consumption.config(text=f"{cons:.1f} л/100 км")
        result_cost.config(text=f"{cost:.0f} ₽")
    except:
        result_consumption.config(text="Ошибка")
        result_cost.config(text="Ошибка")

def reset():
    distance.delete(0, tk.END)
    fuel.delete(0, tk.END)
    price.delete(0, tk.END)
    result_consumption.config(text="0.0 л/100 км")
    result_cost.config(text="0 ₽")

# ======================
# 🖥️ Окно
# ======================
root = tk.Tk()
root.title("CalculatCar")
root.geometry("1200x750")
root.configure(bg=BG)

# ======================
# 🔷 HEADER
# ======================
header = tk.Frame(root, bg=BLUE, height=70)
header.pack(fill="x")

tk.Label(header, text="☰", bg=BLUE, fg="white",
         font=("Arial", 20)).pack(side="left", padx=20)

tk.Label(header, text="CalculatCar", bg=BLUE, fg="white",
         font=("Arial", 22, "bold")).pack(side="top", pady=15)

# ======================
# 📦 КОНТЕЙНЕР
# ======================
container = tk.Frame(root, bg=BG)
container.pack(fill="both", expand=True)

# ======================
# 📚 SIDEBAR
# ======================
sidebar = tk.Frame(container, bg="#f8fafc", width=220)
sidebar.pack(side="left", fill="y")

def side_btn(text, active=False):
    return tk.Button(
        sidebar,
        text=text,
        anchor="w",
        bg=BLUE if active else "#f8fafc",
        fg="white" if active else DARK,
        relief="flat",
        padx=20,
        pady=12,
        font=("Arial", 11, "bold" if active else "normal")
    )

tk.Label(sidebar, text="👤", bg="#f8fafc", font=("Arial", 30)).pack(pady=20)
tk.Label(sidebar, text="Профиль", bg="#f8fafc").pack()

side_btn("🧮 Калькулятор", True).pack(fill="x", pady=10)
side_btn("🕒 История").pack(fill="x")
side_btn("ℹ️ О нас").pack(fill="x")

tk.Label(sidebar, text="⚙️ Настройки", bg="#f8fafc").pack(side="bottom", pady=20)

# ======================
# 🧩 MAIN
# ======================
main = tk.Frame(container, bg=BG)
main.pack(side="left", fill="both", expand=True, padx=20, pady=20)

tk.Label(main, text="Калькулятор расхода топлива",
         bg=BG, fg=DARK, font=("Arial", 20, "bold")).pack(anchor="w", pady=10)

content = tk.Frame(main, bg=BG)
content.pack(fill="both", expand=True)

# ======================
# 🧾 ЛЕВАЯ КАРТОЧКА
# ======================
left = tk.Frame(content, bg=CARD, padx=20, pady=20)
left.pack(side="left", padx=10, fill="y")

def field(label):
    tk.Label(left, text=label, bg=CARD, fg=DARK).pack(anchor="w", pady=(10, 2))
    e = tk.Entry(left, width=25)
    e.pack(pady=5)
    return e

distance = field("Дистанция (км):")
distance.insert(0, "150")

fuel = field("Кол-во топлива (л):")
fuel.insert(0, "12")

price = field("Цена топлива (₽/л):")
price.insert(0, "48.5")

tk.Button(left, text="Рассчитать",
          bg=BLUE, fg="white",
          font=("Arial", 12, "bold"),
          command=calculate).pack(fill="x", pady=20)

# ======================
# 📊 РЕЗУЛЬТАТЫ
# ======================
center = tk.Frame(content, bg=CARD, padx=20, pady=20)
center.pack(side="left", padx=10, fill="y")

tk.Label(center, text="Результаты",
         bg=CARD, font=("Arial", 14, "bold")).pack(anchor="w")

tk.Label(center, text="Расход топлива",
         bg=CARD, fg=MUTED).pack(anchor="w", pady=(10, 0))

result_consumption = tk.Label(center, text="8.0 л/100 км",
                              bg=CARD, font=("Arial", 20, "bold"))
result_consumption.pack(anchor="w")

tk.Label(center, text="Стоимость поездки",
         bg=CARD, fg=MUTED).pack(anchor="w", pady=(20, 0))

result_cost = tk.Label(center, text="720 ₽",
                       bg=CARD, font=("Arial", 20, "bold"))
result_cost.pack(anchor="w")

btns = tk.Frame(center, bg=CARD)
btns.pack(pady=20)

tk.Button(btns, text="Сбросить", command=reset).pack(side="left", padx=5)
tk.Button(btns, text="Копировать").pack(side="left", padx=5)

# ======================
# ⚡ БЫСТРЫЕ ДЕЙСТВИЯ
# ======================
right = tk.Frame(content, bg=CARD, padx=20, pady=20)
right.pack(side="left", padx=10, fill="y")

tk.Label(right, text="Быстрые действия",
         bg=CARD, font=("Arial", 12, "bold")).pack(anchor="w")

tk.Button(right, text="Новый расчёт").pack(fill="x", pady=5)
tk.Button(right, text="Сохранить").pack(fill="x", pady=5)
tk.Button(right, text="Показать график").pack(fill="x", pady=5)

# ======================
# 🕒 ПОСЛЕДНИЕ РАСЧЕТЫ
# ======================
history = tk.Frame(main, bg=CARD, padx=20, pady=20)
history.pack(fill="x", pady=20)

tk.Label(history, text="Последние расчёты",
         bg=CARD, font=("Arial", 12, "bold")).pack(anchor="w")

for txt in [
    "8.0 л/100 км — 720 ₽",
    "7.4 л/100 км — 666 ₽",
    "9.2 л/100 км — 850 ₽"
]:
    tk.Label(history, text=txt, bg=CARD).pack(anchor="w")

# ======================
root.mainloop()
