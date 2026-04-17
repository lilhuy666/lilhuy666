import tkinter as tk
from tkinter import ttk, messagebox
from datetime import datetime

# ===============================
# 🎨 Зеленая тема
# ===============================
COLORS = {
    "primary": "#16a34a",
    "secondary": "#166534",
    "accent": "#22c55e",
    "background": "#052e16",
    "surface": "#064e3b",
    "text": "#ecfdf5",
}

# ===============================
# 💾 Данные
# ===============================
history = []

profile = {
    "name": "",
    "vehicle": "",
    "consumption": ""
}

# ===============================
# 🖥️ Окно
# ===============================
root = tk.Tk()
root.title("⛽ FuelCalc Green")
root.geometry("1100x700")
root.configure(bg=COLORS["background"])

style = ttk.Style()
style.theme_use('clam')

# ===============================
# Header
# ===============================
header = tk.Frame(root, bg=COLORS["surface"], height=80)
header.pack(fill="x")

tk.Label(header, text="⛽ FuelCalc Green",
         bg=COLORS["surface"], fg=COLORS["text"],
         font=("Arial", 20, "bold")).pack(pady=20)

# ===============================
# Layout
# ===============================
container = tk.Frame(root, bg=COLORS["background"])
container.pack(expand=True, fill="both", padx=20, pady=10)

menu_frame = tk.Frame(container, bg=COLORS["surface"], width=220)
menu_frame.pack(side="left", fill="y", padx=(0, 10))
menu_frame.pack_propagate(False)

main_frame = tk.Frame(container, bg=COLORS["background"])
main_frame.pack(side="left", expand=True, fill="both")

# ===============================
def clear_main():
    for w in main_frame.winfo_children():
        w.destroy()

# ===============================
# 👤 Профиль
# ===============================
def show_profile():
    clear_main()

    tk.Label(main_frame, text="👤 Профиль",
             bg=COLORS["background"], fg=COLORS["text"],
             font=("Arial", 20, "bold")).pack(pady=20)

    frame = tk.Frame(main_frame, bg=COLORS["surface"], padx=20, pady=20)
    frame.pack(pady=10)

    labels = ["Имя:", "Авто:", "Средний расход (л/100км):"]
    entries = []

    for i, text in enumerate(labels):
        tk.Label(frame, text=text,
                 bg=COLORS["surface"], fg=COLORS["text"]).grid(row=i, column=0, pady=10, sticky='w')

        entry = tk.Entry(frame, width=30,
                         bg=COLORS["secondary"], fg=COLORS["text"])
        entry.grid(row=i, column=1, pady=10)
        entries.append(entry)

    # автозаполнение
    entries[0].insert(0, profile["name"])
    entries[1].insert(0, profile["vehicle"])
    entries[2].insert(0, profile["consumption"])

    def save_profile():
        profile["name"] = entries[0].get()
        profile["vehicle"] = entries[1].get()
        profile["consumption"] = entries[2].get()

        messagebox.showinfo("Сохранено", "Профиль сохранен!")

    tk.Button(main_frame, text="💾 Сохранить",
              command=save_profile,
              bg=COLORS["accent"], fg=COLORS["text"],
              font=("Arial", 12, "bold")).pack(pady=20)

# ===============================
# 🧮 Калькулятор
# ===============================
def show_calc():
    clear_main()

    tk.Label(main_frame, text="🧮 Калькулятор топлива",
             bg=COLORS["background"], fg=COLORS["text"],
             font=("Arial", 20, "bold")).pack(pady=20)

    frame = tk.Frame(main_frame, bg=COLORS["surface"], padx=20, pady=20)
    frame.pack(pady=10)

    labels = [
        "📏 Расстояние (км):",
        "⛽ Расход (л/100 км):",
        "💰 Цена топлива:",
        "📊 Потрачено топлива (л): (для расчета на 100 км)"
    ]

    entries = []

    for i, text in enumerate(labels):
        tk.Label(frame, text=text,
                 bg=COLORS["surface"], fg=COLORS["text"]).grid(row=i, column=0, pady=10, sticky='w')

        entry = tk.Entry(frame, width=30,
                         bg=COLORS["secondary"], fg=COLORS["text"])
        entry.grid(row=i, column=1, pady=10)
        entries.append(entry)

    # автоподстановка расхода из профиля
    if profile["consumption"]:
        entries[1].insert(0, profile["consumption"])

    def calculate():
        try:
            distance = float(entries[0].get())
            consumption = entries[1].get()
            fuel_price = float(entries[2].get())
            fuel_used_input = entries[3].get()

            # если введен расход напрямую
            if consumption:
                consumption = float(consumption)
                fuel_used = (distance / 100) * consumption
            # если считаем расход на 100 км
            elif fuel_used_input:
                fuel_used = float(fuel_used_input)
                consumption = (fuel_used / distance) * 100
            else:
                messagebox.showerror("Ошибка", "Введите расход или потраченное топливо!")
                return

            total_cost = fuel_used * fuel_price

            record = {
                "name": profile["name"],
                "vehicle": profile["vehicle"],
                "distance": distance,
                "fuel_used": round(fuel_used, 2),
                "consumption": round(consumption, 2),
                "price": round(total_cost, 2),
                "date": datetime.now().strftime("%d.%m.%Y %H:%M")
            }

            history.append(record)

            messagebox.showinfo(
                "Результат",
                f"👤 {profile['name']}\n"
                f"🚗 {profile['vehicle']}\n"
                f"⛽ Расход: {consumption:.2f} л/100км\n"
                f"🔥 Потрачено: {fuel_used:.2f} л\n"
                f"💰 Стоимость: {total_cost:.2f} ₽"
            )

        except:
            messagebox.showerror("Ошибка", "Проверьте ввод!")

    tk.Button(main_frame, text="Рассчитать",
              command=calculate,
              bg=COLORS["accent"], fg=COLORS["text"],
              font=("Arial", 12, "bold"),
              width=20).pack(pady=20)

# ===============================
# 📊 История
# ===============================
def show_history():
    clear_main()

    tk.Label(main_frame, text="📊 История",
             bg=COLORS["background"], fg=COLORS["text"],
             font=("Arial", 20, "bold")).pack(pady=20)

    for h in history:
        tk.Label(main_frame,
                 text=f"{h['date']} | {h['name']} | {h['vehicle']} | "
                      f"{h['distance']} км | {h['fuel_used']} л | {h['price']} ₽",
                 bg=COLORS["background"], fg=COLORS["text"]).pack(anchor='w')

# ===============================
# 📋 Меню
# ===============================
tk.Label(menu_frame, text="Меню",
         bg=COLORS["surface"], fg=COLORS["accent"],
         font=("Arial", 14, "bold")).pack(pady=15)

tk.Button(menu_frame, text="🧮 Калькулятор",
          command=show_calc,
          bg=COLORS["surface"], fg=COLORS["text"]).pack(fill='x', pady=5, padx=10)

tk.Button(menu_frame, text="👤 Профиль",
          command=show_profile,
          bg=COLORS["surface"], fg=COLORS["text"]).pack(fill='x', pady=5, padx=10)

tk.Button(menu_frame, text="📊 История",
          command=show_history,
          bg=COLORS["surface"], fg=COLORS["text"]).pack(fill='x', pady=5, padx=10)

# ===============================
# 🚀 Старт
# ===============================
show_calc()
root.mainloop()
