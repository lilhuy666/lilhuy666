1. Импорт и инициализация (в самом начале файла)
Замените стандартный импорт tkinter на ttkbootstrap и инициализируйте окно с выбранной темой.
python
Копировать
import ttkbootstrap as ttk
from ttkbootstrap.constants import *
# Остальные импорты (json, datetime и т.д.) остаются без изменений

# ===================== WINDOW =====================
root = ttk.Window(themename="superhero")  # Можно менять на: "darkly", "flatly", "cerulean"
root.title("CalculatCar")
root.geometry("1200x750")

main = ttk.Frame(root)  # bg не нужен, тема задаётся в Window
main.pack(fill="both", expand=True)
2. Заголовок (Header)
Замените стандартные виджеты на ttkbootstrap. Кнопка с иконкой теперь использует встроенные стили.
python
Копировать
# ===================== HEADER =====================
header = ttk.Frame(main, height=70)
header.pack(fill="x")
header.pack_propagate(False)

# Кнопка меню
ttk.Button(header, text="☰", style="primary", command=toggle_menu).pack(side="left", padx=15)

# Заголовок приложения
ttk.Label(header, text="CalculatCar", font=("Arial", 20, "bold")).pack(side="left")

# Статус пользователя
user_label = ttk.Label(header, text="", font=("Arial", 11))
user_label.pack(side="right", padx=15)
3. Форма авторизации (Auth)
python
Копировать
def show_auth():
    clear()
    frame = ttk.Frame(content, padding=(30, 30))
    frame.pack(fill="both", expand=True)

    ttk.Label(frame, text="Авторизация", font=("Arial", 18, "bold")).pack(pady=(0, 20))

    # Поля ввода
    ttk.Label(frame, text="Логин:").pack(anchor="w")
    login_entry = ttk.Entry(frame, width=25)
    login_entry.pack(pady=(0, 15), fill="x")

    ttk.Label(frame, text="Пароль:").pack(anchor="w")
    password_entry = ttk.Entry(frame, width=25, show="*")
    password_entry.pack(pady=(0, 20), fill="x")

    # Кнопки
    def login():
        # Ваш код логина
        pass

    ttk.Button(frame, text="Войти", style="primary", command=login).pack(fill="x", pady=(0, 10))

    def register():
        # Ваш код регистрации
        pass

    ttk.Button(frame, text="Зарегистрироваться", style="success").pack(fill="x")
4. Профиль пользователя (Profile)
python
Копировать
def create_left_column(parent, user):
    frame = ttk.Frame(parent, padding=(20, 20))

    # Имя пользователя
    name_var = tk.StringVar(value=user.get("name", ""))
    
    ttk.Label(frame, text=current_user, font=("Arial", 12, "bold")).pack()
    
    ttk.Label(frame, text="Имя:").pack(anchor="w", pady=(10, 0))
    
    name_entry = ttk.Entry(frame, textvariable=name_var)
    name_entry.pack(fill="x")

    def save_name():
        # Ваш код сохранения имени
        pass

    ttk.Button(frame, text="Сохранить имя", style="primary", command=save_name).pack(fill="x", pady=5)

    # Смена пароля (аналогично)
    # ...
    
    return frame

def create_right_column(parent, user):
    frame = ttk.Frame(parent, padding=(15, 15))

    # Уведомления
    notify_var = tk.BooleanVar(value=user.get("notifications", True))
    
    ttk.Checkbutton(frame, text="Уведомления", variable=notify_var).pack(anchor="w")

    def save_notify():
        # Ваш код сохранения настроек
        pass

    ttk.Button(frame, text="Сохранить настройки", style="primary", command=save_notify).pack(fill="x", pady=5)

    # История и автомобили (Listbox остаётся стандартным или стилизуется отдельно)
    # ...
    
    # Кнопки выхода и удаления аккаунта
    def logout():
        # Ваш код выхода
        pass

    def delete_account():
        # Ваш код удаления
        pass

    ttk.Button(frame, text="Выйти", style="danger", command=logout).pack(fill="x", pady=10)
    ttk.Button(frame, text="Удалить аккаунт", style="danger-outline", command=delete_account).pack(fill="x")
    
    return frame
5. Калькулятор (Calculator)
python
Копировать
def show_calc():
    clear()
    
    # Заголовок
    ttk.Label(content, text="Калькулятор расхода топлива",
              font=("Arial", 20, "bold")).pack(pady=20)

    calc_frame = ttk.Frame(content, padding=(40, 40))
    calc_frame.pack(fill="both", expand=True, padx=50, pady=30)

    # Режимы расчёта (кнопки вместо радиокнопок)
    mode_frame = ttk.Frame(calc_frame)
    mode_frame.pack(pady=(0, 20), fill="x")

    calc_mode = tk.StringVar(value="consumption")

    def set_mode(m):
        calc_mode.set(m)
    
    btn_style = "secondary"
    
    consumption_btn = ttk.Button(mode_frame, text="Расход на 100 км",
                                 style=btn_style,
                                 command=lambda: set_mode("consumption"))
    consumption_btn.pack(side="left", fill="x", expand=True, padx=(0, 10))

    cost_btn = ttk.Button(mode_frame, text="Стоимость поездки",
                          style=btn_style,
                          command=lambda: set_mode("cost"))
    cost_btn.pack(side="left", fill="x", expand=True, padx=(10, 0))

    # Поля ввода (Entry)
    mileage_label = ttk.Label(calc_frame, text="Расстояние (км):")
    mileage_label.grid(row=1, column=0, sticky="e", pady=15, padx=(0, 15))
    
    mileage_entry = ttk.Entry(calc_frame, width=25)
    mileage_entry.grid(row=1, column=1, pady=15, sticky="w")

    fuel_label = ttk.Label(calc_frame, text="Израсходовано топлива (л):")
    fuel_label.grid(row=2, column=0, sticky="e", pady=15, padx=(0, 15))
    
    fuel_entry = ttk.Entry(calc_frame, width=25)
    fuel_entry.grid(row=2, column=1, pady=15, sticky="w")

    # Остальные поля и кнопка расчёта аналогично...
6. Настройки (Settings) — Тема оформления
python
Копировать
def show_settings():
    clear()
    
    # ...
    
    theme_frame = ttk.LabelFrame(settings_frame, text="Тема оформления")
    theme_frame.pack(fill="x", pady=(0, 20))

    theme_var = tk.StringVar(value="Тёмная")
    themes = ["Тёмная", "Светлая", "Синяя"]

    for theme in themes:
        rb = ttk.Radiobutton(theme_frame,
                             text=theme,
                             variable=theme_var,
                             value=theme,
                             style=f"Toolbutton") # Стиль для радиокнопок в LabelFrame
        rb.pack(anchor="w", pady=5)
Как это работает:
