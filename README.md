import tkinter as tk
from tkinter import ttk, messagebox
from datetime import datetime
import json, os
import hashlib

# ===================== STYLE =====================
BG = "#0b1220"
PANEL = "#111a2e"
CARD = "#162238"
ACCENT = "#4f8cff"
ACCENT2 = "#22c55e"
TEXT = "#e5e7eb"
SUB = "#94a3b8"
DANGER = "#ef4444"
PROFILE_BG = "#f5f7fa"

DATA_FILE = "data.json"

# ===================== DATA =====================
data = {"users": {}}
current_user = None

def hash_password(password):
    """Хеширование пароля"""
    return hashlib.sha256(password.encode()).hexdigest()

def load_data():
    global data
    if os.path.exists(DATA_FILE):
        try:
            with open(DATA_FILE, "r", encoding="utf-8") as f:
                data = json.load(f)
            if "users" not in data:
                data = {"users": {}}
        except Exception as e:
            messagebox.showerror("Ошибка", f"Не удалось загрузить данные: {e}")
            data = {"users": {}}

def save_data():
    try:
        with open(DATA_FILE, "w", encoding="utf-8") as f:
            json.dump(data, f, indent=2, ensure_ascii=False)
    except Exception as e:
        messagebox.showerror("Ошибка", f"Не удалось сохранить данные: {e}")

# ===================== WINDOW =====================
root = tk.Tk()
root.title("CalculatCar")
root.geometry("1200x750")
root.configure(bg=BG)

main = tk.Frame(root, bg=BG)
main.pack(fill="both", expand=True)

# ===================== HEADER =====================
# ... (остаётся без изменений)

# ===================== CONTENT =====================
# ... (остаётся без изменений)

# ===================== PROFILE =====================
# ... (остаётся без изменений)

# ===================== CALCULATOR (ИЗМЕНЁН) =====================
def show_calc():
    clear()

    tk.Label(content, text="Калькулятор расхода топлива",
             bg=BG, fg=TEXT,
             font=("Arial", 20, "bold")).pack(pady=10)

    # Режим расчёта (комбобокс вместо радиокнопок)
    mode_label = tk.Label(content, text="Режим расчёта:", bg=BG, fg=TEXT)
    mode_label.pack(anchor="w", pady=(10, 5))
    mode = ttk.Combobox(content, values=["Стоимость поездки", "Средний расход на 100 км"], state="readonly")
    mode.set("Стоимость поездки")
    mode.pack(fill="x", padx=20, pady=5)

    form = tk.Frame(content, bg=CARD, padx=40, pady=40)
    form.pack(pady=20)

    entries = {}

    def build_form():
        for w in form.winfo_children():
            w.destroy()
        entries.clear()

        def add_field(label_text, entry_var):
            tk.Label(form, text=label_text, bg=CARD, fg=TEXT).pack(anchor="w", pady=5)
            entry = tk.Entry(form, textvariable=entry_var, font=("Arial", 12))
            entry.pack(fill="x", padx=10, pady=5)
            entries[label_text] = entry

        if mode.get() == "Стоимость поездки":
            entries["fuel"] = tk.DoubleVar()
            add_field("Топливо (л)", entries["fuel"])
            entries["distance"] = tk.DoubleVar()
            add_field("Расстояние (км)", entries["distance"])
            entries["price"] = tk.DoubleVar()
            add_field("Цена за литр (₽)", entries["price"])
        else:
            entries["consumption"] = tk.DoubleVar()
            add_field("Средний расход (л/100км)", entries["consumption"])
            entries["distance"] = tk.DoubleVar()
            add_field("Расстояние (км)", entries["distance"])
            entries["price"] = tk.DoubleVar()
            add_field("Цена за литр (₽)", entries["price"])

    build_form()

    def calc():
        try:
            if mode.get() == "Стоимость поездки":
                fuel = entries["fuel"].get()
                distance = entries["distance"].get()
                price = entries["price"].get()

                if fuel <= 0 or distance <= 0 or price <= 0:
                    messagebox.showerror("Ошибка", "Все значения должны быть положительными")
                    return

                consumption = (fuel / distance) * 100
                total_cost = fuel * price

            else:  # Средний расход
                consumption = entries["consumption"].get()
                distance = entries["distance"].get()
                price = entries["price"].get()

                if consumption <= 0 or distance <= 0 or price <= 0:
                    messagebox.showerror("Ошибка", "Все значения должны быть положительными")
                    return

                total_cost = (consumption * distance / 100) * price

            # Вывод результатов
            result_text = f"Расход: {consumption:.1f} л/100км\n"
            result_text += f"Стоимость: {total_cost:.2f} ₽"
            result.config(text=result_text)

            # Сохранение в историю
            if current_user:
                data["users"][current_user]["history"].append({
                    "date": datetime.now().strftime("%d.%m.%Y %H:%M"),
                    "mode": mode.get(),
                    "fuel": entries.get("fuel", None).get() if "fuel" in entries else None,
                    "distance": distance,
                    "price_per_liter": price,
                    "consumption": consumption,
                    "total_cost": total_cost
                })
                data["users"][current_user]["history"] = data["users"][current_user]["history"][-50:]
                save_data()

        except ValueError:
            messagebox.showerror("Ошибка", "Проверьте ввод — все поля должны содержать числа")
        except Exception as e:
            messagebox.showerror("Ошибка", f"Произошла ошибка: {e}")

    result = tk.Label(content, text="", bg=BG, fg=ACCENT, font=("Arial", 16, "bold"), justify="left")
    result.pack(pady=20)

    # Кнопка расчёта с иконкой
    calc_btn = tk.Button(content, text="🔢 Рассчитать", bg=ACCENT, fg="white",
                         font=("Arial", 14, "bold"), command=calc)
    calc_btn.pack(pady=10)

    # Кнопка сброса формы
    reset_btn = tk.Button(content, text="🔄 Сбросить", bg="#63b3ed", fg="white",
                          font=("Arial", 12, "bold"), command=build_form)
    reset_btn.pack(pady=5)

    # Привязка изменения режима к перестройке формы
    mode.bind("<<ComboboxSelected>>", lambda event: build_form())

# ===================== HISTORY, SETTINGS, ABOUT =====================
# (остаются без изменений)

# ===================== UPDATE USER =====================
# (остаётся без изменений)

# ===================== START =====================
load_data()
update_user()
show_calc()  # По умолчанию показываем калькулятор

root.mainloop()
