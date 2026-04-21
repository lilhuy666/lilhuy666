import tkinter as tk
from tkinter import messagebox
from datetime import datetime
import json
import os
import hashlib
import ttkbootstrap as ttk
from ttkbootstrap.constants import *
import re

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

FONT_MAIN = ("Segoe UI", 11)
FONT_TITLE = ("Segoe UI", 16, "bold")
FONT_BUTTON = ("Segoe UI", 10, "bold")

# ===================== DATA =====================
data = {"users": {}}
current_user = None

def hash_password(password, salt="calculatcar_salt"):
    """Хеширует пароль с солью с помощью SHA-256"""
    return hashlib.sha256((password + salt).encode()).hexdigest()

def is_valid_email(email):
    """Проверяет корректность email"""
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return re.match(pattern, email) is not None

def load_data():
    global data
    if os.path.exists(DATA_FILE):
        try:
            with open(DATA_FILE, "r", encoding="utf-8") as f:
                data = json.load(f)
            if "users" not in data:
                data["users"] = {}
            if "settings" not in data:
                data["settings"] = {"currency": "€", "precision": 2}
        except (json.JSONDecodeError, IOError):
            data = {"users": {}, "settings": {"currency": "€", "precision": 2}}
    else:
        data = {"users": {}, "settings": {"currency": "€", "precision": 2}}

def save_data():
    """Сохраняет данные в файл с обработкой ошибок"""
    try:
        with open(DATA_FILE, "w", encoding="utf-8") as f:
            json.dump(data, f, indent=2, ensure_ascii=False)
    except IOError as e:
        messagebox.showerror("Ошибка", f"Не удалось сохранить данные: {e}")

# ===================== WINDOW =====================
root = ttk.Window(themename="darkly")
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
def show_calc():
    # код калькулятора
    pass

def show_profile():
    # код профиля
    pass

def show_history():
    # код истории
    pass

def show_settings():
    # код настроек
    pass

def show_about():
    clear()
    c = card()
    tk.Label(c, text="CalculatCar 🚗",
             bg=CARD, fg=TEXT,
             font=("Arial", 24, "bold")).pack(pady=15)
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

def update_user():
    user_label.config(text=current_user if current_user else "")

# ===================== CALCULATOR =====================
def show_calc():
    try:
        clear()
        # остальной код калькулятора...
    except Exception as e:
        messagebox.showerror("Ошибка", f"Ошибка при загрузке калькулятора: {str(e)}")

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

    result_label = tk.Label(content, text="—",
                         bg=BG, fg=ACCENT,
                         font=("Arial", 22, "bold"))
    result_label.pack()

    def build():
        for w in form.winfo_children():
            w.destroy()
        entries.clear()

        def add(t):
            tk.Label(form, text=t, bg=CARD, fg=TEXT).pack(anchor="w")
            e = tk.Entry(form, font=("Arial", 12))
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
            precision = data["settings"]["precision"]
            currency = data["settings"]["currency"]

            if mode.get() == "1":
                f = float(entries["Топливо (л)"].get())
                d = float(entries["Расстояние (км)"].get())
                p = float(entries["Цена за литр"].get())

                if f < 0 or d <= 0 or p < 0:
                    raise ValueError("Значения должны быть положительными, расстояние > 0")

                cons = (f / d) * 100
                cost = f * p
            else:
                cons = float(entries["Средний расход (л/100км)"].get())
                d = float(entries["Расстояние (км)"].get())
                p = float(entries["Цена за литр"].get())

                if cons < 0 or d < 0 or p < 0:
                    raise ValueError("Значения должны быть неотрицательными")

                cost = cons * d / 100 * p

            result_text = f"{cons:.{precision}f} л/100км | {cost:.{precision}f} {currency}"
            result_label.config(text=result_text)

            if current_user:
                h = data["users"][current_user]["history"]
                h.append({
                    "date": datetime.now().strftime("%d.%m %H:%M"),
                    "result": result_text
                })
                data["users"][current_user]["history"] = h[-50:]
                save_data()

        except ValueError as e:
            if "could not convert" in str(e):
                messagebox.showerror("Ошибка", "Введите корректные числа")
            else:
                messagebox.showerror("Ошибка", str(e))
        except ZeroDivisionError:
            messagebox.showerror("Ошибка", "Расстояние не может быть нулевым")
        except KeyError:
            messagebox.showerror("Ошибка", "Заполните все поля")
        except Exception as e:
            messagebox.showerror("Ошибка", f"Неожиданная ошибка: {str(e)}")

        tk.Button(content, text="Рассчитать",
                  bg=ACCENT, fg="white",
                  font=("Arial", 14, "bold"),
                  command=calc).pack(pady=10)

        mode.trace("w", lambda *a: build())
        build()


# ===================== HISTORY =====================
def clear_history():
    """Очищает историю расчётов текущего пользователя"""
    if not current_user:
        messagebox.showwarning("Предупреждение", "Войдите в аккаунт для работы с историей")
        return

    if messagebox.askyesno("Очистка истории", "Вы уверены, что хотите очистить всю историю расчётов?"):
        try:
            data["users"][current_user]["history"] = []
            save_data()
            show_history()  # Обновляем отображение
        except KeyError:
            messagebox.showerror("Ошибка", "Данные пользователя повреждены")
        except Exception as e:
            messagebox.showerror("Ошибка", f"Не удалось очистить историю: {str(e)}")


def show_history():
    clear()

    tk.Label(content, text="История расчётов",
             bg=BG, fg=TEXT,
             font=("Arial", 20, "bold")).pack(pady=10)

    if not current_user:
        tk.Label(content, text="Войдите в аккаунт для просмотра истории",
                 bg=BG, fg=SUB,
                 font=FONT_MAIN).pack(pady=20)
        return

    for widget in content.winfo_children():
        if isinstance(widget, tk.Frame) and widget != content:
            widget.destroy()

    history = data["users"][current_user].get("history", [])

    history_frame = tk.Frame(content, bg=BG)
    history_frame.pack(fill="both", expand=True, padx=40, pady=20)

    canvas = tk.Canvas(history_frame, bg=CARD, highlightthickness=0)
    scrollbar = tk.Scrollbar(history_frame, orient="vertical", command=canvas.yview)
    scrollable_frame = tk.Frame(canvas, bg=CARD)

    scrollable_frame.bind(
        "<Configure>",
        lambda e: canvas.configure(scrollregion=canvas.bbox("all"))
    )

    canvas.create_window((0, 0), window=scrollable_frame, anchor="nw")
    canvas.configure(yscrollcommand=scrollbar.set)

    canvas.pack(side="left", fill="both", expand=True)
    scrollbar.pack(side="right", fill="y")

    if not history:
        empty_label = tk.Label(scrollable_frame, text="История расчётов пуста",
                               bg=CARD, fg=SUB,
                               font=FONT_MAIN,
                               padx=20, pady=15)
        empty_label.pack(fill="x")
        clear_btn = tk.Button(content, text="Очистить историю",
                              bg=DANGER, fg="white",
                              command=clear_history,
                              state="disabled")
        clear_btn.pack(pady=10)
        return

    for h in reversed(history):
        item_frame = tk.Frame(scrollable_frame, bg=CARD, bd=1, relief="solid")
        item_frame.pack(fill="x", pady=5, padx=10)

        date_label = tk.Label(item_frame,
                              text=f"{h['date']}",
                              bg=CARD, fg=ACCENT,
                              font=("Arial", 10, "bold"),
                              anchor="w")
        date_label.pack(side="top", fill="x", padx=5)

        result_label = tk.Label(item_frame,
                                text=h['result'],
                                bg=CARD, fg=TEXT,
                                font=("Arial", 11),
                                anchor="w")
        result_label.pack(side="top", fill="x", padx=5, pady=(0, 5))

    clear_btn = tk.Button(content, text="Очистить историю",
                          bg=DANGER, fg="white",
                          command=clear_history)
    clear_btn.pack(pady=10)


# ===================== SETTINGS =====================
def show_settings():
    clear()

    c = card()
    tk.Label(c, text="Настройки",
             bg=CARD, fg=TEXT,
             font=("Arial", 20, "bold")).pack(pady=10)

    currency_frame = tk.Frame(c, bg=CARD)
    currency_frame.pack(fill="x", pady=10)

    tk.Label(currency_frame, text="Валюта:", bg=CARD, fg=SUB).pack(anchor="w")

    currency_var = tk.StringVar(value=data["settings"]["currency"])
    currencies = ["€", "$", "₽", "¥", "£"]

    for curr in currencies:
        tk.Radiobutton(currency_frame,
                       text=curr,
                       variable=currency_var,
                       value=curr,
                       bg=CARD, fg=TEXT,
                       selectcolor=ACCENT).pack(side="left", padx=10)

    precision_frame = tk.Frame(c, bg=CARD)
    precision_frame.pack(fill="x", pady=10)

    tk.Label(precision_frame, text="Точность округления:", bg=CARD, fg=SUB).pack(anchor="w")

    precision_var = tk.IntVar(value=data["settings"]["precision"])
    precisions = [1, 2, 3]

    for prec in precisions:
        tk.Radiobutton(precision_frame,
                       text=f"{prec} знака",
                       variable=precision_var,
                       value=prec,
                       bg=CARD, fg=TEXT,
                       selectcolor=ACCENT).pack(side="left", padx=10)

    def save_settings():
        data["settings"]["currency"] = currency_var.get()
        data["settings"]["precision"] = precision_var.get()
        save_data()
        messagebox.showinfo("Успех", "Настройки сохранены")

    def reset_settings():
        if messagebox.askyesno("Сброс", "Сбросить настройки к значениям по умолчанию?"):
            data["settings"] = {"currency": "€", "precision": 2}
            save_data()
            show_settings()

    tk.Button(c, text="Сохранить настройки",
              bg=ACCENT, fg="white",
              command=save_settings).pack(pady=20)

    tk.Button(c, text="Сбросить к умолчанию",
              bg=DANGER, fg="white",
              command=reset_settings).pack(pady=5)

    # ===================== ABOUT =====================
    def show_about():
        clear()

        c = card()
        tk.Label(c, text="CalculatCar 🚗",
                 bg=CARD, fg=TEXT,
                 font=("Arial", 24, "bold")).pack(pady=15)

        about_text = """
            Калькулятор расхода топлива для автолюбителей


            ОСНОВНЫЕ ВОЗМОЖНОСТИ:
            • Расчёт стоимости поездки
            • Определение среднего расхода топлива
            • История последних 50 расчётов
            • Личный профиль пользователя
            • Сохранение данных между запусками
            • Настраиваемая валюта отображения

            ТЕХНИЧЕСКАЯ ИНФОРМАЦИЯ:
            Версия: 1.0
            Разработчик: CalculatCar Team
            Язык программирования: Python 3
            Графический интерфейс: Tkinter
            Хранение данных: JSON-файл

            ПОДДЕРЖКА:
            Email: support@calculatcar.com
            Сайт: www.calculatcar.com
        """

        tk.Label(c, text=about_text,
                 bg=CARD, fg=SUB,
                 justify="left",
                 font=("Arial", 11),
                 anchor="w").pack(pady=10, padx=20)

        # Кнопка «Проверить обновления»
        def check_updates():
            messagebox.showinfo("Обновления", "У вас установлена последняя версия")

        tk.Button(c, text="Проверить обновления",
                  bg=ACCENT2, fg="black",
                  command=check_updates).pack(pady=10)

        # Кнопка «Лицензия»
        def show_license():
            license_text = """
            CalculatCar — бесплатное программное обеспечение

            Эта программа распространяется на условиях
            свободной лицензии. Вы можете использовать,
            модифицировать и распространять её.

            © 2024 CalculatCar Team. Все права защищены.
            """
            messagebox.showinfo("Лицензия", license_text)

        tk.Button(c, text="Лицензия",
                  bg=PANEL, fg=TEXT,
                  command=show_license).pack(pady=5)

    # ===================== MENU =====================
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
        nav("О программе", show_about)  # Теперь show_about определена!

        menu_window.focus_force()

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

                if not is_valid_email(e):
                    return messagebox.showerror("Ошибка", "Введите корректный email")

                if e in data["users"] and data["users"][e]["password"] == hash_password(p):
                    current_user = e
                    update_user()
                    show_profile()
                else:
                    messagebox.showerror("Ошибка", "Неверные данные")

            def register():
                e = email.get().strip()
                p = password.get().strip()

                if not e or not p:
                    return messagebox.showerror("Ошибка", "Заполните все поля")

                if not is_valid_email(e):
                    return messagebox.showerror("Ошибка", "Введите корректный email")

                if len(p) < 4:
                    return messagebox.showerror("Ошибка", "Пароль должен содержать минимум 4 символа")

                if e in data["users"]:
                    return messagebox.showerror("Ошибка", "Пользователь с таким email уже существует")

                data["users"][e] = {
                    "password": hash_password(p),
                    "name": "",
                    "car": "",
                    "history": []
                }
                save_data()
                messagebox.showinfo("Успех", "Аккаунт создан")

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
        name_entry.pack(side="left", fill="x", expand=True, padx=(0, 10))

        def save_name():
            user["name"] = name_entry.get().strip()
            save_data()
            messagebox.showinfo("Успех", "Имя сохранено")

        name_entry.bind("<FocusOut>", lambda e: save_name())
        tk.Button(row1, text="Сохранить",
                  bg=ACCENT, fg="white",
                  command=save_name).pack(side="right")

        # ==== АВТО ====
        row2 = tk.Frame(c, bg=CARD)
        row2.pack(fill="x", pady=5)

        tk.Label(row2, text="Автомобиль", bg=CARD, fg=SUB).pack(anchor="w")

        car_entry = tk.Entry(row2, font=("Arial", 14))
        car_entry.insert(0, user.get("car", ""))
        car_entry.pack(side="left", fill="x", expand=True, padx=(0, 10))

        def save_car():
            user["car"] = car_entry.get().strip()
            save_data()
            messagebox.showinfo("Успех", "Автомобиль сохранён")

        car_entry.bind("<FocusOut>", lambda e: save_car())
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
        pass2.pack(side="left", fill="x", expand=True, padx=(0, 10))

        def save_password():
            p1 = pass1.get()
            p2 = pass2.get()

            if not p1 or not p2:
                return messagebox.showerror("Ошибка", "Введите пароль")

            if len(p1) < 4:
                return messagebox.showerror("Ошибка", "Пароль должен содержать минимум 4 символа")

            if p1 != p2:
                return messagebox.showerror("Ошибка", "Пароли не совпадают")

            user["password"] = hash_password(p1)
            save_data()
            messagebox.showinfo("Успех", "Пароль изменён")
            pass1.delete(0, tk.END)
            pass2.delete(0, tk.END)

        tk.Button(row4, text="Сохранить",
                  bg=ACCENT, fg="white",
                  command=save_password).pack(side="right")

        def logout():
            if messagebox.askyesno("Выход", "Вы уверены, что хотите выйти?"):
                global current_user
                current_user = None
                update_user()
                show_profile()

        tk.Button(c, text="Выйти",
                  bg=DANGER, fg="white",
                  command=logout).pack(fill="x", pady=10)

    # ===================== INITIALIZATION =====================
if __name__ == "__main__":
 load_data()
 update_user()
 show_calc()
 root.mainloop()
