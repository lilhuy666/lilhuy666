import tkinter as tk
from tkinter import messagebox

# ===============================
# 🎨 Цвета (приближены к макету)
# ===============================
BG = "#eef1f5"
SIDEBAR = "#1f4aa8"
HEADER = "#2f5fb3"
CARD = "#ffffff"
TEXT = "#1f2937"
SUB = "#6b7280"
BLUE = "#3b82f6"

root = tk.Tk()
root.title("CalculatCar")
root.geometry("1200x750")
root.configure(bg=BG)

# ===============================
# 📌 SIDEBAR
# ===============================
sidebar = tk.Frame(root, bg=SIDEBAR, width=220)
sidebar.pack(side="left", fill="y")

def menu_item(text):
    tk.Button(
        sidebar,
        text=text,
        bg=SIDEBAR,
        fg="white",
        font=("Arial", 13),
        bd=0,
        anchor="w",
        padx=20,
        pady=12,
        activebackground="#1a3f8a"
    ).pack(fill="x")

tk.Label(sidebar, text="👤\nПрофиль",
         bg=SIDEBAR, fg="white",
         font=("Arial", 12)).pack(pady=20)

menu_item("Калькулятор")
menu_item("История")
menu_item("О нас")

# ===============================
# 📌 MAIN
# ===============================
main = tk.Frame(root, bg=BG)
main.pack(side="right", fill="both", expand=True)

# HEADER
header = tk.Frame(main, bg=HEADER, height=70)
header.pack(fill="x")

tk.Label(header, text="CalculatCar",
         bg=HEADER, fg="white",
         font=("Arial", 22, "bold")).pack(pady=15)

# ===============================
# 📌 CONTENT
# ===============================
content = tk.Frame(main, bg=BG)
content.pack(fill="both", expand=True, padx=20, pady=20)

content.grid_columnconfigure(0, weight=2)
content.grid_columnconfigure(1, weight=1)

# ===============================
# 🧮 ЛЕВАЯ КАРТОЧКА
# ===============================
left = tk.Frame(content, bg=CARD, padx=20, pady=20)
left.grid(row=0, column=0, sticky="nsew", padx=10, pady=10)

tk.Label(left, text="Калькулятор расхода топлива",
         bg=CARD, fg=TEXT,
         font=("Arial", 18, "bold")).pack(anchor="w", pady=10)

def field(parent, text):
    tk.Label(parent, text=text,
             bg=CARD, fg=SUB,
             font=("Arial", 11)).pack(anchor="w")
    e = tk.Entry(parent, font=("Arial", 13))
    e.pack(fill="x", pady=5)
    return e

distance = field(left, "Дистанция (км)")
fuel = field(left, "Кол-во топлива (л)")
price = field(left, "Цена топлива (₽/л)")

def calc():
    try:
        d = float(distance.get())
        f = float(fuel.get())
        p = float(price.get())

        cons = (f / d) * 100
        cost = f * p

        cons_label.config(text=f"{cons:.1f} л/100 км")
        cost_label.config(text=f"{cost:.0f} ₽")

    except:
        messagebox.showerror("Ошибка", "Проверь ввод")

tk.Button(left, text="Рассчитать",
          bg=BLUE, fg="white",
          font=("Arial", 13),
          command=calc).pack(fill="x", pady=15)

# ===============================
# 📊 ПРАВАЯ КАРТОЧКА (результат)
# ===============================
right = tk.Frame(content, bg=BG)
right.grid(row=0, column=1, sticky="nsew")

# RESULT CARD 1
card1 = tk.Frame(right, bg=CARD, padx=20, pady=20)
card1.pack(fill="x", pady=10)

tk.Label(card1, text="Расход топлива",
         bg=CARD, fg=SUB,
         font=("Arial", 11)).pack(anchor="w")

cons_label = tk.Label(card1,
                      text="—",
                      bg=CARD,
                      fg=TEXT,
                      font=("Arial", 22, "bold"))
cons_label.pack()

# RESULT CARD 2
card2 = tk.Frame(right, bg=CARD, padx=20, pady=20)
card2.pack(fill="x", pady=10)

tk.Label(card2, text="Стоимость поездки",
         bg=CARD, fg=SUB,
         font=("Arial", 11)).pack(anchor="w")

cost_label = tk.Label(card2,
                      text="—",
                      bg=CARD,
                      fg=TEXT,
                      font=("Arial", 22, "bold"))
cost_label.pack()

# ===============================
# 📉 ГРАФИК (заглушка)
# ===============================
graph = tk.Frame(content, bg=CARD, height=200)
graph.grid(row=1, column=0, columnspan=2,
           sticky="nsew", padx=10, pady=10)

tk.Label(graph, text="График расхода (пример)",
         bg=CARD, fg=SUB,
         font=("Arial", 12)).pack(pady=60)

root.mainloop()
