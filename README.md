import tkinter as tk
from tkinter import messagebox
from datetime import datetime

# ===============================
# 🎨 ТЕМА
# ===============================
BG = "#eef2f5"
CARD = "#ffffff"
ACCENT = "#2563eb"
TEXT = "#111827"

root = tk.Tk()
root.title("Fuel Calculator")
root.geometry("900x700")
root.configure(bg=BG)

# ===============================
# HERO БАННЕР
# ===============================
hero = tk.Frame(root, bg="#1f2937", height=200)
hero.pack(fill="x")

tk.Label(hero,
         text="Калькулятор\nрасхода топлива",
         bg="#1f2937",
         fg="white",
         font=("Arial", 28, "bold"),
         justify="left").pack(anchor="w", padx=30, pady=40)

# ===============================
# КАРТОЧКА
# ===============================
card = tk.Frame(root, bg=CARD, padx=30, pady=30)
card.pack(pady=20, padx=40, fill="both")

# ===============================
# ВЫБОР ТРАНСПОРТА
# ===============================
vehicle = tk.StringVar(value="car")

btn_frame = tk.Frame(card, bg=CARD)
btn_frame.pack(pady=10)

def vehicle_btn(text, val):
    return tk.Radiobutton(btn_frame,
                          text=text,
                          variable=vehicle,
                          value=val,
                          indicatoron=0,
                          width=12,
                          padx=10,
                          pady=10,
                          bg="#e5e7eb",
                          selectcolor="#93c5fd",
                          font=("Arial", 12))

vehicle_btn("🚗 Авто", "car").pack(side="left", padx=5)
vehicle_btn("🏍 Мото", "moto").pack(side="left", padx=5)
vehicle_btn("🚚 Грузовик", "truck").pack(side="left", padx=5)

# ===============================
# ПОЛЯ
# ===============================
def field(label, placeholder=""):
    frame = tk.Frame(card, bg=CARD)
    frame.pack(fill="x", pady=10)

    tk.Label(frame, text=label,
             bg=CARD, fg=TEXT,
             font=("Arial", 12, "bold")).pack(anchor="w")

    entry = tk.Entry(frame, font=("Arial", 14))
    entry.pack(fill="x", pady=5)

    return entry

distance = field("Пройденное расстояние (км):")
fuel = field("Израсходованное топливо (литры):")

# ===============================
# РЕЗУЛЬТАТ
# ===============================
result_var = tk.StringVar()

result_box = tk.Label(card,
                      textvariable=result_var,
                      bg="#f3f4f6",
                      fg=TEXT,
                      font=("Arial", 14),
                      pady=10)
result_box.pack(fill="x", pady=15)

# ===============================
# КНОПКА
# ===============================
def calculate():
    try:
        d = float(distance.get())
        f = float(fuel.get())

        consumption = (f / d) * 100

        now = datetime.now().strftime("%d.%m.%Y %H:%M")

        result_var.set(f"Расход: {consumption:.2f} л/100 км\n{now}")

    except:
        messagebox.showerror("Ошибка", "Проверь ввод")

tk.Button(card,
          text="Рассчитать",
          command=calculate,
          bg=ACCENT,
          fg="white",
          font=("Arial", 14, "bold"),
          pady=10).pack(fill="x")

# ===============================
# СТАРТ
# ===============================
root.mainloop()
