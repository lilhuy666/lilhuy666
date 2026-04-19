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

DATA_FILE = "history.json"
history_data = []

# ===============================
# 💾 LOAD / SAVE
# ===============================
def load_data():
    global history_data
    if os.path.exists(DATA_FILE):
        with open(DATA_FILE, "r", encoding="utf-8") as f:
            history_data = json.load(f)

def save_data():
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(history_data, f, ensure_ascii=False, indent=2)

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
        tk.Label(left, text=text, bg=CARD, fg=SUB).pack(anchor="w")
        e = tk.Entry(left)
        e.pack(fill="x", pady=5)
        return e

    distance = field("Дистанция (км)")
    fuel = field("Топливо (л)")
    price = field("Цена (₽/л)")

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
                "date": datetime.now().strftime("%d.%m.%Y %H:%M"),
                "cons": round(cons, 2),
                "cost": round(cost, 2)
            })
            save_data()

        except:
            messagebox.showerror("Ошибка", "Проверь ввод")

    tk.Button(left, text="Рассчитать",
              bg=BLUE, fg="white",
              command=calc).pack(fill="x", pady=10)

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

    def delete_item(index):
        history_data.pop(index)
        save_data()
        show_history()

    for i, h in enumerate(history_data[::-1]):
        real_index = len(history_data) - 1 - i

        row = tk.Frame(table, bg=CARD, pady=10)
        row.pack(fill="x", padx=10)

        tk.Label(row, text=h["date"],
                 bg=CARD, width=20, anchor="w").pack(side="left")

        tk.Label(row, text=f"{h['cons']} л/100км",
                 bg=CARD, width=15).pack(side="left")

        tk.Label(row, text=f"{h['cost']} ₽",
                 bg=CARD, width=10).pack(side="left")

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
             bg=CARD, fg=TEXT,
             font=("Arial", 18, "bold")).pack()

    tk.Label(card,
             text="CalculatCar — простой калькулятор расхода топлива.\n"
                  "Создан для удобного расчета поездок 🚗",
             bg=CARD, fg=SUB,
             font=("Arial", 12),
             justify="center").pack(pady=10)

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
menu_btn("История", show_history)
menu_btn("О нас", show_about)

# ===============================
# START
# ===============================
load_data()
show_calc()
root.mainloop()
