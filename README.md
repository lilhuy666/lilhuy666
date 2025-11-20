import tkinter as tk
from tkinter import ttk, messagebox
import math
import random
import qrcode
from PIL import Image, ImageTk
import io
from datetime import datetime

# ===============================
# 🌍 Координаты городов
# ===============================
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

# ===============================
# 📏 Формула Хаверсина
# ===============================
def haversine(lat1, lon1, lat2, lon2):
    R = 6371
    phi1, phi2 = math.radians(lat1), math.radians(lat2)
    dphi = math.radians(lat2 - lat1)
    dlambda = math.radians(lon2 - lon1)
    a = math.sin(dphi / 2)**2 + math.cos(phi1)*math.cos(phi2)*math.sin(dlambda/2)**2
    return R * (2 * math.atan2(math.sqrt(a), math.sqrt(1 - a)))

# ===============================
# 🚛 Транспорт
# ===============================
vehicles = {
    "Легкий фургон": {"max_weight": 1200, "rate": 28},
    "Грузовик 5т": {"max_weight": 5000, "rate": 34},
    "Грузовик 10т": {"max_weight": 10000, "rate": 40},
    "Грузовик 20т": {"max_weight": 20000, "rate": 55},  # Добавлен до 20 тонн
}
BASE_FEE = 1200

# ===============================
# 💾 История расчетов и клиенты
# ===============================
history = []
clients = []

# ===============================
# 🖥️ Главное окно
# ===============================
root = tk.Tk()
root.title("🚚 LogiCalc Pro - Калькулятор логистики")
root.geometry("1200x800")
root.configure(bg="#0f172a")
root.minsize(1100, 700)

# Стиль для виджетов
style = ttk.Style()
style.theme_use('clam')

# Современные цвета
COLORS = {
    "primary": "#3b82f6",
    "secondary": "#1e40af",
    "accent": "#06b6d4",
    "background": "#0f172a",
    "surface": "#1e293b",
    "text": "#f8fafc",
    "text_secondary": "#94a3b8",
    "success": "#10b981",
    "warning": "#f59e0b",
    "error": "#ef4444"
}

# ===============================
# Header
# ===============================
header = tk.Frame(root, bg=COLORS["surface"], height=100)
header.pack(fill="x", pady=(0, 10))
header.pack_propagate(False)

header_content = tk.Frame(header, bg=COLORS["surface"])
header_content.pack(expand=True, fill='both', padx=30)

tk.Label(header_content, text="🚚", bg=COLORS["surface"], fg=COLORS["accent"],
         font=("Arial", 32)).pack(side='left', pady=20)
tk.Label(header_content, text="LogiCalc Pro", bg=COLORS["surface"], fg=COLORS["text"],
         font=("Arial", 24, "bold")).pack(side='left', padx=10, pady=20)
tk.Label(header_content, text="Умный калькулятор логистических перевозок", 
         bg=COLORS["surface"], fg=COLORS["text_secondary"],
         font=("Arial", 12)).pack(side='left', padx=10, pady=20)

# ===============================
# 🌟 Основная структура с меню слева
# ===============================
container = tk.Frame(root, bg=COLORS["background"])
container.pack(expand=True, fill="both", padx=20, pady=10)

# Menu frame (слева)
menu_frame = tk.Frame(container, bg=COLORS["surface"], width=250, relief='flat')
menu_frame.pack(side="left", fill="y", padx=(0, 15))
menu_frame.pack_propagate(False)

# Main frame
main_frame = tk.Frame(container, bg=COLORS["background"], relief='flat')
main_frame.pack(side="left", expand=True, fill="both")

# ===============================
# Функции для смены страниц
# ===============================
def clear_main_frame():
    for widget in main_frame.winfo_children():
        widget.destroy()

def show_calc_frame():
    clear_main_frame()
    build_calc_frame(main_frame)

def show_clients_frame():
    clear_main_frame()
    build_clients_frame(main_frame)

def show_history_frame():
    clear_main_frame()
    build_history_frame(main_frame)

# ===============================
# 🌟 Меню слева
# ===============================
tk.Label(menu_frame, text="Навигация", bg=COLORS["surface"], fg=COLORS["accent"], 
         font=("Arial", 16, "bold")).pack(pady=(25, 20))

buttons_info = [
    ("🧮 Расчет перевозки", show_calc_frame),
    ("👥 База клиентов", show_clients_frame),
    ("📊 История заказов", show_history_frame)
]

for text, cmd in buttons_info:
    btn = tk.Button(menu_frame, text=text, command=cmd,
                    bg=COLORS["surface"], fg=COLORS["text"], 
                    font=("Arial", 12), 
                    width=20, height=2, relief='flat',
                    cursor='hand2', 
                    activebackground=COLORS["secondary"],
                    activeforeground=COLORS["text"])
    btn.pack(pady=8, padx=15)

# Разделитель
separator = tk.Frame(menu_frame, height=1, bg=COLORS["secondary"])
separator.pack(fill='x', pady=20, padx=20)

# Статистика
stats_frame = tk.Frame(menu_frame, bg=COLORS["surface"])
stats_frame.pack(pady=10, padx=15, fill='x')

def update_stats():
    for widget in stats_frame.winfo_children():
        widget.destroy()
    
    total_clients = len(clients)
    total_orders = len(history)
    total_revenue = sum(h['price'] for h in history if h['paid'])
    
    tk.Label(stats_frame, text="📈 Статистика", bg=COLORS["surface"], fg=COLORS["accent"],
             font=("Arial", 12, "bold")).pack(anchor='w', pady=(0, 8))
    
    stats_data = [
        ("👥 Клиенты:", f"{total_clients}"),
        ("📦 Заказы:", f"{total_orders}"),
        ("💰 Выручка:", f"{total_revenue:,.0f} ₽")
    ]
    
    for label, value in stats_data:
        stat_frame = tk.Frame(stats_frame, bg=COLORS["surface"])
        stat_frame.pack(fill='x', pady=3)
        tk.Label(stat_frame, text=label, bg=COLORS["surface"], fg=COLORS["text_secondary"],
                 font=("Arial", 10)).pack(side='left')
        tk.Label(stat_frame, text=value, bg=COLORS["surface"], fg=COLORS["text"],
                 font=("Arial", 10, "bold")).pack(side='right')

# ===============================
# 🧮 Расчет перевозки
# ===============================
def build_calc_frame(parent):
    parent.configure(bg=COLORS["background"])
    
    main_container = tk.Frame(parent, bg=COLORS["background"])
    main_container.pack(expand=True, fill='both', padx=30, pady=25)
    
    tk.Label(main_container, text="🧮 Расчет логистической перевозки", 
             bg=COLORS["background"], fg=COLORS["text"], 
             font=("Arial", 24, "bold")).pack(pady=(0, 25))
    
    # Форма ввода данных
    form_container = tk.Frame(main_container, bg=COLORS["surface"], 
                             relief='flat', padx=30, pady=30)
    form_container.pack(fill='x', pady=15)
    
    # Сетка для полей ввода
    input_grid = tk.Frame(form_container, bg=COLORS["surface"])
    input_grid.pack(expand=True, fill='both')
    
    labels = [
        "📍 Город отправления:", 
        "🎯 Город назначения:", 
        "⚖️ Вес груза (кг):", 
        "🚛 Тип транспорта:", 
        "👤 Имя клиента:",
        "💳 Способ оплаты:"
    ]
    
    # Создаем переменные и виджеты
    combo_from = ttk.Combobox(input_grid, values=cities, width=28, 
                             state="readonly", font=("Arial", 11))
    combo_to = ttk.Combobox(input_grid, values=cities, width=28, 
                           state="readonly", font=("Arial", 11))
    
    weight_frame = tk.Frame(input_grid, bg=COLORS["surface"])
    scale_weight = tk.Scale(weight_frame, from_=1, to=20000, orient="horizontal", 
                           length=300, font=("Arial", 10), showvalue=True,
                           bg=COLORS["surface"], fg=COLORS["text"], 
                           troughcolor=COLORS["secondary"],
                           highlightbackground=COLORS["surface"])
    scale_weight.set(1000)
    
    combo_vehicle = ttk.Combobox(input_grid, values=list(vehicles.keys()), 
                                width=28, state="readonly", font=("Arial", 11))
    entry_client = tk.Entry(input_grid, width=31, font=("Arial", 11), 
                           bg=COLORS["secondary"], fg=COLORS["text"], 
                           insertbackground=COLORS["text"],
                           relief='flat')
    combo_payment = ttk.Combobox(input_grid, 
                                values=["QR-код", "Банковская карта"], 
                                width=28, state="readonly", font=("Arial", 11))
    
    widgets = [combo_from, combo_to, weight_frame, combo_vehicle, entry_client, combo_payment]
    
    # Размещаем в сетке
    for i, (lbl_text, widget) in enumerate(zip(labels, widgets)):
        row = i
        tk.Label(input_grid, text=lbl_text, font=("Arial", 12, "bold"), 
                bg=COLORS["surface"], fg=COLORS["text"], justify='left').grid(
                row=row, column=0, sticky='w', pady=15, padx=(0, 20))
        
        if lbl_text == "⚖️ Вес груза (кг):":
            scale_weight.pack(fill='x', expand=True)
            widget.grid(row=row, column=1, sticky='ew', pady=15)
        else:
            widget.grid(row=row, column=1, sticky='ew', pady=15)
    
    input_grid.columnconfigure(1, weight=1)
    
    # Кнопка расчета
    button_frame = tk.Frame(main_container, bg=COLORS["background"])
    button_frame.pack(pady=25)
    
    def calculate_price():
        origin = combo_from.get()
        destination = combo_to.get()
        weight = scale_weight.get()
        vehicle = combo_vehicle.get()
        client_name = entry_client.get()
        payment_method = combo_payment.get()

        if not all([origin, destination, vehicle, client_name, payment_method]):
            messagebox.showwarning("Ошибка ввода", "Пожалуйста, заполните все поля формы!")
            return
        if origin == destination:
            messagebox.showerror("Ошибка маршрута", "Города отправления и назначения не могут совпадать!")
            return
            
        v = vehicles[vehicle]
        if weight > v["max_weight"]:
            messagebox.showerror("Ошибка веса", 
                               f"Вес {weight} кг превышает максимальный для {vehicle} ({v['max_weight']} кг)!")
            return

        # Расчет стоимости
        lat1, lon1 = cities_coords[origin]
        lat2, lon2 = cities_coords[destination]
        dist = round(haversine(lat1, lon1, lat2, lon2))
        total = (dist * v["rate"] + BASE_FEE) * random.uniform(1.02, 1.06)
        total = round(total, 2)

        record = {
            "client": client_name,
            "route": f"{origin} → {destination}",
            "distance": dist,
            "vehicle": vehicle,
            "weight": weight,
            "price": total,
            "paid": False,
            "payment_method": payment_method,
            "order_id": f"LC{datetime.now().strftime('%Y%m%d')}{len(history)+1:03d}",
            "created_date": datetime.now().strftime("%d.%m.%Y %H:%M")
        }
        history.append(record)
        if client_name not in clients:
            clients.append(client_name)
        
        update_stats()
        show_payment_window(record, total)

    # ===============================
    # Оплата
    # ===============================
    def show_payment_window(record, total):
        payment_window = tk.Toplevel()
        payment_window.title("💳 Оплата логистической услуги")
        payment_window.geometry("700x800")
        payment_window.configure(bg=COLORS["background"])
        payment_window.resizable(False, False)
        payment_window.transient(root)
        payment_window.grab_set()

        tk.Label(payment_window, text=f"Оплата заказа: {record['order_id']}", 
                 bg=COLORS["background"], fg=COLORS["accent"], font=("Arial", 16, "bold")).pack(pady=20)

        if record["payment_method"] == "QR-код":
            # QR-код
            qr_frame = tk.Frame(payment_window, bg=COLORS["background"])
            qr_frame.pack(pady=10)
            
            qr_data = f"""
Заказ: {record['order_id']}
Клиент: {record['client']}
Маршрут: {record['route']}
Сумма: {total:,.2f} ₽
"""
            qr = qrcode.QRCode(version=1, box_size=10, border=4)
            qr.add_data(qr_data)
            qr.make(fit=True)
            qr_image = qr.make_image(fill_color=COLORS["accent"], back_color=COLORS["background"])
            qr_image = qr_image.resize((300, 300))
            img_bytes = io.BytesIO()
            qr_image.save(img_bytes, format='PNG')
            img_bytes.seek(0)
            photo_image = ImageTk.PhotoImage(Image.open(img_bytes))
            tk.Label(qr_frame, image=photo_image, bg=COLORS["background"]).pack()
            qr_frame.image = photo_image

            def mark_as_paid():
                record["paid"] = True
                messagebox.showinfo("Оплата", f"✅ Заказ {record['order_id']} оплачен!")
                payment_window.destroy()
            tk.Button(payment_window, text="✅ Оплачено", command=mark_as_paid, 
                      bg=COLORS["success"], fg=COLORS["text"], font=("Arial", 14), padx=20, pady=10).pack(pady=20)

        else:
            # Банковская карта
            card_frame = tk.Frame(payment_window, bg=COLORS["surface"], padx=20, pady=20)
            card_frame.pack(fill='x', pady=10)
            
            fields = [
                ("Номер карты (16 цифр):", 16),
                ("Срок действия (ММ/ГГ):", 5),
                ("CVC код (3 цифры):", 3),
                ("Имя владельца карты:", None)
            ]
            entries = {}
            for label_text, length in fields:
                frame = tk.Frame(card_frame, bg=COLORS["surface"])
                frame.pack(fill='x', pady=10)
                tk.Label(frame, text=label_text, bg=COLORS["surface"], fg=COLORS["text"], font=("Arial", 11)).pack(anchor='w')
                entry = tk.Entry(frame, width=35, bg=COLORS["secondary"], fg=COLORS["text"], insertbackground=COLORS["text"], relief='flat')
                entry.pack(fill='x', pady=(5,0), ipady=5)
                entries[label_text] = entry
            tk.Label(card_frame, text=f"Сумма к оплате: {total:,.2f} ₽", bg=COLORS["surface"], fg=COLORS["accent"], font=("Arial", 12, "bold")).pack(anchor='w', pady=10)

            def pay_card():
                card_number = entries["Номер карты (16 цифр):"].get().replace(" ", "")
                expiry = entries["Срок действия (ММ/ГГ):"].get()
                cvc = entries["CVC код (3 цифры):"].get()
                owner = entries["Имя владельца карты:"].get()
                # Валидация
                if len(card_number) != 16 or not card_number.isdigit():
                    messagebox.showerror("Ошибка", "Номер карты должен содержать 16 цифр!")
                    return
                if len(expiry) != 5 or expiry[2] != "/" or not expiry.replace("/", "").isdigit():
                    messagebox.showerror("Ошибка", "Срок действия должен быть ММ/ГГ!")
                    return
                if len(cvc) != 3 or not cvc.isdigit():
                    messagebox.showerror("Ошибка", "CVC код должен содержать 3 цифры!")
                    return
                if len(owner.strip()) == 0:
                    messagebox.showerror("Ошибка", "Введите имя владельца карты!")
                    return
                # Оплата успешна
                record["paid"] = True
                messagebox.showinfo("Оплата", f"Оплата {total:,.2f} ₽ успешно проведена!")
                payment_window.destroy()
            tk.Button(card_frame, text="✅ Оплатить картой", command=pay_card, 
                      bg=COLORS["primary"], fg=COLORS["text"], font=("Arial", 12, "bold"), width=20, height=2).pack(pady=20)
    
    tk.Button(button_frame, text="Рассчитать стоимость", command=calculate_price, 
              bg=COLORS["accent"], fg=COLORS["text"], font=("Arial", 14, "bold"), width=25, height=2).pack(pady=10)

# ===============================
# 👥 База клиентов
# ===============================
def build_clients_frame(parent):
    parent.configure(bg=COLORS["background"])
    tk.Label(parent, text="👥 База клиентов", bg=COLORS["background"], fg=COLORS["text"], font=("Arial", 24, "bold")).pack(pady=20)
    
    list_frame = tk.Frame(parent, bg=COLORS["surface"], padx=20, pady=20)
    list_frame.pack(fill='both', expand=True, padx=30, pady=15)
    
    for i, client in enumerate(clients, 1):
        tk.Label(list_frame, text=f"{i}. {client}", bg=COLORS["surface"], fg=COLORS["text"], font=("Arial", 12)).pack(anchor='w', pady=3)

# ===============================
# 📊 История заказов
# ===============================
def build_history_frame(parent):
    parent.configure(bg=COLORS["background"])
    tk.Label(parent, text="📊 История заказов", bg=COLORS["background"], fg=COLORS["text"], font=("Arial", 24, "bold")).pack(pady=20)
    
    list_frame = tk.Frame(parent, bg=COLORS["surface"], padx=20, pady=20)
    list_frame.pack(fill='both', expand=True, padx=30, pady=15)
    
    for h in history:
        status = "✅ Оплачен" if h["paid"] else "❌ Не оплачен"
        tk.Label(list_frame, text=f"{h['order_id']} | {h['client']} | {h['route']} | {h['vehicle']} | {h['weight']} кг | {h['price']:,.2f} ₽ | {status}", 
                 bg=COLORS["surface"], fg=COLORS["text"], font=("Arial", 12), anchor='w').pack(fill='x', pady=2)

# ===============================
# Старт
# ===============================
show_calc_frame()
update_stats()
root.mainloop()
