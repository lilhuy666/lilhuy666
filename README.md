
import tkinter as tk
from tkinter import ttk, messagebox
import random
from datetime import datetime

# ===============================
# 🚗 Транспорт и расход
# ===============================
vehicles = {
    "Легковой автомобиль": {"consumption": 8},   # л / 100 км
    "Кроссовер": {"consumption": 10},
    "Внедорожник": {"consumption": 14},
    "Грузовик": {"consumption": 25},
    "Мотоцикл": {"consumption": 5}
}

fuel_types = {
    "АИ-92": 52,
    "АИ-95": 57,
    "Дизель": 60
}

history = []

# ===============================
# 🎨 UI цвета
# ===============================
COLORS = {
    "bg": "#0f172a",
    "surface": "#1e293b",
    "accent": "#06b6d4",
    "text": "#f8fafc",
    "secondary": "#334155",
    "success": "#10b981"
}

# ===============================
# 🖥️ Окно
# ===============================
root = tk.Tk()
root.title("⛽ FuelCalc Pro")
root.geometry("1100x700")
root.configure(bg=COLORS["bg"])

# ===============================
# Контейнеры
# ===============================
container = tk.Frame(root, bg=COLORS["bg"])
container.pack(fill="both", expand=True)

menu = tk.Frame(container, bg=COLORS["surface"], width=220)
menu.pack(side="left", fill="y")

main = tk.Frame(container, bg=COLORS["bg"])
main.pack(side="left", expand=True, fill="both")

# ===============================
# Очистка
# ===============================
def clear():
    for w in main.winfo_children():
        w.destroy()

# ===============================
# 📊 Статистика
# ===============================
stats_frame = tk.Frame(menu, bg=COLORS["surface"])
stats_frame.pack(pady=20)

def update_stats():
    for w in stats_frame.winfo_children():
        w.destroy()

    total = len(history)
    fuel_sum = sum(h["fuel"] for h in history)
    cost_sum = sum(h["cost"] for h in history)

    tk.Label(stats_frame, text="📊 Статистика",
             bg=COLORS["surface"], fg=COLORS["accent"]).pack()

    tk.Label(stats_frame, text=f"Поездки: {total}",
             bg=COLORS["surface"], fg=COLORS["text"]).pack()

    tk.Label(stats_frame, text=f"Топливо: {fuel_sum:.1f} л",
             bg=COLORS["surface"], fg=COLORS["text"]).pack()

    tk.Label(stats_frame, text=f"Затраты: {cost_sum:.0f} ₽",
             bg=COLORS["surface"], fg=COLORS["text"]).pack()

# ===============================
# 🧮 Расчёт
# ===============================
def show_calc():
    clear()

    frame = tk.Frame(main, bg=COLORS["bg"])
    frame.pack(pady=40)

    tk.Label(frame, text="⛽ Калькулятор расхода топлива",
             bg=COLORS["bg"], fg=COLORS["text"],
             font=("Arial", 20, "bold")).pack(pady=20)

    # Поля
    vehicle = ttk.Combobox(frame, values=list(vehicles.keys()), width=30)
    vehicle.pack(pady=10)

    distance = tk.Entry(frame, width=33)
    distance.insert(0, "Дистанция (км)")
    distance.pack(pady=10)

    fuel_type = ttk.Combobox(frame, values=list(fuel_types.keys()), width=30)
    fuel_type.pack(pady=10)

    # Результат
    result = tk.Label(frame, text="", bg=COLORS["bg"],
                      fg=COLORS["accent"], font=("Arial", 14))
    result.pack(pady=20)

    def calc():
        try:
            v = vehicle.get()
            d = float(distance.get())
            f = fuel_type.get()

            if not v or not f:
                raise ValueError

            cons = vehicles[v]["consumption"]
            fuel_used = d * cons / 100
            cost = fuel_used * fuel_types[f]

            record = {
                "vehicle": v,
                "distance": d,
                "fuel": fuel_used,
                "cost": cost,
                "date": datetime.now().strftime("%d.%m.%Y %H:%M")
            }
            history.append(record)
            update_stats()

            result.config(
                text=f"🚗 Топливо: {fuel_used:.2f} л\n💰 Стоимость: {cost:.2f} ₽"
            )

        except:
            messagebox.showerror("Ошибка", "Проверьте ввод данных")

    tk.Button(frame, text="Рассчитать", command=calc,
              bg=COLORS["accent"], fg=COLORS["text"],
              width=20, height=2).pack()

# ===============================
# 📊 История
# ===============================
def show_history():
    clear()

    tk.Label(main, text="📊 История поездок",
             bg=COLORS["bg"], fg=COLORS["text"],
             font=("Arial", 20)).pack(pady=20)

    frame = tk.Frame(main, bg=COLORS["surface"])
    frame.pack(fill="both", expand=True, padx=30, pady=20)

    for h in history:
        tk.Label(frame,
                 text=f"{h['date']} | {h['vehicle']} | {h['distance']} км | {h['fuel']:.2f} л | {h['cost']:.0f} ₽",
                 bg=COLORS["surface"], fg=COLORS["text"]).pack(anchor="w")

# ===============================
# 🚗 Транспорт
# ===============================
def show_vehicles():
    clear()

    tk.Label(main, text="🚗 Транспорт",
             bg=COLORS["bg"], fg=COLORS["text"],
             font=("Arial", 20)).pack(pady=20)

    for name, data in vehicles.items():
        tk.Label(main,
                 text=f"{name} — {data['consumption']} л/100км",
                 bg=COLORS["bg"], fg=COLORS["text"]).pack(pady=5)

# ===============================
# 📌 Меню
# ===============================
buttons = [
    ("🧮 Калькулятор", show_calc),
    ("📊 История", show_history),
    ("🚗 Транспорт", show_vehicles)
]

for text, cmd in buttons:
    tk.Button(menu, text=text, command=cmd,
              bg=COLORS["surface"], fg=COLORS["text"],
              width=20, height=2).pack(pady=10)

# ===============================
# Старт
# ===============================
show_calc()
update_stats()
root.mainloop()
