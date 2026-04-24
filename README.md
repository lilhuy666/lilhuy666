import tkinter as tk
from tkinter import messagebox
from datetime import datetime
import json, os

# ===================== STYLE =====================
BG = "#0b1220"
PANEL = "#111a2e"
CARD = "#162238"
ACCENT = "#4f8cff"
ACCENT2 = "#22c55e"
TEXT = "#e5e7eb"
SUB = "#94a3b8"
DANGER = "#ef4444"

DATA_FILE = "data.json"

# ===================== DATA =====================
data = {"users": {}}
current_user = None

def load_data():
    global data
    if os.path.exists(DATA_FILE):
        try:
            with open(DATA_FILE, "r", encoding="utf-8") as f:
                data = json.load(f)
            if "users" not in data:
                data = {"users": {}}
        except:
            data = {"users": {}}

def save_data():
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=2, ensure_ascii=False)

# ===================== WINDOW =====================
root = tk.Tk()
root.title("CalculatCar")
root.geometry("1200x750")
root.configure(bg=BG)

main = tk.Frame(root, bg=BG)
main.pack(fill="both", expand=True)

# ===================== HEADER =====================
header = tk.Frame(main, bg=ACCENT, height=70)
header.pack(fill="x")
header.pack_propagate(False)

menu_window = None

def toggle_menu():
    global menu_window

    if menu_window and menu_window.winfo_exists():
        menu_window.destroy()
        return

    x = root.winfo_x() + 20
    y = root.winfo_y() + 70

    menu_window = tk.Toplevel(root)
    menu_window.overrideredirect(True)
    menu_window.configure(bg=PANEL)
    menu_window.geometry(f"220x240+{x}+{y}")

    def close(e=None):
        if menu_window:
            menu_window.destroy()

    menu_window.bind("<FocusOut>", close)

    def nav(text, cmd):
        tk.Button(menu_window,
                  text=text,
                  bg=PANEL,
                  fg=TEXT,
                  bd=0,
                  anchor="w",
                  padx=15,
                  pady=10,
                  font=("Arial", 11),
                  activebackground=CARD,
                  activeforeground=ACCENT,
                  command=lambda: [cmd(), close()]
                  ).pack(fill="x")

    nav("Калькулятор", show_calc)
    nav("Профиль", show_profile)
    nav("История", show_history)
    nav("Настройки", show_settings)
    nav("О программе", show_about)

    menu_window.focus_force()

tk.Button(header, text="☰", bg=ACCENT, fg="white",
          font=("Arial", 16, "bold"), bd=0,
          command=toggle_menu).pack(side="left", padx=15)

tk.Label(header, text="CalculatCar",
         bg=ACCENT, fg="white",
         font=("Arial", 20, "bold")).pack(side="left")

user_label = tk.Label(header, text="",
                      bg=ACCENT, fg="white",
                      font=("Arial", 11))
user_label.pack(side="right", padx=15)

# ===================== CONTENT =====================
content = tk.Frame(main, bg=BG)
content.pack(fill="both", expand=True)

def clear():
    for w in content.winfo_children():
        w.destroy()

def card():
    f = tk.Frame(content, bg=CARD, padx=40, pady=40)
    f.pack(pady=30)
    return f

# ===================== PROFILE =====================
def show_profile():
    clear()

    if not current_user:
        c = card()

        tk.Label(c, text="Вход / Регистрация",
                 bg=CARD, fg=TEXT,
                 font=("Arial", 20, "bold")).pack(pady=10)

        email = tk.Entry(c, font=("Arial", 14))
        password = tk.Entry(c, show="*", font=("Arial", 14))

        email.pack(fill="x", pady=8)
        password.pack(fill="x", pady=8)

        def login():
            global current_user
            e = email.get().strip()
            p = password.get().strip()

            if e in data["users"] and data["users"][e]["password"] == p:
                current_user = e
                update_user()
                show_profile()
            else:
                messagebox.showerror("Ошибка", "Неверные данные")

        def register():
            e = email.get().strip()
            p = password.get().strip()

            if not e or not p:
                return messagebox.showerror("Ошибка", "Заполни поля")

            if e in data["users"]:
                return messagebox.showerror("Ошибка", "Уже существует")

            data["users"][e] = {
                "password": p,
                "name": "",
                "car": "",
                "history": []
            }
            save_data()
            messagebox.showinfo("OK", "Аккаунт создан")

        tk.Button(c, text="Войти", bg=ACCENT, fg="white",
                  command=login).pack(fill="x", pady=10)

        tk.Button(c, text="Регистрация", bg=ACCENT2, fg="black",
                  command=register).pack(fill="x")
        return

    user = data["users"][current_user]
    c = card()

    tk.Label(c, text="Профиль",
             bg=CARD, fg=TEXT,
             font=("Arial", 20, "bold")).pack(pady=10)

    # ==== ИМЯ ====
    row1 = tk.Frame(c, bg=CARD)
    row1.pack(fill="x", pady=5)

    tk.Label(row1, text="Имя", bg=CARD, fg=SUB).pack(anchor="w")

    name_entry = tk.Entry(row1, font=("Arial", 14))
    name_entry.insert(0, user.get("name", ""))
    name_entry.pack(side="left", fill="x", expand=True, padx=(0,10))

    def save_name():
        user["name"] = name_entry.get()
        save_data()
        messagebox.showinfo("OK", "Имя сохранено")

    tk.Button(row1, text="Сохранить",
              bg=ACCENT, fg="white",
              command=save_name).pack(side="right")

    # ==== АВТО ====
    row2 = tk.Frame(c, bg=CARD)
    row2.pack(fill="x", pady=5)

    tk.Label(row2, text="Автомобиль", bg=CARD, fg=SUB).pack(anchor="w")

    car_entry = tk.Entry(row2, font=("Arial", 14))
    car_entry.insert(0, user.get("car", ""))
    car_entry.pack(side="left", fill="x", expand=True, padx=(0,10))

    def save_car():
        user["car"] = car_entry.get()
        save_data()
        messagebox.showinfo("OK", "Авто сохранено")

    tk.Button(row2, text="Сохранить",
                            bg=ACCENT, fg="white",
              command=save_car).pack(side="right")

    # ==== ПАРОЛЬ ====
    row3 = tk.Frame(c, bg=CARD)
    row3.pack(fill="x", pady=5)

    tk.Label(row3, text="Новый пароль", bg=CARD, fg=SUB).pack(anchor="w")

    pass1 = tk.Entry(row3, show="*", font=("Arial", 14))
    pass1.pack(fill="x", pady=5)

    row4 = tk.Frame(c, bg=CARD)
    row4.pack(fill="x", pady=5)

    tk.Label(row4, text="Повтор пароля", bg=CARD, fg=SUB).pack(anchor="w")

    pass2 = tk.Entry(row4, show="*", font=("Arial", 14))
    pass2.pack(side="left", fill="x", expand=True, padx=(0,10))

    def save_password():
        p1 = pass1.get()
        p2 = pass2.get()

        if not p1 or not p2:
            return messagebox.showerror("Ошибка", "Введите пароль")

        if p1 != p2:
            return messagebox.showerror("Ошибка", "Пароли не совпадают")

        user["password"] = p1
        save_data()
        messagebox.showinfo("OK", "Пароль изменён")

    tk.Button(row4, text="Сохранить",
              bg=ACCENT, fg="white",
              command=save_password).pack(side="right")

    def logout():
        global current_user
        current_user = None
        update_user()
        show_profile()

    tk.Button(c, text="Выйти",
              bg=DANGER, fg="white",
              command=logout).pack(fill="x", pady=10)

# ===================== CALCULATOR =====================
def show_calc():
    clear()

    tk.Label(content, text="Калькулятор расхода",
             bg=BG, fg=TEXT,
             font=("Arial", 20, "bold")).pack(pady=10)

    mode = tk.StringVar(value="1")

    switch = tk.Frame(content, bg=BG)
    switch.pack(pady=10)

    tk.Radiobutton(switch, text="Рассчитать стоимость поездки",
                   variable=mode, value="1",
                   indicatoron=0, width=28,
                   bg=PANEL, fg=TEXT,
                   selectcolor=ACCENT).pack(side="left", padx=5)

    tk.Radiobutton(switch, text="Средний расход на 100 км",
                   variable=mode, value="2",
                   indicatoron=0, width=28,
                   bg=PANEL, fg=TEXT,
                   selectcolor=ACCENT).pack(side="left", padx=5)

    form = tk.Frame(content, bg=CARD, padx=40, pady=40)
    form.pack(pady=20)

    entries = {}

    def build():
        for w in form.winfo_children():
            w.destroy()
        entries.clear()

        def add(t):
            tk.Label(form, text=t, bg=CARD, fg=TEXT).pack(anchor="w")
            e = tk.Entry(form)
            e.pack(fill="x", pady=5)
            entries[t] = e

        if mode.get() == "1":
            add("Топливо (л)")
            add("Расстояние (км)")
            add("Цена за литр")
        else:
            add("Средний расход (л/100км)")
            add("Расстояние (км)")
            add("Цена за литр")

    def calc():
        try:
            if mode.get() == "1":
                f = float(entries["Топливо (л)"].get())
                d = float(entries["Расстояние (км)"].get())
                p = float(entries["Цена за литр"].get())
                cons = (f / d) * 100
                cost = f * p
            else:
                cons = float(entries["Средний расход (л/100км)"].get())
                d = float(entries["Расстояние (км)"].get())
                p = float(entries["Цена за литр"].get())
                cost = cons * d / 100 * p

            result.config(text=f"{cons:.1f} л/100км | {cost:.2f} ₽")

            if current_user:
                data["users"][current_user]["history"].append({
                    "date": datetime.now().strftime("%d.%m.%Y %H:%M"),
                    "distance": d,
                    "fuel": f if mode.get() == "1" else None,
                    "price_per_liter": p,
                    "consumption": cons,
            "total_cost": cost
                })
                data["users"][current_user]["history"] = data["users"][current_user]["history"][-50:]
                save_data()

        except ValueError:
            messagebox.showerror("Ошибка", "Проверь ввод — все поля должны содержать числа")
        except Exception as e:
            messagebox.showerror("Ошибка", f"Произошла ошибка: {e}")

    result = tk.Label(content, text="—",
                      bg=BG, fg=ACCENT,
                      font=("Arial", 22, "bold"))
    result.pack()

    tk.Button(content, text="Рассчитать",
              bg=ACCENT, fg="white",
              font=("Arial", 14, "bold"),
              command=calc).pack(pady=10)

    mode.trace("w", lambda *a: build())
    build()

# ===================== HISTORY =====================
def show_history():
    clear()

    if not current_user:
        tk.Label(content, text="Авторизуйтесь для просмотра истории", bg=BG, fg=TEXT).pack(pady=20)
        return

    # Заголовок
    tk.Label(content, text="История расчётов", bg=BG, fg=TEXT, font=("Arial", 20, "bold")).pack(pady=10)
    tk.Label(content, text="Ваши последние расчёты", bg=BG, fg=SUB).pack(pady=5)

    # Кнопки «Фильтры» и «Очистить историю»
    btn_frame = tk.Frame(content, bg=BG)
    btn_frame.pack(fill="x", pady=10)

    tk.Button(btn_frame, text="Фильтры", bg=ACCENT, fg="white", padx=10).pack(side="right", padx=5)
    tk.Button(btn_frame, text="Очистить историю", bg=DANGER, fg="white", padx=10,
              command=lambda: clear_history()).pack(side="right", padx=5)

    # Таблица истории
    history_frame = tk.Frame(content, bg=CARD)
    history_frame.pack(fill="both", expand=True, padx=20, pady=10)

    # Шапка таблицы
    header = tk.Frame(history_frame, bg=CARD, height=30)
    header.pack(fill="x")

    tk.Label(header, text="Дата и время", bg=CARD, fg=TEXT, width=15).pack(side="left")
    tk.Label(header, text="Параметры расчёта", bg=CARD, fg=TEXT, width=20).pack(side="left")
    tk.Label(header, text="Результат", bg=CARD, fg=TEXT, width=15).pack(side="left")
    tk.Label(header, text="Действия", bg=CARD, fg=TEXT, width=10).pack(side="left")

    # Строки истории
    for item in reversed(data["users"][current_user]["history"]):
        row = tk.Frame(history_frame, bg=CARD, height=50)
        row.pack(fill="x", pady=2)
                # Дата и время
        tk.Label(row, text=f"{item['date']}", bg=CARD, fg=TEXT, width=15).pack(side="left", padx=5)

        # Параметры расчёта
        params = tk.Frame(row, bg=CARD)
        params.pack(side="left", padx=10)

        if item['fuel'] is not None:
            tk.Label(params, text=f"Топливо: {item['fuel']} л", bg=CARD, fg=TEXT).pack(anchor="w")
        tk.Label(params, text=f"Расстояние: {item['distance']} км", bg=CARD, fg=TEXT).pack(anchor="w")
        tk.Label(params, text=f"Цена: {item['price_per_liter']} ₽/л", bg=CARD, fg=TEXT).pack(anchor="w")

        # Результат
        result_frame = tk.Frame(row, bg=ACCENT, padx=8, pady=4)
        result_frame.pack(side="left", padx=10)
        tk.Label(result_frame, text=f"{item['consumption']:.1f} л/100 км",
                  bg=ACCENT, fg="white", font=("Arial", 9, "bold")).pack()
        tk.Label(result_frame, text=f"{item['total_cost']:.2f} ₽",
                  bg=ACCENT, fg="white", font=("Arial", 11, "bold")).pack()

        # Действия
        actions = tk.Frame(row, bg=CARD)
        actions.pack(side="left")

        def open_calculation(item_data):
            # Открывает калькулятор с предзаполненными данными
            show_calc()
            # Здесь можно добавить логику для предзаполнения полей калькулятора
            messagebox.showinfo("Детали расчёта",
                               f"Дата: {item_data['date']}\n"
                               f"Расход: {item_data['consumption']:.1f} л/100 км\n"
                               f"Стоимость: {item_data['total_cost']:.2f} ₽")

        def delete_calculation(index):
            data["users"][current_user]["history"].pop(index)
            save_data()
            show_history()  # Обновляем отображение истории

        # Находим индекс текущего элемента в истории
        index = data["users"][current_user]["history"].index(item)

        tk.Button(actions, text="Открыть", bg=PANEL, fg="white", width=6,
                  command=lambda idx=index: open_calculation(data["users"][current_user]["history"][idx])).pack(side="left", padx=2)
        tk.Button(actions, text="❌", bg=DANGER, fg="white", width=2,
                  command=lambda idx=index: delete_calculation(idx)).pack(side="left")

def clear_history():
    if current_user and messagebox.askyesno("Подтверждение", "Очистить всю историю расчётов?"):
        data["users"][current_user]["history"] = []
        save_data()
        show_history()

# ===================== OTHER =====================
def show_settings():
    clear()
    tk.Label(content, text="Настройки", bg=BG, fg=TEXT).pack(pady=40)

def show_about():
    clear()
    tk.Label(content, text="CalculatCar 🚗", bg=BG, fg=TEXT).pack(pady=40)

def update_user():
    user_label.config(text=current_user if current_user else "")

# ===================== START =====================
load_data()
show_calc()
root.mainloop()
