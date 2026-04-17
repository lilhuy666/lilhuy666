import tkinter as tk
from tkinter import ttk, messagebox
from datetime import datetime

# ===============================
# 🎨 Цвета интерфейса
# ===============================
COLORS = {
    "primary": "#3b82f6",
    "secondary": "#1e40af",
    "accent": "#06b6d4",
    "background": "#0f172a",
    "surface": "#1e293b",
    "text": "#f8fafc",
    "text_secondary": "#94a3b8",
    "success": "#10b981",
}

# ===============================
# 💾 Данные
# ===============================
history = []
clients = []

# ===============================
# 🖥️ Главное окно
# ===============================
root = tk.Tk()
root.title("⛽ FuelCalc Pro - Калькулятор расхода топлива")
root.geometry("1100x700")
root.configure(bg=COLORS["background"])

style = ttk.Style()
style.theme_use('clam')

# ===============================
# Header
# ===============================
header = tk.Frame(root, bg=COLORS["surface"], height=80)
header.pack(fill="x")

tk.Label(header, text="⛽ FuelCalc Pro",
         bg=COLORS["surface"], fg=COLORS["text"],
         font=("Arial", 20, "bold")).pack(pady=20)

# ===============================
# Основной контейнер
# ===============================
container = tk.Frame(root, bg=COLORS["background"])
container.pack(expand=True, fill="both", padx=20, pady=10)

menu_frame = tk.Frame(container, bg=COLORS["surface"], width=220)
menu_frame.pack(side="left", fill="y", padx=(0, 10))
menu_frame.pack_propagate(False)

main_frame = tk.Frame(container, bg=COLORS["background"])
main_frame.pack(side="left", expand=True, fill="both")

# ===============================
# Очистка экрана
# ===============================
def clear_main_frame():
    for widget in main_frame.winfo_children():
        widget.destroy()

# ===============================
# 📈 Статистика
# ===============================
stats_frame = tk.Frame(menu_frame, bg=COLORS["surface"])
stats_frame.pack(pady=10, padx=10, fill='x')

def update_stats():
    for widget in stats_frame.winfo_children():
        widget.destroy()

    total_clients = len(clients)
    total_orders = len(history)
    total_fuel = sum(h['fuel_used'] for h in history)

    tk.Label(stats_frame, text="📈 Статистика",
             bg=COLORS["surface"], fg=COLORS["accent"],
             font=("Arial", 12, "bold")).pack(anchor='w', pady=5)

    data = [
        ("👥 Клиенты:", total_clients),
        ("🚗 Поездки:", total_orders),
        ("⛽ Топливо:", f"{total_fuel:.1f} л")
    ]

    for label, value in data:
        frame = tk.Frame(stats_frame, bg=COLORS["surface"])
        frame.pack(fill='x', pady=2)

        tk.Label(frame, text=label,
                 bg=COLORS["surface"], fg=COLORS["text_secondary"]).pack(side='left')
        tk.Label(frame, text=value,
                 bg=COLORS["surface"], fg=COLORS["text"]).pack(side='right')

# ===============================
# 🧮 Калькулятор
# ===============================
def show_calc():
    clear_main_frame()

    tk.Label(main_frame, text="🧮 Расчет расхода топлива",
             bg=COLORS["background"], fg=COLORS["text"],
             font=("Arial", 20, "bold")).pack(pady=20)

    form = tk.Frame(main_frame, bg=COLORS["surface"], padx=20, pady=20)
    form.pack(pady=10)

    labels = [
        "📏 Расстояние (км):",
        "⛽ Расход (л/100 км):",
        "💰 Цена топлива:",
        "🚗 Тип транспорта:",
        "👤 Клиент:"
    ]

    entries = []

    for i, text in enumerate(labels):
        tk.Label(form, text=text,
                 bg=COLORS["surface"], fg=COLORS["text"]).grid(row=i, column=0, pady=10, sticky='w')

        if i == 3:
            combo = ttk.Combobox(form,
                                 values=["Легковой", "SUV", "Грузовик", "Фура"],
                                 state="readonly", width=25)
            combo.grid(row=i, column=1, pady=10)
            entries.append(combo)
        else:
            entry = tk.Entry(form, width=28,
                             bg=COLORS["secondary"], fg=COLORS["text"])
            entry.grid(row=i, column=1, pady=10)
            entries.append(entry)

    def calculate():
        try:
            distance = float(entries[0].get())
            consumption = float(entries[1].get())
            fuel_price = float(entries[2].get())
            vehicle = entries[3].get()
            client = entries[4].get()

            if not all([distance, consumption, fuel_price, vehicle, client]):
                messagebox.showwarning("Ошибка", "Заполните все поля!")
                return

            fuel_used = (distance / 100) * consumption
            total_cost = fuel_used * fuel_price

            record = {
                "client": client,
                "vehicle": vehicle,
                "distance": distance,
                "fuel_used": round(fuel_used, 2),
                "price": round(total_cost, 2),
                "date": datetime.now().strftime("%d.%m.%Y %H:%M")
            }

            history.append(record)

            if client not in clients:
                clients.append(client)

            update_stats()

            messagebox.showinfo(
                "Результат",
                f"🚗 {vehicle}\n"
                f"⛽ Расход: {fuel_used:.2f} л\n"
                f"💰 Стоимость: {total_cost:.2f} ₽"
            )

        except:
            messagebox.showerror("Ошибка", "Введите корректные данные!")

    tk.Button(main_frame, text="Рассчитать",
              command=calculate,
              bg=COLORS["accent"], fg=COLORS["text"],
              font=("Arial", 12, "bold"),
              width=20).pack(pady=20)

# ===============================
# 👥 Клиенты
# ===============================
def show_clients():
    clear_main_frame()

    tk.Label(main_frame, text="👥 Клиенты",
             bg=COLORS["background"], fg=COLORS["text"],
             font=("Arial", 20, "bold")).pack(pady=20)

    for i, c in enumerate(clients, 1):
        tk.Label(main_frame, text=f"{i}. {c}",
                 bg=COLORS["background"], fg=COLORS["text"]).pack(anchor='w')

# ===============================
# 📊 История
# ===============================
def show_history():
    clear_main_frame()

    tk.Label(main_frame, text="📊 История поездок",
             bg=COLORS["background"], fg=COLORS["text"],
             font=("Arial", 20, "bold")).pack(pady=20)

    for h in history:
        tk.Label(main_frame,
                 text=f"{h['client']} | {h['vehicle']} | {h['distance']} км | "
                      f"{h['fuel_used']} л | {h['price']} ₽",
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

tk.Button(menu_frame, text="👥 Клиенты",
          command=show_clients,
          bg=COLORS["surface"], fg=COLORS["text"]).pack(fill='x', pady=5, padx=10)

tk.Button(menu_frame, text="📊 История",
          command=show_history,
          bg=COLORS["surface"], fg=COLORS["text"]).pack(fill='x', pady=5, padx=10)

# ===============================
# 🚀 Старт
# ===============================
show_calc()
update_stats()

root.mainloop()
