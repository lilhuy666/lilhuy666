import tkinter as tk
from tkinter import messagebox
from datetime import datetime
import matplotlib.pyplot as plt
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg

# ======================
# 🎨 Цвета
# ======================
BG = "#eef2f7"
BLUE = "#2f6fed"
DARK = "#1e293b"
CARD = "#ffffff"
MUTED = "#94a3b8"

history_data = []

# ======================
# 🖥️ Окно
# ======================
root = tk.Tk()
root.title("CalculatCar")
root.geometry("1200x750")
root.configure(bg=BG)

# ======================
# HEADER
# ======================
header = tk.Frame(root, bg=BLUE, height=70)
header.pack(fill="x")

tk.Label(header, text="☰", bg=BLUE, fg="white",
         font=("Arial", 20)).pack(side="left", padx=20)

tk.Label(header, text="CalculatCar", bg=BLUE, fg="white",
         font=("Arial", 22, "bold")).pack(pady=15)

# ======================
# КОНТЕЙНЕР
# ======================
container = tk.Frame(root, bg=BG)
container.pack(fill="both", expand=True)

sidebar = tk.Frame(container, bg="#f8fafc", width=220)
sidebar.pack(side="left", fill="y")

main = tk.Frame(container, bg=BG)
main.pack(side="left", fill="both", expand=True, padx=20, pady=20)

def clear_main():
    for w in main.winfo_children():
        w.destroy()

# ======================
# КАЛЬКУЛЯТОР
# ======================
def show_calc():
    clear_main()

    mode = tk.StringVar(value="consumption")

    tk.Label(main, text="Калькулятор",
             bg=BG, font=("Arial", 20, "bold")).pack(anchor="w")

    # 🔄 ПЕРЕКЛЮЧАТЕЛЬ
    switch_frame = tk.Frame(main, bg=BG)
    switch_frame.pack(anchor="w", pady=10)

    tk.Radiobutton(switch_frame, text="Расход на 100 км",
                   variable=mode, value="consumption",
                   bg=BG).pack(side="left")

    tk.Radiobutton(switch_frame, text="Сколько потрачу",
                   variable=mode, value="trip",
                   bg=BG).pack(side="left", padx=20)

    content = tk.Frame(main, bg=BG)
    content.pack(fill="both", expand=True)

    # -------- ЛЕВО --------
    left = tk.Frame(content, bg=CARD, padx=20, pady=20)
    left.pack(side="left", padx=10)

    def field(label):
        tk.Label(left, text=label, bg=CARD).pack(anchor="w")
        e = tk.Entry(left, width=18)  # 👈 УКОРОЧЕНО
        e.pack(pady=5)
        return e

    distance = field("Дистанция (км):")
    fuel = field("Топливо (л):")
    price = field("Цена (₽/л):")
    trip_distance = field("План дистанция (км):")

    # -------- ЦЕНТР --------
    center = tk.Frame(content, bg=CARD, padx=20, pady=20)
    center.pack(side="left", padx=10)

    result_text = tk.Label(center, text="0",
                           bg=CARD, font=("Arial", 22, "bold"))
    result_text.pack()

    # -------- ПРАВО (ГРАФИК) --------
    right = tk.Frame(content, bg=CARD, padx=10, pady=10)
    right.pack(side="left", padx=10)

    fig, ax = plt.subplots()
    canvas = FigureCanvasTkAgg(fig, master=right)
    canvas.get_tk_widget().pack()

    # -------- ЛОГИКА --------
    def calculate():
        try:
            d = float(distance.get())
            f = float(fuel.get())
            p = float(price.get())
            td = float(trip_distance.get() or 0)

            ax.clear()

            if mode.get() == "consumption":
                cons = f / d * 100
                result_text.config(text=f"{cons:.1f} л/100км")

                ax.plot([0, d], [0, f])
                ax.set_title("Расход топлива")

            else:
                cons = f / d * 100
                needed = cons / 100 * td
                cost = needed * p

                result_text.config(text=f"{cost:.0f} ₽")

                ax.plot([0, td], [0, cost])
                ax.set_title("Стоимость поездки")

            canvas.draw()

        except:
            messagebox.showerror("Ошибка", "Проверь ввод")

    def reset():
        for e in [distance, fuel, price, trip_distance]:
            e.delete(0, tk.END)

        result_text.config(text="0")
        ax.clear()
        canvas.draw()

    tk.Button(left, text="Рассчитать",
              bg=BLUE, fg="white",
              command=calculate).pack(fill="x", pady=10)

    tk.Button(left, text="Сброс",
              command=reset).pack(fill="x")

# ======================
# СТРАНИЦЫ
# ======================
def show_history():
    clear_main()
    tk.Label(main, text="История",
             font=("Arial", 20), bg=BG).pack()

def show_profile():
    clear_main()
    tk.Label(main, text="Профиль",
             font=("Arial", 20), bg=BG).pack()

def show_about():
    clear_main()
    tk.Label(main, text="О программе",
             font=("Arial", 20), bg=BG).pack()

# ======================
# КНОПКИ
# ======================
def btn(text, cmd):
    return tk.Button(sidebar, text=text,
                     command=cmd,
                     anchor="w",
                     bg="#f8fafc",
                     relief="flat",
                     padx=20,
                     pady=12)

btn("Профиль", show_profile).pack(fill="x")
btn("Калькулятор", show_calc).pack(fill="x")
btn("История", show_history).pack(fill="x")
btn("О нас", show_about).pack(fill="x")

# ======================
# СТАРТ
# ======================
show_calc()
root.mainloop()
