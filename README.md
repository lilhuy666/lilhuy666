import tkinter as tk
from tkinter import ttk, messagebox
from datetime import datetime
import json, os
import hashlib, secrets

# ===================== СТИЛИ =====================
class Styles:
    BG = "#0b1220"
    PANEL = "#111a2e"
    CARD = "#162238"
    ACCENT = "#4f8cff"
    ACCENT2 = "#22c55e"
    TEXT = "#e5e7eb"
    SUB = "#94a3b8"
    DANGER = "#ef4444"
    PROFILE_BG = "#162238"
    AVATAR_BG = "#253145"
    AUTH_BG = "#f8f9fa"

# ===================== ДАННЫЕ =====================
DATA_FILE = "data.json"
BACKUP_FILE = "data.json.backup"
data = {"users": {}}
current_user = None

# ===================== БЕЗОПАСНОСТЬ =====================
def hash_password(password: str, salt: str | None = None) -> str:
    """Хеширует пароль с солью."""
    if salt is None:
        salt = secrets.token_hex(16)
    hashed = hashlib.pbkdf2_hmac('sha256', password.encode(), salt.encode(), 100000)
    return f"{salt}:{hashed.hex()}"

def verify_password(stored: str, provided: str) -> bool:
    """Проверяет пароль по хешу."""
    salt, _ = stored.split(':')
    return hash_password(provided, salt) == stored

# ===================== РАБОТА С ФАЙЛАМИ =====================
def load_data() -> None:
    """Загружает данные из файла."""
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
    create_backup()

def create_backup() -> None:
    """Создаёт резервную копию данных."""
    try:
        with open(DATA_FILE, "r", encoding="utf-8") as fin, open(BACKUP_FILE, "w", encoding="utf-8") as fout:
            fout.write(fin.read())
    except FileNotFoundError:
        pass  # Файла нет — нечего бэкапить

def save_data() -> None:
    """Сохраняет данные в файл с предварительным бэкапом."""
    try:
        create_backup()
        with open(DATA_FILE, "w", encoding="utf-8") as f:
            json.dump(data, f, indent=2, ensure_ascii=False)
    except Exception as e:
        messagebox.showerror("Ошибка", f"Не удалось сохранить данные: {e}")

# ===================== ГЛАВНОЕ ОКНО =====================
class App(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("CalculatCar")
        self.geometry("1200x750")
        self.resizable(True, True)
        self.configure(bg=Styles.BG)
        
        # Стили ttk
        self.style = ttk.Style()
        self.style.theme_use('clam')
        
        # Контейнеры
        self.header_frame = tk.Frame(self, bg=Styles.PANEL, height=70)
        self.header_frame.pack(fill="x")
        
        self.content_frame = tk.Frame(self, bg=Styles.BG)
        self.content_frame.pack(fill="both", expand=True)
        
        # Состояние
        self.menu_window = None
        
        self.create_header()
        self.update_user_label()
        
        # Запуск с калькулятора
        self.show_calc()

    def create_header(self):
        """Создаёт шапку приложения."""
        tk.Button(
            self.header_frame,
            text="☰",
            bg=Styles.PANEL,
            fg=Styles.TEXT,
            font=("Arial", 16, "bold"),
            bd=0,
            command=self.toggle_menu,
            relief="flat",
            activebackground=Styles.CARD,
            activeforeground=Styles.ACCENT,
        ).pack(side="left", padx=15)

        tk.Label(
            self.header_frame,
            text="CalculatCar",
            bg=Styles.PANEL,
            fg=Styles.TEXT,
            font=("Arial", 20, "bold"),
        ).pack(side="left")

        self.user_label = tk.Label(
            self.header_frame,
            text="",
            bg=Styles.PANEL,
            fg=Styles.TEXT,
            font=("Arial", 11),
        )
        self.user_label.pack(side="right", padx=15)

    def update_user_label(self):
        """Обновляет имя пользователя в шапке."""
        if current_user:
            self.user_label.config(text=f"Пользователь: {current_user}")
        else:
            self.user_label.config(text="Гость")

    def clear_content(self):
        """Очищает контентную область."""
        for widget in self.content_frame.winfo_children():
            widget.destroy()

    def toggle_menu(self):
        """Показывает/скрывает боковое меню."""
        if self.menu_window and self.menu_window.winfo_exists():
            self.menu_window.destroy()
            return

        x = self.winfo_x() + 20
        y = self.winfo_y() + 70

        self.menu_window = tk.Toplevel(self)
        self.menu_window.overrideredirect(True)
        self.menu_window.configure(bg=Styles.PANEL)
        self.menu_window.geometry(f"220x240+{x}+{y}")

        def close_menu(e=None):
            if self.menu_window:
                self.menu_window.destroy()

        self.menu_window.bind("<FocusOut>", close_menu)
        self.menu_window.bind("<Escape>", close_menu)

        def nav_button(text, command):
            btn = tk.Button(
                self.menu_window,
                text=text,
                bg=Styles.PANEL,
                fg=Styles.TEXT,
                bd=0,
                anchor="w",
                padx=15,
                pady=10,
                font=("Arial", 11),
                activebackground=Styles.CARD,
                activeforeground=Styles.ACCENT,
                command=lambda: [command(), close_menu()],
                relief="flat",
                highlightthickness=0,
            )
            btn.pack(fill="x")

        nav_button("Калькулятор", self.show_calc)
        nav_button("Профиль", self.show_profile)
        nav_button("История", self.show_history)
        nav_button("Настройки", self.show_settings)
        nav_button("О программе", self.show_about)

# ===================== АВТОРИЗАЦИЯ =====================
class AuthForm(tk.Frame):
    def __init__(self, parent, app_instance):
        super().__init__(parent, bg=Styles.AUTH_BG, padx=30, pady=30)
        
        tk.Label(
            self,
            text="Авторизация",
            bg=Styles.AUTH_BG,
            fg=Styles.ACCENT,
            font=("Arial", 18, "bold"),
        ).pack(pady=(0, 20))

        # Поля ввода
        fields = [
            ("Логин:", "login"),
            ("Пароль:", "password"),
        ]
        
        self.entries = {}
        
        for label_text, key in fields:
            frame_row = tk.Frame(self, bg=Styles.AUTH_BG)
            frame_row.pack(anchor="w")
            
            tk.Label(
                frame_row,
                text=f"{label_text}",
                bg=Styles.AUTH_BG,
                fg=Styles.TEXT,
                font=("Arial", 12),
                anchor="w",
                width=10,
                justify="right",
                padx=(0, 5),
            ).pack(side="left")
            
            entry = ttk.Entry(frame_row, font=("Arial", 12), width=25)
            entry.pack(side="left")
            
            if key == "password":
                entry.config(show="*")
                
            self.entries[key] = entry

        # Кнопки
        btn_frame = tk.Frame(self, bg=Styles.AUTH_BG)
        
        ttk.Button(
             btn_frame,
             text="Войти",
             style="Accent.TButton",
             command=self.login_action,
         ).pack(side="left", padx=(0, 10), pady=(20, 0))
         
         ttk.Button(
             btn_frame,
             text="Зарегистрироваться",
             style="AccentGreen.TButton",
             command=self.register_action,
         ).pack(side="left", padx=(10, 0), pady=(20, 0))
         
         btn_frame.pack()
         
         # Подсказка
         tk.Label(
             self,
             text="Используйте существующие учётные данные или создайте новую",
             bg=Styles.AUTH_BG,
             fg=Styles.SUB,
             font=("Arial", 9),
             justify="center",
         ).pack(pady=(20, 0))
         
         # Сохраняем ссылку на главное окно для обновления статуса
         self.app_instance = app_instance

    def get_values(self):
       return {k: v.get().strip() for k, v in self.entries.items()}
       
    def login_action(self):
       values = self.get_values()
       login, password = values["login"], values["password"]
       
       if not login or not password:
           messagebox.showerror("Ошибка", "Заполните все поля")
           return

       user_data = data["users"].get(login)
       if user_data and verify_password(user_data["password"], password):
           global current_user
           current_user = login
           self.app_instance.update_user_label()
           self.app_instance.show_profile()
       else:
           messagebox.showerror("Ошибка", "Неверный логин или пароль")
           
    def register_action(self):
       values = self.get_values()
       login, password = values["login"], values["password"]
       
       if not login or not password:
           messagebox.showerror("Ошибка", "Заполните все поля")
           return

       if len(password) < 6:
           messagebox.showerror("Ошибка", "Пароль должен быть не менее 6 символов")
           return

       if login in data["users"]:
           messagebox.showerror("Ошибка", "Пользователь с таким логином уже существует")
           return

       data["users"][login] = {
           "name": login,
           "password": hash_password(password),
           "registration_date": datetime.now().strftime("%d.%m.%Y"),
           "notifications": True,
           "history": [],
           "cars": []
       }
       save_data()
       
       global current_user
       current_user = login
       self.app_instance.update_user_label()
       self.app_instance.show_profile()

# ===================== ПРОФИЛЬ =====================
class ProfilePage(tk.Frame):
    def __init__(self, parent, app_instance):
      super().__init__(parent, bg=Styles.PROFILE_BG, padx=20, pady=20)
      
      user_data = data["users"].get(current_user) or {}
      
      # Заголовок и кнопка назад
      header_frame = tk.Frame(self, bg=Styles.PANEL, height=60)
      header_frame.pack(fill="x", pady=(0, 15))
      header_frame.pack_propagate(False)
      
      ttk.Button(
          header_frame,
          text="← Назад",
          style="Danger.TButton",
          command=lambda: [app_instance.clear_content(), app_instance.show_main_menu()],
      ).place(x=10, y=15)
      
      tk.Label(
          header_frame,
          text=f"Профиль пользователя: {user_data.get('name', 'Гость')}",
          bg=Styles.PANEL,
          fg=Styles.ACCENT,
          font=("Arial", 16, "bold"),
      ).pack(pady=(10))
      
      # Разделение на колонки
      grid_frame = tk.Frame(self, bg=Styles.PROFILE_BG)
      grid_frame.pack(fill="both", expand=True)
      
      left_col = self.create_left_column(user_data)
      right_col = self.create_right_column(user_data)
      
      left_col.grid(row=0, column=0, padx=(0, 20), sticky="n")
      right_col.grid(row=0, column=1, sticky="nsew")
      
      grid_frame.columnconfigure(1, weight=1)
      
      # Сохраняем ссылку для обновления данных в реальном времени (если нужно)
      self.user_data_ref = user_data

    def create_left_column(self, user_data):
      frame = tk.Frame(self.parent_ref if hasattr(self, 'parent_ref') else self.master.parent_ref,
                      bg=Styles.CARD, padx=20, pady=20)
      
      name_var = tk.StringVar(value=user_data.get("name", ""))
      
      ttk.Label(frame, text=f"Пользователь: {current_user}", style='CardText.TLabel').pack()
      
      name_entry = ttk.Entry(frame, textvariable=name_var)
      name_entry.pack(fill='x', pady=(15, 5))
      
      def save_name():
          user_data["name"] = name_var.get().strip()
          save_data()
          messagebox.showinfo("OK", "Имя обновлено")
          
      ttk.Button(frame, text="Сохранить имя", style='Accent.TButton', command=save_name).pack(fill='x', pady=(5))
      
      # Смена пароля
      ttk.Separator(frame).pack(fill='x', pady=(25, 5))
      ttk.Label(frame, text="Смена пароля", style='CardText.TLabel').pack(pady=(5,))
      
      old_p = ttk.Entry(frame, show='*')
      new_p = ttk.Entry(frame, show='*')
      
      old_p.pack(fill='x', pady=(2,))
      new_p.pack(fill='x', pady=(2,))
      
      def change_password():
          if not verify_password(user_data["password"], old_p.get()):
              return messagebox.showerror("Ошибка", "Старый пароль неверный")
              
          if len(new_p.get()) < 6:
              return messagebox.showerror("Ошибка", "Пароль слишком короткий")
              
          user_data["password"] = hash_password(new_p.get())
         save_data()
          messagebox.showinfo("OK", "Пароль изменён")

      ttk.Button(
          frame,
          text="Изменить пароль",
          style='AccentGreen.TButton',
          command=change_password,
      ).pack(fill='x', pady=(5,))

      ttk.Label(
          frame,
          text=f"Регистрация: {user_data.get('registration_date', '-')}",
          style='SubText.TLabel'
      ).pack(pady=(25,))

      return frame

    def create_right_column(self, user_data):
      frame = tk.Frame(self, bg=Styles.CARD, padx=20, pady=20)

      # -------- УВЕДОМЛЕНИЯ --------
      notify_var = tk.BooleanVar(value=user_data.get("notifications", True))

      ttk.Checkbutton(
          frame,
          text="Уведомления",
          variable=notify_var,
          style='CardText.TLabel'
      ).pack(anchor='w')

      def save_notify():
          user_data["notifications"] = notify_var.get()
          save_data()
          messagebox.showinfo("OK", "Настройки сохранены")

      ttk.Button(
          frame,
          text="Сохранить настройки",
          style='Accent.TButton',
          command=save_notify
      ).pack(fill='x', pady=(5,))

      # -------- ИСТОРИЯ --------
      ttk.Separator(frame).pack(fill='x', pady=(15, 5))
      ttk.Label(frame, text="История заказов", style='CardText.TLabel').pack(anchor='w')

      history_list = tk.Listbox(frame, height=4)
      history_list.pack(fill='x', pady=(5, 10))

      for item in user_data.get("history", []):
         history_list.insert(tk.END, item["result"])

      # -------- АВТОМОБИЛИ --------
      ttk.Separator(frame).pack(fill='x', pady=(15, 5))
      ttk.Label(frame, text="Автомобили", style='CardText.TLabel').pack(anchor='w')

      cars_list = tk.Listbox(frame, height=4)
      cars_list.pack(fill='x', pady=(5, 10))

      for car in user_data.get("cars", []):
         cars_list.insert(tk.END, f"{car['make']} {car['model']} ({car['year']})")

      def car_form(edit_index=None):
         win = tk.Toplevel(self)
         win.geometry("350x250")
         win.title("Автомобиль")
         
         makes = ["Toyota", "BMW", "Audi", "Ford"]
         make_var = tk.StringVar(value=makes[0])
         
         model_entry = ttk.Entry(win)
         year_entry = ttk.Entry(win)
         
         ttk.Combobox(win, textvariable=make_var, values=makes).pack(fill='x', pady=(5))
         model_entry.pack(fill='x', pady=(5))
         year_entry.pack(fill='x', pady=(5))
         
         if edit_index is not None and edit_index < len(user_data["cars"]):
             car_editing = user_data["cars"][edit_index]
             make_var.set(car_editing["make"])
             model_entry.insert(0, car_editing["model"])
             year_entry.insert(0, str(car_editing["year"]))
             
         def save_car():
             model_val = model_entry.get().strip()
             year_val = year_entry.get().strip()
             
             if not model_val or not year_val.isdigit():
                 return messagebox.showerror("Ошибка", "Некорректные данные автомобиля.")
                 
             car_data = {
                 "make": make_var.get(),
                 "model": model_val,
                 "year": int(year_val),
             }
             
             if edit_index is None:
                 user_data.setdefault("cars", []).append(car_data)
             else:
                 user_data["cars"][edit_index] = car_data
                 
             save_data()
             cars_list.delete(0, tk.END)
             for c in sorted(user_data["cars"], key=lambda x: x['year']):
                 cars_list.insert(tk.END, f"{c['make']} {c['model']} ({c['year']})")
                 
             win.destroy()
             
         ttk.Button(win, text="Сохранить автомобиль", style='Accent.TButton', command=save_car).pack(pady=(15,))

      btn_car_frame = tk.Frame(frame)
      btn_car_frame.pack(fill='x')
      
      ttk.Button(btn_car_frame, text="Добавить", style='Accent2.TButton', command=lambda: car_form()).pack(side='left', expand=True, padx=5)
      
      def edit_car():
          sel = cars_list.curselection()
          if not sel:
              return messagebox.showwarning("Ошибка", "Выберите авто")
          car_form(sel[0])
          
      def delete_car():
          sel = cars_list.curselection()
          if not sel:
              return
              
          if messagebox.askyesno("Удаление", "Удалить авто?"):
              user_data["cars"].pop(sel[0])
              save_data()
              cars_list.delete(0, tk.END)
              for c in sorted(user_data["cars"], key=lambda x: x['year']):
                  cars_list.insert(tk.END, f"{c['make']} {c['model']} ({c['year']})")
              
      ttk.Button(btn_car_frame, text="Редактировать", command=edit_car).pack(side='left', expand=True, padx=5)
      ttk.Button(btn_car_frame, text="Удалить", style='Danger.TButton', command=delete_car).pack(side='left', expand=True, padx=5)

      # -------- АККАУНТ --------
      ttk.Separator(frame).pack(fill='x', pady=(25, 5))
      
      def logout():
          global current_user_global
          current_user_global = None
          self.app_instance.update_user_label()
          self.app_instance.show_auth()
          
      def delete_account():
          if messagebox.askyesno("Удаление", "Удалить аккаунт навсегда?"):
              del data_global["users"][current_user_global]
              save_data()
              logout()

      btn_frame = tk.Frame(frame)
      btn_frame.pack(fill='x')
      
      ttk.Button(btn_frame, text="Выйти", style='Danger.TButton', command=logout).pack(side='left', expand=True, padx=5)
      ttk.Button(btn_frame, text="Удалить аккаунт", command=delete_account).pack(side='left', expand=True, padx=5)

      return frame

# ===================== КАЛЬКУЛЯТОР =====================
class CalculatorPage(tk.Frame):
    def __init__(self, parent):
        super().__init__(parent)
        parent.configure(bg=Styles.BG)

        # Заголовок
        tk.Label(
            parent,
            text="Калькулятор расхода топлива",
            bg=Styles.BG,
            fg=Styles.TEXT,
            font=("Arial", 20, "bold"),
        ).pack(pady=20)

        # Основной фрейм калькулятора
        calc_frame = tk.Frame(parent, bg=Styles.CARD, padx=40, pady=40)
        calc_frame.pack(padx=50, pady=30, fill='both', expand=True)

        # Переменная для выбора режима
        calc_mode = tk.StringVar(value="consumption")

        # Кнопки режима
        mode_frame = tk.Frame(calc_frame, bg=Styles.CARD)
        mode_frame.pack(pady=(0, 20), fill='x')

        ttk.Button(
            mode_frame,
            text="Расход на 100 км",
            style='Accent.TButton',
            command=lambda: calc_mode.set("consumption"),
        ).pack(side='left', expand=True, padx=10, fill='x')

        ttk.Button(
            mode_frame,
            text="Стоимость поездки",
            style='AccentGreen.TButton',
            command=lambda: calc_mode.set("cost"),
        ).pack(side='left', expand=True, padx=10, fill='x')

        # Словарь для динамических меток
        labels_texts = {
            'consumption': ('Расстояние (км):', 'Израсходовано топлива (л):'),
            'cost': ('Расстояние (км):', 'Расход топлива (л):'),
        }

        # Создаем метки и поля ввода
        mileage_label = tk.Label(calc_frame, text="Расстояние (км):", bg=Styles.CARD, fg=Styles.TEXT, font=("Arial", 14))
        mileage_label.grid(row=1, column=0, sticky="e", pady=15, padx=(0, 15))
        mileage_entry = tk.Entry(calc_frame, width=25, font=("Arial", 14), bg=Styles.PANEL, fg=Styles.TEXT)
        mileage_entry.grid(row=1, column=1, pady=15, sticky="w")

        fuel_label = tk.Label(calc_frame, text="Израсходовано топлива (л):", bg=Styles.CARD, fg=Styles.TEXT, font=("Arial", 14))
        fuel_label.grid(row=2, column=0, sticky="e", pady=15, padx=(0, 15))
        fuel_entry = tk.Entry(calc_frame, width=25, font=("Arial", 14), bg=Styles.PANEL, fg=Styles.TEXT)
        fuel_entry.grid(row=2, column=1, pady=15, sticky="w")

        price_label = tk.Label(calc_frame, text="Цена за литр (руб):", bg=Styles.CARD, fg=Styles.TEXT, font=("Arial", 14))
        price_label.grid(row=3, column=0, sticky="e", pady=15, padx=(0, 15))
        price_entry = tk.Entry(calc_frame, width=25, font=("Arial", 14), bg=Styles.PANEL, fg=Styles.TEXT)
        price_entry.grid(row=3, column=1, pady=15, sticky="w")

        def update_labels():
            """Обновляет текст меток в зависимости от выбранного режима"""
            mode = calc_mode.get()
            mileage_label.config(text=f"{labels_texts[mode][0]}")
            fuel_label.config(text=f"{labels_texts[mode][1]}")

        # Привязываем обновление меток к изменению режима
        calc_mode.trace_add("write", lambda *args: update_labels())

        def calculate():
            try:
                mileage = float(mileage_entry.get().strip())
                fuel = float(fuel_entry.get().strip())
                price = float(price_entry.get().strip())

                if mileage <= 0 or fuel <= 0 or price <= 0:
                    messagebox.showerror("Ошибка", "Все значения должны быть положительными числами больше нуля")
                    return

                mode = calc_mode.get()
                result_text = ""

                if mode == "consumption":
                    consumption = (fuel / mileage) * 100
                    result_text = f"Расход топлива: {consumption:.2f} л/100км"
                else:  # mode == "cost"
                    total_cost = fuel * price
                    result_text = f"Общая стоимость поездки: {total_cost:.2f} руб"

                result_label.config(text=result_text)

                # Сохраняем в историю пользователя
                if current_user_global:
                    record = {
                        "date": datetime.now().strftime("%d.%m.%Y %H:%M"),
                        "mileage": mileage,
                        "fuel": fuel,
                        "price": price,
                        "mode": "Расход на 100 км" if mode == "consumption" else "Стоимость поездки",
                        "result": result_text,
                        "consumption": (fuel / mileage) * 100 if mode == "consumption" else None,
                        "total_cost": fuel * price if mode == "cost" else None,
                    }
                    data_global["users"][current_user_global]["history"].append(record)
                    save_data()

            except ValueError:
                messagebox.showerror("Ошибка", "Введите корректные числовые значения во всех полях")

        # Кнопка расчета и результат
        calc_btn = ttk.Button(
            calc_frame,
            text="Рассчитать",
            style='Accent.TButton',
            command=calculate,
        )
        calc_btn.grid(row=4, columnspan=2, pady=(25, 15))

        result_label = tk.Label(calc_frame, text="", bg=Styles.CARD, fg="#2d882d", font=("Arial", 16, "bold"))
        result_label.grid(rowspanspan??, columnspan???)
        result_label = tk.Label(calc_frame, text="", bg=Styles.CARD, fg="#2d882d", font=("Arial", 16, "bold"))
        result_label.grid(row=5, column=0, columnspan=2, pady=(0, 15), sticky="we")

        # Дополнительная информация
        info_label = tk.Label(
            calc_frame,
            text="Выберите режим расчёта и введите данные",
            bg=Styles.CARD,
            fg=Styles.SUB,
            font=("Arial", 12)
        )
        info_label.grid(row=6, column=0, columnspan=2, pady=(10, 0), sticky="we")

       # Настройка сетки для адаптивности
        for i in range(7):
            calc_frame.rowconfigure(i, weight=1)
        calc_frame.columnconfigure(1, weight=1)

# ===================== ИСТОРИЯ =====================
class HistoryPage(tk.Frame):
    def __init__(self, parent, app_instance):
        super().__init__(parent)
        parent.configure(bg=Styles.BG)

        tk.Label(
            parent,
            text="История расчётов",
            bg=Styles.BG,
            fg=Styles.TEXT,
            font=("Arial", 20, "bold")
        ).pack(pady=20)

        if not current_user_global or not data_global["users"][current_user_global].get("history"):
            tk.Label(
                parent,
                text="История пуста",
                bg=Styles.BG,
                fg=Styles.SUB,
                font=("Arial", 14)
            ).pack()
            return

        history_frame = tk.Frame(parent, bg=Styles.CARD)
        history_frame.pack(fill='both', expand=True, padx=20, pady=10)

        # Заголовки таблицы
        headers = ["Дата", "Пробег", "Топливо", "Цена", "Расход", "Стоимость"]
        for col, header in enumerate(headers):
            tk.Label(
                history_frame,
                text=header,
                bg=Styles.PANEL,
                fg=Styles.TEXT,
                font=("Arial", 10, "bold"),
                padx=10
            ).grid(row=0, column=col, sticky="ew")

        # Данные истории
        history = data_global["users"][current_user_global]["history"]
        for row, record in enumerate(history, start=1):
            values = [
                record["date"],
                f"{record['mileage']} км",
                f"{record['fuel']} л",
                f"{record['price']} руб",
                f"{record['consumption']:.2f} л/100км" if record['consumption'] is not None else "-",
                f"{record['total_cost']:.2f} руб" if record['total_cost'] is not None else "-",
            ]
            for col, value in enumerate(values):
                tk.Label(
                    history_frame,
                    text=value,
                    bg=Styles.CARD,
                    fg=Styles.TEXT,
                    font=("Arial", 9),
                    padx=5
                ).grid(row=row, column=col, sticky="ew", padx=5, pady=2)

# ===================== НАСТРОЙКИ =====================
class SettingsPage(tk.Frame):
    def __init__(self, parent):
        super().__init__(parent)
        parent.configure(bg=Styles.BG)

        tk.Label(
            parent,
            text="Настройки приложения",
            bg=Styles.BG,
            fg=Styles.TEXT,
            font=("Arial", 20, "bold")
        ).pack(pady=20)

        settings_frame = tk.Frame(parent, bg=Styles.CARD, padx=30, pady=30)
        settings_frame.pack(fill='both', expand=True, padx=20, pady=(10, 20))

        # Тема оформления
        theme_frame = tk.LabelFrame(settings_frame, text="Тема оформления", bg=Styles.CARD, fg=Styles.TEXT, padx=15, pady=15)
        theme_frame.pack(fill='x', pady=(0, 20))

        theme_var = tk.StringVar(value="Тёмная")
        themes = ["Тёмная", "Светлая", "Синяя"]

        for theme in themes:
            ttk.Radiobutton(
                theme_frame,
                text=theme,
                variable=theme_var,
                value=theme,
                style='CardText.TLabel'
            ).pack(anchor="w", pady=(5, 0))

        def apply_theme():
            theme = theme_var.get()
            
            # Сохраняем выбранную тему в данные пользователя
            if current_user_global:
                data_global["users"][current_user_global]["theme"] = theme
                save_data()
                
            # Применяем тему к интерфейсу
            if theme == "Тёмная":
                apply_dark_theme()
            elif theme == "Светлая":
                apply_light_theme()
            elif theme == "Синяя":
                apply_blue_theme()
                
            messagebox.showinfo("Тема", f"Тема '{theme}' успешно применена")

        ttk.Button(
            theme_frame,
            text="Применить тему",
            style='Accent.TButton',
            command=apply_theme
        ).pack(pady=(15, 5))

# ===================== О ПРОГРАММЕ =====================
class AboutPage(tk.Frame):
    def __init__(self, parent):
        super().__init__(parent)
        parent.configure(bg=Styles.BG)

        about_frame = tk.Frame(parent, bg=Styles.CARD, padx=40, pady=40)
        about_frame.pack(fill='both', expand=True, padx=(20, 50), pady=(20, 50))

        tk.Label(
            about_frame,
            text="CalculatCar",
            bg=Styles.CARD,
            fg=Styles.ACCENT,
            font=("Arial", 24, "bold")
        ).pack(pady=(0, 10))

        tk.Label(
            about_frame,
            text="Версия 1.0.0",
            bg=Styles.CARD,
            fg=Styles.SUB,
            font=("Arial", 12)
        ).pack(pady=(0, 20))

        features = [
            "Калькулятор расхода топлива",
            "История расчётов с возможностью повторного использования",
            "Персональный профиль с данными автомобиля",
            "Система уведомлений",
            "Резервное копирование данных"
        ]

        tk.Label(
            about_frame,
            text="Возможности:",
            bg=Styles.CARD,
            fg=Styles.TEXT,
            font=("Arial", 14, "bold")
        ).pack(anchor="w", pady=(0, 5))

        for feature in features:
             tk.Label(
                 about_frame,
                 text=f"• {feature}",
                 bg=Styles.CARD,
                 fg=Styles.SUB,
                 font=("Arial", 11),
                 justify="left"
             ).pack(anchor="w", pady=(2, 2))

         tk.Label(
             about_frame,
             text="© 2024 CalculatCar. Все права защищены.",
             bg=Styles.CARD,
             fg=Styles.SUB,
             font=("Arial", 10)
         ).pack(side="bottom", pady=(30, 0))
         # ===================== ФУНКЦИИ СМЕНЫ ТЕМЫ =====================
def apply_dark_theme():
    global Styles
    Styles.BG = "#0b1220"
    Styles.PANEL = "#111a2e"
    Styles.CARD = "#162238"
    Styles.ACCENT = "#4f8cff"
    Styles.ACCENT2 = "#22c55e"
    Styles.TEXT = "#e5e7eb"
    Styles.SUB = "#94a3b8"
    Styles.DANGER = "#ef4444"
    Styles.PROFILE_BG = "#162238"
    Styles.AVATAR_BG = "#253145"
    refresh_ui()

def apply_light_theme():
    global Styles
    Styles.BG = "#f8fafc"
    Styles.PANEL = "#e2e8f0"
    Styles.CARD = "#ffffff"
    Styles.ACCENT = "#3b82f6"
    Styles.ACCENT2 = "#10b981"
    Styles.TEXT = "#1e293b"
    Styles.SUB = "#64748b"
    Styles.DANGER = "#dc2626"
    Styles.PROFILE_BG = "#ffffff"
    Styles.AVATAR_BG = "#e2e8f0"
    refresh_ui()

def apply_blue_theme():
    global Styles
    Styles.BG = "#0c1b33"
    Styles.PANEL = "#1a2b4d"
    Styles.CARD = "#2a3f66"
    Styles.ACCENT = "#60a5fa"
    Styles.ACCENT2 = "#818cf8"
    Styles.TEXT = "#dbeafe"
    Styles.SUB = "#93c5fd"
    Styles.DANGER = "#fca5a5"
    Styles.PROFILE_BG = "#2a3f66"
    Styles.AVATAR_BG = "#384e77"
    refresh_ui()

def refresh_ui():
    """Обновляет цвета всех элементов интерфейса"""
    if 'app_instance' in globals() and app_instance:
         app_instance.configure(bg=Styles.BG)
         app_instance.header_frame.configure(bg=Styles.PANEL)
         app_instance.user_label.configure(bg=Styles.PANEL, fg=Styles.TEXT)
         
         # Обновляем все фреймы и метки в content
         for widget in app_instance.content_frame.winfo_children():
             if isinstance(widget, tk.Frame):
                 widget.configure(bg=CARD_COLOR())
             elif isinstance(widget, tk.Label):
                 if widget.cget("text") in ["Калькулятор расхода топлива", "История расчётов", "Настройки приложения", "О программе"]:
                     widget.configure(bg=CARD_COLOR(), fg=DANGER_COLOR())
                 else:
                     widget.configure(bg=CARD_COLOR(), fg=DANGER_COLOR())
# ... (Здесь должны быть определения функций hash_password и т.д., которые были в начале) ...


# ===================== ЗАПУСК ПРИЛОЖЕНИЯ =====================
if __name__ == "__main__":
     load_data() # Загружаем данные при старте

     app_instance = App() # Создаем экземпляр приложения

     # Проверяем наличие пользователя и тему из настроек (если есть)
     if current_user_global and data_global["users"][current_user_global].get("theme"):
         theme = data_global["users"][current_user_global]["theme"]
         if theme == "Тёмная":
             apply_dark_theme()
         elif theme == "Светлая":
             apply_light_theme()
         elif theme == "Синяя":
             apply_blue_theme()
             
     app_instance.mainloop() # Запускаем главный цикл
