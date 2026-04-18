import tkinter as tk
from tkinter import messagebox
from datetime import datetime

# ======================
# 🎨 Цвета
# ======================
BG = "#eef2f7"
BLUE = "#2f6fed"
DARK = "#1e293b"
CARD = "#ffffff"
MUTED = "#94a3b8"

history_data = []  # хранение истории

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
         font=("Arial", 22, "bold")).pack(pady=15)

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

main = tk.Frame(container, bg=BG)
main.pack(side="left", fill="both", expand=True, padx=20, pady=20)

def clear_main():
    for w in main.winfo_children():
        w.destroy()

# ======================
# 🧮 КАЛЬКУЛЯТОР
# ======================
def show_calc():
    clear_main()

    tk.Label(main, text="Калькулятор расхода топлива",
             bg=BG, fg=DARK, font=("Arial", 20, "bold")).pack(anchor="w", pady=10)

    content = tk.Frame(main, bg=BG)
    content.pack(fill="both", expand=True)

    # -------- ЛЕВО --------
    left = tk.Frame(content, bg=CARD, padx=20, pady=20)
    left.pack(side="left", padx=10, fill="y")

    def field(label):
        tk.Label(left, text=label, bg=CARD).pack(anchor="w")
        e = tk.Entry(left, width=25)
        e.pack(pady=5)
        return e

    distance = field("Дистанция (км):")
    fuel = field("Кол-во топлива (л):")
    price = field("Цена топлива (₽/л):")

    # -------- ЦЕНТР --------
    center = tk.Frame(content, bg=CARD, padx=20, pady=20)
    center.pack(side="left", padx=10, fill="y")

    tk.Label(center, text="Результаты",
             bg=CARD, font=("Arial", 14, "bold")).pack(anchor="w")

    tk.Label(center, text="Расход топлива",
             bg=CARD, fg=MUTED).pack(anchor="w", pady=(10, 0))

    result_consumption = tk.Label(center, text="0.0 л/100 км",
                                 bg=CARD, font=("Arial", 18, "bold"))
    result_consumption.pack(anchor="w")

    tk.Label(center, text="Стоимость поездки",
             bg=CARD, fg=MUTED).pack(anchor="w", pady=(15, 0))

    result_cost = tk.Label(center, text="0 ₽",
                           bg=CARD, font=("Arial", 18, "bold"))
    result_cost.pack(anchor="w")

    # -------- ИСТОРИЯ (последние) --------
    history_frame = tk.Frame(main, bg=CARD, padx=20, pady=20)
    history_frame.pack(fill="x", pady=20)

    tk.Label(history_frame, text="Последние расчёты",
             bg=CARD, font=("Arial", 12, "bold")).pack(anchor="w")

    def update_history_view():
        for w in history_frame.winfo_children()[1:]:
            w.destroy()

        for item in history_data[-5:][::-1]:
            tk.Label(history_frame,
                     text=f"{item['cons']} — {item['cost']} ({item['time']})",
                     bg=CARD).pack(anchor="w")

    # -------- ЛОГИКА --------
    def calculate():
        try:
            d = float(distance.get())
            f = float(fuel.get())
            p = float(price.get())

            cons = f / d * 100
            cost = f * p

            result_consumption.config(text=f"{cons:.1f} л/100 км")
            result_cost.config(text=f"{cost:.0f} ₽")

            record = {
                "cons": f"{cons:.1f} л/100 км",
                "cost": f"{cost:.0f} ₽",
                "time": datetime.now().strftime("%H:%M")
            }

            history_data.append(record)
            update_history_view()

        except:
            messagebox.showerror("Ошибка", "Проверь ввод")

    def reset():
        distance.delete(0, tk.END)
        fuel.delete(0, tk.END)
        price.delete(0, tk.END)

    tk.Button(left, text="Рассчитать",
              bg=BLUE, fg="white",
              command=calculate).pack(fill="x", pady=10)

    tk.Button(center, text="Сбросить",
              command=reset).pack(pady=10)

    update_history_view()

# ======================
# 🕒 ИСТОРИЯ
# ======================
def show_history():
    clear_main()

    tk.Label(main, text="История",
             bg=BG, font=("Arial", 20, "bold")).pack()

    for item in history_data[::-1]:
        tk.Label(main,
                 text=f"{item['cons']} — {item['cost']} ({item['time']})",
                 bg=BG).pack(anchor="w")

# ======================
# 👤 ПРОФИЛЬ
# ======================
def show_profile():
    clear_main()

    tk.Label(main, text="Профиль",
             bg=BG, font=("Arial", 20, "bold")).pack(pady=20)

    tk.Label(main, text="Имя пользователя: Гость",
             bg=BG).pack()

    tk.Label(main, text="Статус: Пользователь",
             bg=BG).pack()

# ======================
# ℹ️ О НАС
# ======================
def show_about():
    clear_main()

    tk.Label(main, text="О программе",
             bg=BG, font=("Arial", 20, "bold")).pack(pady=20)

    tk.Label(main,
             text="Добро пожаловать в CalculatCar!\n\n"
                  "Это приложение для расчёта расхода топлива и стоимости поездки.\n"
                  "Создано для удобства водителей 🚗",
             bg=BG, justify="left").pack()

# ======================
# 📚 КНОПКИ
# ======================
def btn(text, cmd):
    return tk.Button(sidebar, text=text,
                     command=cmd,
                     anchor="w",
                     bg="#f8fafc",
                     relief="flat",
                     padx=20,
                     pady=12)

tk.Label(sidebar, text="👤", bg="#f8fafc",
         font=("Arial", 30)).pack(pady=20)

btn("Профиль", show_profile).pack(fill="x")
btn("Калькулятор", show_calc).pack(fill="x")
btn("История", show_history).pack(fill="x")
btn("О нас", show_about).pack(fill="x")

# ======================
# ▶️ СТАРТ
# ======================
show_calc()
root.mainloop()
