import tkinter as tk
from tkinter import ttk
from datetime import datetime
import matplotlib.pyplot as plt
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg

# ===================== ЛОГИКА =====================
history = []

def calculate():
    try:
        distance = float(entry_distance.get())
        fuel = float(entry_fuel.get())
        price = float(entry_price.get())

        consumption = (fuel / distance) * 100
        cost = fuel * price

        result_consumption.config(text=f"{consumption:.1f} л/100 км")
        result_cost.config(text=f"{cost:.0f} ₽")

        # сохраняем историю
        now = datetime.now().strftime("%H:%M")
        history.append((consumption, cost, now))
        update_history()
        update_graph()

    except:
        result_consumption.config(text="Ошибка")
        result_cost.config(text="Ошибка")

def update_history():
    history_box.delete(0, tk.END)
    for item in reversed(history[-5:]):
        history_box.insert(tk.END, f"{item[0]:.1f} л/100км — {item[1]:.0f} ₽ ({item[2]})")

def update_graph():
    ax.clear()
    values = [h[0] for h in history[-7:]]
    if values:
        ax.plot(values, marker="o")
        ax.set_title("График расхода")
    canvas.draw()

# ===================== UI =====================
root = tk.Tk()
root.title("CalculatCar")
root.geometry("1000x600")
root.configure(bg="#f0f2f5")

# ===== Верхняя панель =====
top = tk.Frame(root, bg="#2f6db3", height=60)
top.pack(fill="x")

title = tk.Label(top, text="CalculatCar", fg="white", bg="#2f6db3",
                 font=("Arial", 20, "bold"))
title.pack(pady=10)

# ===== Основной контейнер =====
main = tk.Frame(root, bg="#f0f2f5")
main.pack(fill="both", expand=True)

# ===== Сайдбар =====
sidebar = tk.Frame(main, bg="#e4e7eb", width=200)
sidebar.pack(side="left", fill="y")

buttons = ["Профиль", "Калькулятор", "История", "О нас"]
for b in buttons:
    tk.Button(sidebar, text=b, relief="flat", bg="#e4e7eb",
              font=("Arial", 12)).pack(fill="x", pady=10, padx=10)

# ===== Контент =====
content = tk.Frame(main, bg="#f0f2f5")
content.pack(side="left", fill="both", expand=True, padx=10, pady=10)

# ===== Левая часть (ввод) =====
left = tk.Frame(content, bg="white", bd=1, relief="solid")
left.grid(row=0, column=0, padx=10, pady=10, sticky="n")

tk.Label(left, text="Дистанция (км):").pack(pady=5)
entry_distance = tk.Entry(left)
entry_distance.pack()

tk.Label(left, text="Кол-во топлива (л):").pack(pady=5)
entry_fuel = tk.Entry(left)
entry_fuel.pack()

tk.Label(left, text="Цена топлива (₽):").pack(pady=5)
entry_price = tk.Entry(left)
entry_price.pack()

tk.Button(left, text="Рассчитать", bg="#2f6db3", fg="white",
          command=calculate).pack(pady=10)

# ===== Центр (результаты) =====
center = tk.Frame(content, bg="white", bd=1, relief="solid")
center.grid(row=0, column=1, padx=10, pady=10, sticky="n")

tk.Label(center, text="Расход топлива").pack(pady=5)
result_consumption = tk.Label(center, text="0.0 л/100 км", font=("Arial", 14))
result_consumption.pack()

tk.Label(center, text="Стоимость поездки").pack(pady=5)
result_cost = tk.Label(center, text="0 ₽", font=("Arial", 14))
result_cost.pack()

# ===== Правая часть (история) =====
right = tk.Frame(content, bg="white", bd=1, relief="solid")
right.grid(row=0, column=2, padx=10, pady=10, sticky="n")

tk.Label(right, text="История").pack()
history_box = tk.Listbox(right, width=30, height=10)
history_box.pack()

# ===== График =====
graph_frame = tk.Frame(content, bg="white", bd=1, relief="solid")
graph_frame.grid(row=1, column=0, columnspan=3, pady=10, sticky="ew")

fig, ax = plt.subplots(figsize=(6, 2))
canvas = FigureCanvasTkAgg(fig, master=graph_frame)
canvas.get_tk_widget().pack()

# ===== Запуск =====
root.mainloop()
