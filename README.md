import tkinter as tk
from tkinter import messagebox
from datetime import datetime

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

history_data = []

root = tk.Tk()
root.title("CalculatCar")
root.geometry("1200x750")
root.configure(bg=BG)

# ===============================
# SIDEBAR
# ===============================
sidebar = tk.Frame(root, bg=SIDEBAR, width=220)
sidebar.pack(side="left", fill="y")

main = tk.Frame(root, bg=BG)
main.pack(side="right", fill="both", expand=True)

# HEADER
header = tk.Frame(main, bg=HEADER, height=70)
header.pack(fill="x")

tk.Label(header, text="CalculatCar",
         bg=HEADER, fg="white",
         font=("Arial", 22, "bold")).pack(pady=15)

content = tk.Frame(main, bg=BG)
content.pack(fill="both", expand=True, padx=20, pady=20)

# ===============================
# CLEAR
# ===============================
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

    def field(text):
        tk.Label(left, text=text,
                 bg=CARD, fg=SUB).pack(anchor="w")
        e = tk.Entry(left)
        e.pack(fill="x", pady=5)
        return e

    distance = field("Дистанция (км)")
    fuel = field("Топливо (л)")
    price = field("Цена (₽)")

    right = tk.Frame(content, bg=BG)
    right.grid(row=0, column=1, sticky="nsew")

    card1 = tk.Frame(right, bg=CARD, padx=20, pady=20)
    card1.pack(fill="x", pady=10)

    cons_label = tk.Label(card1, text="—",
                          bg=CARD, font=("Arial", 20, "bold"))
    cons_label.pack()

    card2 = tk.Frame(right, bg=CARD, padx=20, pady=20)
    card2.pack(fill="x", pady=10)

    cost_label = tk.Label(card2, text="—",
                          bg=CARD, font=("Arial", 20, "bold"))
    cost_label.pack()

    def calc():
        try:
            d = float(distance.get())
            f = float(fuel.get())
            p = float(price.get())

            cons = (f / d) * 100
            cost = f * p

            cons_label.config(text=f"{cons:.1f} л/100км")
            cost_label.config(text=f"{cost:.0f} ₽")

            history_data.append({
                "date": datetime.now().strftime("%d.%m %H:%M"),
                "cons": cons,
                "cost": cost
            })

        except:
            messagebox.showerror("Ошибка", "Неверный ввод")

    tk.Button(left, text="Рассчитать",
              bg=BLUE, fg="white",
              command=calc).pack(fill="x", pady=10)

# ===============================
# 👤 ПРОФИЛЬ
# ===============================
def show_profile():
    clear()

    content.grid_columnconfigure(0, weight=1)
    content.grid_columnconfigure(1, weight=2)

    # ЛЕВАЯ КАРТОЧКА
    card_left = tk.Frame(content, bg=CARD, padx=20, pady=20)
    card_left.grid(row=0, column=0, sticky="n", padx=10, pady=10)

    tk.Label(card_left, text="👤",
             bg=CARD, font=("Arial", 50)).pack()

    tk.Label(card_left, text="Алексей Смирнов",
             bg=CARD, font=("Arial", 14, "bold")).pack()

    tk.Label(card_left, text="alex@email.com",
             bg=CARD, fg=SUB).pack()

    tk.Button(card_left, text="Редактировать",
              bg=BLUE, fg="white").pack(pady=10, fill="x")

    # ПРАВАЯ КАРТОЧКА
    card_right = tk.Frame(content, bg=CARD, padx=20, pady=20)
    card_right.grid(row=0, column=1, sticky="nsew", padx=10, pady=10)

    tk.Label(card_right, text="Личная информация",
             bg=CARD, font=("Arial", 16, "bold")).pack(anchor="w")

    def field(text, value):
        tk.Label(card_right, text=text,
                 bg=CARD, fg=SUB).pack(anchor="w")
        e = tk.Entry(card_right)
        e.insert(0, value)
        e.pack(fill="x", pady=5)

    field("Имя", "Алексей")
    field("Фамилия", "Смирнов")
    field("Email", "alex@email.com")
    field("Телефон", "+7 999 123 45 67")

# ===============================
# 📊 ИСТОРИЯ
# ===============================
def show_history():
    clear()

    tk.Label(content, text="История расчетов",
             bg=BG, fg=TEXT,
             font=("Arial", 18, "bold")).pack(anchor="w")

    table = tk.Frame(content, bg=CARD)
    table.pack(fill="both", expand=True, pady=10)

    for h in history_data[::-1]:
        row = tk.Frame(table, bg=CARD, pady=10)
        row.pack(fill="x", padx=10)

        tk.Label(row, text=h["date"],
                 bg=CARD, width=15).pack(side="left")

        tk.Label(row, text=f"{h['cons']:.1f} л/100км",
                 bg=CARD, width=15).pack(side="left")

        tk.Label(row, text=f"{h['cost']:.0f} ₽",
                 bg=CARD, width=10).pack(side="left")

        tk.Button(row, text="Удалить",
                  command=lambda r=row: r.destroy()).pack(side="right")

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

menu_btn("Профиль", show_profile)
menu_btn("Калькулятор", show_calc)
menu_btn("История", show_history)

# START
show_calc()
root.mainloop()
