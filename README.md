import tkinter as tk
from datetime import datetime

history = []

# ================= ЛОГИКА =================
def calculate():
    try:
        distance = float(entry_distance.get())
        fuel = float(entry_fuel.get())
        price = float(entry_price.get())

        consumption = (fuel / distance) * 100
        cost = fuel * price

        result_consumption.config(text=f"{consumption:.1f} л/100 км")
        result_cost.config(text=f"{cost:.0f} ₽")

        now = datetime.now().strftime("%H:%M")
        history.append(consumption)

        update_history(consumption, cost, now)
        draw_graph()

    except:
        result_consumption.config(text="Ошибка")
        result_cost.config(text="Ошибка")


def update_history(cons, cost, time):
    history_box.insert(0, f"{cons:.1f} л/100км — {cost:.0f} ₽ ({time})")
    if history_box.size() > 5:
        history_box.delete(5)


# ================= ГРАФИК =================
def draw_graph():
    canvas.delete("all")

    if len(history) < 2:
        return

    width = 400
    height = 150
    max_val = max(history)

    step_x = width / (len(history) - 1)

    for i in range(len(history) - 1):
        x1 = i * step_x
        y1 = height - (history[i] / max_val * height)

        x2 = (i + 1) * step_x
        y2 = height - (history[i+1] / max_val * height)

        canvas.create_line(x1, y1, x2, y2, width=2)


# ================= UI =================
root = tk.Tk()
root.title("CalculatCar")
root.geometry("900x600")
root.configure(bg="#f0f2f5")

# Верх
top = tk.Frame(root, bg="#2f6db3", height=60)
top.pack(fill="x")

tk.Label(top, text="CalculatCar", fg="white", bg="#2f6db3",
         font=("Arial", 20, "bold")).pack(pady=10)

# Основной блок
main = tk.Frame(root, bg="#f0f2f5")
main.pack(fill="both", expand=True)

# Сайдбар
sidebar = tk.Frame(main, bg="#e4e7eb", width=200)
sidebar.pack(side="left", fill="y")

for text in ["Профиль", "Калькулятор", "История", "О нас"]:
    tk.Button(sidebar, text=text, relief="flat", bg="#e4e7eb",
              font=("Arial", 12)).pack(fill="x", pady=10, padx=10)

# Контент
content = tk.Frame(main, bg="#f0f2f5")
content.pack(side="left", fill="both", expand=True, padx=10, pady=10)

# Ввод
frame_input = tk.Frame(content, bg="white", padx=10, pady=10)
frame_input.grid(row=0, column=0, padx=10)

tk.Label(frame_input, text="Дистанция (км):").pack()
entry_distance = tk.Entry(frame_input)
entry_distance.pack()

tk.Label(frame_input, text="Топливо (л):").pack()
entry_fuel = tk.Entry(frame_input)
entry_fuel.pack()

tk.Label(frame_input, text="Цена (₽):").pack()
entry_price = tk.Entry(frame_input)
entry_price.pack()

tk.Button(frame_input, text="Рассчитать", bg="#2f6db3",
          fg="white", command=calculate).pack(pady=10)

# Результаты
frame_result = tk.Frame(content, bg="white", padx=10, pady=10)
frame_result.grid(row=0, column=1, padx=10)

tk.Label(frame_result, text="Расход:").pack()
result_consumption = tk.Label(frame_result, text="0.0 л/100 км", font=("Arial", 14))
result_consumption.pack()

tk.Label(frame_result, text="Стоимость:").pack()
result_cost = tk.Label(frame_result, text="0 ₽", font=("Arial", 14))
result_cost.pack()

# История
frame_history = tk.Frame(content, bg="white", padx=10, pady=10)
frame_history.grid(row=0, column=2, padx=10)

tk.Label(frame_history, text="История").pack()
history_box = tk.Listbox(frame_history, width=30, height=10)
history_box.pack()

# График
frame_graph = tk.Frame(content, bg="white")
frame_graph.grid(row=1, column=0, columnspan=3, pady=20)

canvas = tk.Canvas(frame_graph, width=400, height=150, bg="white")
canvas.pack()

# Запуск
root.mainloop()
