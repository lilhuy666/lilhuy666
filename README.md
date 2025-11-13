import tkinter as tk
from tkinter import ttk, messagebox
import math
import random

# ==============================
# 🌍 Координаты основных городов России
# ==============================

cities_coords = {
    "Москва": (55.7558, 37.6176),
    "Санкт-Петербург": (59.9343, 30.3351),
    "Новосибирск": (55.0084, 82.9357),
    "Екатеринбург": (56.8389, 60.6057),
    "Казань": (55.7903, 49.1115),
    "Нижний Новгород": (56.3269, 44.0059),
    "Челябинск": (55.1644, 61.4368),
    "Самара": (53.1959, 50.1008),
    "Омск": (54.9885, 73.3242),
    "Ростов-на-Дону": (47.2357, 39.7015),
    "Уфа": (54.7388, 55.9721),
    "Красноярск": (56.0153, 92.8932),
    "Пермь": (58.0105, 56.2502),
    "Воронеж": (51.6608, 39.2003),
    "Волгоград": (48.708, 44.5133),
    "Краснодар": (45.0355, 38.9753),
    "Тюмень": (57.1522, 65.5272),
    "Тольятти": (53.5206, 49.3895),
    "Ижевск": (56.8526, 53.2045),
    "Барнаул": (53.3478, 83.7784),
    "Владивосток": (43.1155, 131.8855),
    "Хабаровск": (48.4808, 135.0928),
    "Махачкала": (42.9849, 47.5047),
    "Ярославль": (57.6261, 39.8845),
    "Иркутск": (52.2978, 104.2964),
    "Калининград": (54.7104, 20.4522),
    "Сочи": (43.5855, 39.7231),
    "Севастополь": (44.6167, 33.5254),
    "Симферополь": (44.9482, 34.1003)
}

cities = list(cities_coords.keys())

# ==============================
# 📏 Расчет расстояния (формула Хаверсина)
# ==============================

def haversine(lat1, lon1, lat2, lon2):
    R = 6371  # радиус Земли в км
    phi1, phi2 = math.radians(lat1), math.radians(lat2)
    dphi = math.radians(lat2 - lat1)
    dlambda = math.radians(lon2 - lon1)
    a = math.sin(dphi / 2) ** 2 + math.cos(phi1) * math.cos(phi2) * math.sin(dlambda / 2) ** 2
    return R * (2 * math.atan2(math.sqrt(a), math.sqrt(1 - a)))

# ==============================
# 🚛 Типы транспорта
# ==============================

vehicles = {
    "Легкий фургон": {"max_weight": 1500, "multiplier": 1.0},
    "Грузовик 7.5т": {"max_weight": 7500, "multiplier": 1.25},
    "Грузовик 18т": {"max_weight": 18000, "multiplier": 1.5},
    "Тягач с полуприцепом": {"max_weight": 25000, "multiplier": 1.8}
}

# ==============================
# 💰 Тарифы (рубли)
# ==============================

BASE_RATE = 35.0    # ₽ за км
BASE_FEE = 1500.0   # ₽ фиксированная часть

# ==============================
# ⚙️ Расчёт стоимости
# ==============================

def calculate_price():
    origin = combo_from.get()
    destination = combo_to.get()
    weight = entry_weight.get()
    volume = entry_volume.get()
    vehicle = combo_vehicle.get()

    if not origin or not destination or not weight or not volume or not vehicle:
        messagebox.showwarning("Ошибка", "Пожалуйста, заполните все поля!")
        return

    if origin == destination:
        messagebox.showerror("Ошибка", "Города отправления и назначения совпадают!")
        return

    try:
        weight = float(weight)
        volume = float(volume)
    except ValueError:
        messagebox.showerror("Ошибка", "Введите числовые значения для веса и объема.")
        return

    if origin not in cities_coords or destination not in cities_coords:
        messagebox.showerror("Ошибка", "Один из выбранных городов не найден в базе координат.")
        return

    lat1, lon1 = cities_coords[origin]
    lat2, lon2 = cities_coords[destination]
    dist = round(haversine(lat1, lon1, lat2, lon2))

    vdata = vehicles[vehicle]
    if weight > vdata["max_weight"]:
        messagebox.showerror("Ошибка", f"Вес превышает лимит для {vehicle}!")
        return

    # Основной расчёт
    base_cost = BASE_FEE + BASE_RATE * dist
    weight_factor = 1 + (weight / vdata["max_weight"]) * 0.25
    total = base_cost * vdata["multiplier"] * weight_factor

    # Имитация рыночных колебаний (например, топливо)
    total *= random.uniform(1.03, 1.1)
    total = round(total, 2)

    # Форматированный вывод
    total_str = f"{total:,.2f}".replace(",", " ").replace(".", ",")
    messagebox.showinfo(
        "💰 Расчет стоимости",
        f"Маршрут: {origin} → {destination}\n"
        f"Расстояние: {dist} км\n"
        f"Транспорт: {vehicle}\n"
        f"Вес: {weight} кг\n"
        f"Объем: {volume} м³\n"
        f"\n💵 Итоговая стоимость: {total_str} ₽"
    )

# ==============================
# 🖥️ Интерфейс
# ==============================

root = tk.Tk()
root.title("🚛 Калькулятор автомобильных перевозок (₽)")
root.geometry("540x510")
root.config(bg="#eef5ff")

title = tk.Label(root, text="Расчет стоимости перевозки по России", font=("Arial", 16, "bold"), bg="#eef5ff")
title.pack(pady=15)

frame = tk.Frame(root, bg="#eef5ff")
frame.pack(pady=10)

tk.Label(frame, text="Откуда:", font=("Arial", 12), bg="#eef5ff").grid(row=0, column=0, sticky="e", padx=5, pady=5)
combo_from = ttk.Combobox(frame, values=cities, width=28, state="readonly")
combo_from.grid(row=0, column=1, pady=5)

tk.Label(frame, text="Куда:", font=("Arial", 12), bg="#eef5ff").grid(row=1, column=0, sticky="e", padx=5, pady=5)
combo_to = ttk.Combobox(frame, values=cities, width=28, state="readonly")
combo_to.grid(row=1, column=1, pady=5)

tk.Label(frame, text="Вес (кг):", font=("Arial", 12), bg="#eef5ff").grid(row=2, column=0, sticky="e", padx=5, pady=5)
entry_weight = tk.Entry(frame, width=30)
entry_weight.grid(row=2, column=1, pady=5)

tk.Label(frame, text="Объем (м³):", font=("Arial", 12), bg="#eef5ff").grid(row=3, column=0, sticky="e", padx=5, pady=5)
entry_volume = tk.Entry(frame, width=30)
entry_volume.grid(row=3, column=1, pady=5)

tk.Label(frame, text="Тип транспорта:", font=("Arial", 12), bg="#eef5ff").grid(row=4, column=0, sticky="e", padx=5, pady=5)
combo_vehicle = ttk.Combobox(frame, values=list(vehicles.keys()), width=28, state="readonly")
combo_vehicle.grid(row=4, column=1, pady=5)

# Кнопки
btn_calc = tk.Button(root, text="Рассчитать стоимость", command=calculate_price,
                     bg="#4CAF50", fg="white", font=("Arial", 13, "bold"), width=25)
btn_calc.pack(pady=25)

btn_exit = tk.Button(root, text="Выход", command=root.quit, bg="#d32f2f", fg="white",
                     font=("Arial", 11), width=15)
btn_exit.pack(pady=10)

footer = tk.Label(root, text="© 2025 Автоперевозки РФ | Расчет в рублях (₽)",
                  font=("Arial", 9), bg="#eef5ff", fg="gray")
footer.pack(side="bottom", pady=5)

root.mainloop()
