import tkinter as tk
from tkinter import messagebox
from datetime import datetime

# ===============================
# 🎨 Цвета
# ===============================
COLORS = {
    "bg": "#eef2f7",
    "sidebar": "#ffffff",
    "header": "#2b6cb0",
    "card": "#ffffff",
    "text": "#1a202c",
    "gray": "#718096",
    "accent": "#3b82f6"
}

history = []

# ===============================
# 🖥️ Окно
# ===============================
root = tk.Tk()
root.title("CalculatCar")
root.geometry("1200x750")
root.configure(bg=COLORS["bg"])

# ===============================
# HEADER
# ===============================
header = tk.Frame(root, bg=COLORS["header"], height=70)
header.pack(fill="x")

tk.Label(header, text="CalculatCar",
         bg=COLORS["header"], fg="white",
         font=("Arial", 22, "bold")).pack(pady=15)

# ===============================
# ОСНОВА
# ===============================
container = tk.Frame(root, bg=COLORS["bg"])
container.pack(fill="both", expand=True)

# SIDEBAR
sidebar = tk.Frame(container, bg=COLORS["sidebar"], width=220)
sidebar.pack(side="left", fill="y")

tk.Label(sidebar, text="👤\nПрофиль",
         bg=COLORS["sidebar"], font=("Arial", 12)).pack(pady=20)

tk.Button(sidebar, text="🧮 Калькулятор", bg=COLORS["sidebar"],
          relief="flat").pack(fill="x", pady=5)

tk.Button(sidebar, text="🕒 История", bg=COLORS["sidebar"],
          relief="flat").pack(fill="x", pady=5)

tk.Button(sidebar, text="ℹ️ О нас", bg=COLORS["sidebar"],
          relief="flat").pack(fill="x", pady=5)

# MAIN
main = tk.Frame(container, bg=COLORS["bg"])
main.pack(side="left", expand=True, fill="both", padx=20, pady=20)

tk.Label(main, text="Калькулятор расхода топлива",
         bg=COLORS["bg"], fg=COLORS["text"],
         font=("Arial", 18, "bold")).pack(anchor="w", pady=10)

content = tk.Frame(main, bg=COLORS["bg"])
content.pack()

# ===============================
# КАРТОЧКА ВВОДА
# ===============================
left = tk.Frame(content, bg=COLORS["card"], padx=20, pady=20)
left.grid(row=0, column=0, padx=10)

tk.Label(left, text="Дистанция (км):",
         bg="white").pack(anchor="w")
entry_distance = tk.Entry(left)
entry_distance.insert(0, "150")
entry_distance.pack(fill="x", pady=5)

tk.Label(left, text="Кол-во топлива (л):",
         bg="white").pack(anchor="w")
entry_fuel = tk.Entry(left)
entry_fuel.insert(0, "12")
entry_fuel.pack(fill="x", pady=5)

tk.Label(left, text="Цена топлива (₽/л):",
         bg="white").pack(anchor="w")
entry_price = tk.Entry(left)
entry_price.insert(0, "48.5")
entry_price.pack(fill="x", pady=5)

# ===============================
# РЕЗУЛЬТАТЫ
# ===============================
right = tk.Frame(content, bg=COLORS["card"], padx=20, pady=20)
right.grid(row=0, column=1, padx=10)

result_label = tk.Label(right, text="0.0 л/100 км",
                        bg="white", font=("Arial", 16, "bold"))
result_label.pack(pady=10)

cost_label = tk.Label(right, text="0 ₽",
                      bg="white", font=("Arial", 18, "bold"))
cost_label.pack(pady=10)

# ===============================
# ИСТОРИЯ
# ===============================
history_frame = tk.Frame(content, bg=COLORS["card"], padx=15, pady=15)
history_frame.grid(row=0, column=2, padx=10)

tk.Label(history_frame, text="Последние расчёты",
         bg="white", font=("Arial", 12, "bold")).pack(anchor="w")

history_list = tk.Frame(history_frame, bg="white")
history_list.pack()

def update_history():
    for w in history_list.winfo_children():
        w.destroy()

    for item in history[-5:][::-1]:
        tk.Label(history_list, text=item,
                 bg="white").pack(anchor="w")

# ===============================
# РАСЧЕТ
# ===============================
def calculate():
    try:
        d = float(entry_distance.get())
        f = float(entry_fuel.get())
        p = float(entry_price.get())

        consumption = f / d * 100
        cost = f * p

        result_label.config(text=f"{consumption:.1f} л/100 км")
        cost_label.config(text=f"{cost:.0f} ₽")

        record = f"{consumption:.1f} л/100км — {cost:.0f} ₽"
        history.append(record)

        update_history()

    except:
        messagebox.showerror("Ошибка", "Проверь ввод!")

tk.Button(left, text="Рассчитать",
          command=calculate,
          bg=COLORS["accent"], fg="white",
          height=2).pack(fill="x", pady=10)

# ===============================
# ПРОСТОЙ ГРАФИК
# ===============================
canvas = tk.Canvas(main, height=150, bg="white")
canvas.pack(fill="x", pady=20)

points = [8, 10, 8, 9, 10, 8, 7]

def draw_graph():
    canvas.delete("all")
    step = 100

    for i in range(len(points)-1):
        x1 = i * step + 20
        y1 = 140 - points[i]*10
        x2 = (i+1)*step + 20
        y2 = 140 - points[i+1]*10

        canvas.create_line(x1, y1, x2, y2, width=2)

draw_graph()

root.mainloop()
