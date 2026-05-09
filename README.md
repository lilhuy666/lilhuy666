# ─── History Panel Class ────────────────────────────────────────────────────────
class HistoryPanel:
    """Класс для управления панелью истории"""
    
    def __init__(self, parent, app):
        self.parent = parent
        self.app = app
        self.current_page = 1
        self.items_per_page = 10
        self.total_pages = 1
        self.current_history = []
        self.filter_car = None
        self.filter_days = None
        
    def build(self):
        """Построение интерфейса истории"""
        t = T()
        
        # Заголовок
        header_frame = tk.Frame(self.parent, bg=t["bg"])
        header_frame.pack(fill="x", pady=(0, 16))
        
        tk.Label(header_frame, text=TR("history_title"), 
                font=("Georgia", 18, "bold"),
                bg=t["bg"], fg=t["fg"]).pack(side="left")
        
        # Кнопка очистки
        clear_btn = tk.Button(header_frame, text=TR("clear_all"), 
                            font=("Courier", 10),
                            bg=t["red"], fg="white", bd=0, padx=12, pady=6,
                            command=self.clear_all_history)
        clear_btn.pack(side="right")
        
        # Фильтры
        self.filter_frame = tk.Frame(self.parent, bg=t["bg2"], pady=10)
        self.filter_frame.pack(fill="x", pady=(0, 16))
        
        tk.Label(self.filter_frame, text="Фильтр:", 
                font=("Courier", 10, "bold"), bg=t["bg2"], fg=t["fg"]).pack(side="left", padx=10)
        
        tk.Label(self.filter_frame, text="Автомобиль:", 
                font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(side="left", padx=(20, 5))
        
        self.car_filter_var = tk.StringVar(value=TR("all_vehicles"))
        self.car_combo = ttk.Combobox(self.filter_frame, textvariable=self.car_filter_var,
                                     state="readonly", width=20, font=("Courier", 10))
        self.car_combo.pack(side="left", padx=5)
        
        tk.Label(self.filter_frame, text="Период:", 
                font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(side="left", padx=10)
        
        self.days_filter_var = tk.StringVar(value=TR("all_time"))
        days_combo = ttk.Combobox(self.filter_frame, textvariable=self.days_filter_var,
                                 values=[TR("all_time"), TR("days_7"), TR("days_30"), 
                                        TR("days_90"), TR("days_180"), TR("days_365")],
                                 state="readonly", width=12, font=("Courier", 10))
        days_combo.pack(side="left", padx=5)
        
        apply_btn = tk.Button(self.filter_frame, text="Применить", 
                            font=("Courier", 10), bg=t["btn"], fg="white",
                            bd=0, padx=12, pady=4, command=self.apply_filters)
        apply_btn.pack(side="left", padx=10)
        
        # Статистика
        self.stats_frame = tk.Frame(self.parent, bg=t["bg3"])
        self.stats_frame.pack(fill="x", pady=(0, 16))
        
        # Контейнер для записей истории
        self.history_container = tk.Frame(self.parent, bg=t["bg"])
        self.history_container.pack(fill="both", expand=True)
        
        # Пагинация
        self.pagination_frame = tk.Frame(self.parent, bg=t["bg"])
        self.pagination_frame.pack(fill="x", pady=16)
        
        # Загрузка данных
        self.load_cars_for_filter()
        self.refresh()
    
    def load_cars_for_filter(self):
        """Загрузка списка автомобилей для фильтра"""
        cars = [TR("all_vehicles")]
        if self.app.current_user and self.app._db_connected:
            try:
                user_cars = db_get_cars(self.app.current_user)
                cars += [c["name"] for c in user_cars]
            except Exception as e:
                print(f"Error loading cars for filter: {e}")
        
        # Обновляем комбобокс
        self.car_combo['values'] = cars
    
    def apply_filters(self):
        """Применение фильтров"""
        car_filter = self.car_filter_var.get()
        self.filter_car = None if car_filter == TR("all_vehicles") else car_filter
        
        days_filter = self.days_filter_var.get()
        if days_filter != TR("all_time"):
            days_map = {TR("days_7"): 7, TR("days_30"): 30, TR("days_90"): 90, 
                       TR("days_180"): 180, TR("days_365"): 365}
            self.filter_days = days_map.get(days_filter)
        else:
            self.filter_days = None
        
        self.current_page = 1
        self.refresh()
    
    def refresh(self):
        """Обновление истории"""
        if not self.app.current_user or not self.app._db_connected:
            self.show_login_message()
            return
        
        # Загрузка истории
        self.current_history = db_get_history(
            self.app.current_user, 
            self.filter_car, 
            self.filter_days
        )
        
        # Расчет пагинации
        total_items = len(self.current_history)
        self.total_pages = max(1, (total_items + self.items_per_page - 1) // self.items_per_page)
        
        # Отображение статистики
        self.display_stats()
        
        # Отображение записей
        self.display_history_entries()
        
        # Отображение пагинации
        self.display_pagination()
    
    def display_stats(self):
        """Отображение статистики"""
        for w in self.stats_frame.winfo_children():
            w.destroy()
        
        t = T()
        stats = db_get_history_stats(self.app.current_user, self.filter_car)
        
        if stats and stats['total_trips'] > 0:
            # Создаем сетку статистики
            stats_grid = tk.Frame(self.stats_frame, bg=t["bg3"])
            stats_grid.pack(padx=16, pady=12, fill="x")
            
            # Статистические карточки
            stats_items = [
                ("🚗 Всего поездок:", f"{stats['total_trips']}"),
                ("📏 Общий пробег:", f"{stats['total_distance']:.0f} км"),
                ("⛽ Всего топлива:", f"{stats['total_fuel']:.1f} л"),
                ("💧 Средний расход:", f"{stats['avg_consumption']:.1f} л/100км"),
                ("💰 Общая стоимость:", f"{stats['total_cost']:,.2f} {get_currency_symbol()}")
            ]
            
            for i, (label, value) in enumerate(stats_items):
                card = tk.Frame(stats_grid, bg=t["bg2"], padx=12, pady=8)
                card.grid(row=i//3, column=i%3, padx=8, pady=4, sticky="nsew")
                
                tk.Label(card, text=label, font=("Courier", 9), 
                        bg=t["bg2"], fg=t["fg2"]).pack()
                tk.Label(card, text=value, font=("Georgia", 14, "bold"), 
                        bg=t["bg2"], fg=t["accent"]).pack()
        else:
            tk.Label(self.stats_frame, text=TR("history_empty"), 
                    font=("Courier", 12), bg=t["bg3"], fg=t["fg2"]).pack(pady=20)
    
    def display_history_entries(self):
        """Отображение записей истории"""
        for w in self.history_container.winfo_children():
            w.destroy()
        
        if not self.current_history:
            t = T()
            tk.Label(self.history_container, text=TR("history_empty"), 
                    font=("Courier", 12), bg=t["bg"], fg=t["fg2"]).pack(pady=40)
            return
        
        # Расчет отображаемых записей для текущей страницы
        start_idx = (self.current_page - 1) * self.items_per_page
        end_idx = min(start_idx + self.items_per_page, len(self.current_history))
        page_entries = self.current_history[start_idx:end_idx]
        
        # Создаем canvas для скроллинга
        canvas = tk.Canvas(self.history_container, bg=T()["bg"], highlightthickness=0)
        scrollbar = ttk.Scrollbar(self.history_container, orient="vertical", command=canvas.yview)
        scrollable_frame = tk.Frame(canvas, bg=T()["bg"])
        
        scrollable_frame.bind(
            "<Configure>",
            lambda e: canvas.configure(scrollregion=canvas.bbox("all"))
        )
        
        canvas.create_window((0, 0), window=scrollable_frame, anchor="nw")
        canvas.configure(yscrollcommand=scrollbar.set)
        
        # Отображение каждой записи
        for i, entry in enumerate(page_entries):
            self.create_history_card(scrollable_frame, entry, i)
            if i < len(page_entries) - 1:
                tk.Frame(scrollable_frame, bg=T()["border"], height=1).pack(fill="x", pady=8)
        
        canvas.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")
        
        # Привязка скроллинга
        def _on_mousewheel(event):
            canvas.yview_scroll(int(-1*(event.delta/120)), "units")
        
        canvas.bind("<Enter>", lambda e: canvas.bind_all("<MouseWheel>", _on_mousewheel))
        canvas.bind("<Leave>", lambda e: canvas.unbind_all("<MouseWheel>"))
    
    def create_history_card(self, parent, entry, index):
        """Создание карточки записи истории"""
        t = T()
        sym = entry.get('currency', '₽')
        
        card = tk.Frame(parent, bg=t["bg2"], padx=16, pady=12)
        card.pack(fill="x", pady=4)
        
        # Верхняя строка с датой и кнопками
        top_row = tk.Frame(card, bg=t["bg2"])
        top_row.pack(fill="x", pady=(0, 10))
        
        tk.Label(top_row, text=f"📅 {entry['date']}", 
                font=("Courier", 11, "bold"), bg=t["bg2"], fg=t["fg"]).pack(side="left")
        
        # Кнопки действий
        btn_frame = tk.Frame(top_row, bg=t["bg2"])
        btn_frame.pack(side="right")
        
        # Кнопка копирования
        copy_text = self.format_entry_for_copy(entry, sym)
        copy_btn = tk.Button(btn_frame, text="📋 " + TR("copy"), 
                            font=("Courier", 9), bg=t["green"], fg="white",
                            bd=0, padx=8, pady=4, cursor="hand2",
                            command=lambda: self.copy_to_clipboard(copy_text))
        copy_btn.pack(side="left", padx=(0, 5))
        
        # Кнопка удаления - ИСПРАВЛЕНО!
        delete_btn = tk.Button(btn_frame, text="🗑 " + TR("delete"), 
                              font=("Courier", 9), bg=t["red"], fg="white",
                              bd=0, padx=8, pady=4, cursor="hand2",
                              command=lambda: self.delete_entry(entry['id']))
        delete_btn.pack(side="left")
        
        # Основная информация
        content = tk.Frame(card, bg=t["bg2"])
        content.pack(fill="x")
        
        # Левая колонка - информация об автомобиле
        left_col = tk.Frame(content, bg=t["bg2"])
        left_col.pack(side="left", fill="x", expand=True)
        
        tk.Label(left_col, text=f"🚗 {entry.get('car', '—')}", 
                font=("Courier", 11, "bold"), bg=t["bg2"], fg=t["accent"]).pack(anchor="w")
        
        info_text = f"📏 {TR('distance')}: {entry['distance']:.0f} км  |  ⛽ {TR('fuel')}: {entry['fuel']:.1f} л"
        tk.Label(left_col, text=info_text, font=("Courier", 10), 
                bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", pady=(4, 2))
        
        price_text = f"💰 {TR('fuel_price')}: {entry['price']:.2f} {sym}/{TR('fuel_unit')}"
        tk.Label(left_col, text=price_text, font=("Courier", 10), 
                bg=t["bg2"], fg=t["fg2"]).pack(anchor="w")
        
        # Правая колонка - результаты
        right_col = tk.Frame(content, bg=t["bg3"], padx=12, pady=8)
        right_col.pack(side="right")
        
        consumption_text = f"💧 {TR('avg_consumption')}: {entry['consumption']:.1f} л/100км"
        tk.Label(right_col, text=consumption_text, font=("Courier", 11, "bold"), 
                bg=t["bg3"], fg=t["green"]).pack(anchor="w")
        
        cost_text = f"💰 {TR('trip_cost')}: {entry['cost']:,.2f} {sym}"
        tk.Label(right_col, text=cost_text, font=("Courier", 11, "bold"), 
                bg=t["bg3"], fg=t["yellow"]).pack(anchor="w")
    
    def format_entry_for_copy(self, entry, sym):
        """Форматирование записи для копирования"""
        return (
            f"📅 Дата: {entry['date']}\n"
            f"🚗 Автомобиль: {entry.get('car', '—')}\n"
            f"📏 Расстояние: {entry['distance']:.0f} км\n"
            f"⛽ Топливо: {entry['fuel']:.1f} л\n"
            f"💰 Цена: {entry['price']:.2f} {sym}/л\n"
            f"💧 Расход: {entry['consumption']:.1f} л/100км\n"
            f"💰 Стоимость: {entry['cost']:,.2f} {sym}"
        )
    
    def copy_to_clipboard(self, text):
        """Копирование в буфер обмена"""
        self.app.clipboard_clear()
        self.app.clipboard_append(text)
        messagebox.showinfo(TR("copied"), TR("copied_msg"))
    
    def delete_entry(self, entry_id):
        """Удаление записи из истории - ИСПРАВЛЕНО!"""
        if messagebox.askyesno(TR("delete_confirm"), "Удалить эту запись?"):
            if db_delete_history_entry(self.app.current_user, entry_id):  # Правильное название функции
                self.refresh()
                messagebox.showinfo("Успех", "Запись удалена")
            else:
                messagebox.showerror(TR("error"), "Ошибка при удалении")
    
    def clear_all_history(self):
        """Очистка всей истории"""
        if messagebox.askyesno(TR("clear_history_confirm"), TR("clear_history_msg")):
            if db_clear_history(self.app.current_user):
                self.refresh()
                messagebox.showinfo("Успех", "История очищена")
            else:
                messagebox.showerror(TR("error"), "Ошибка при очистке")
    
    def display_pagination(self):
        """Отображение пагинации"""
        for w in self.pagination_frame.winfo_children():
            w.destroy()
        
        if self.total_pages <= 1:
            return
        
        t = T()
        
        # Кнопка "Первая"
        if self.current_page > 1:
            first_btn = tk.Button(self.pagination_frame, text="«", 
                                 font=("Courier", 10), bg=t["btn2"], fg=t["fg"],
                                 bd=0, padx=10, pady=4, command=self.first_page)
            first_btn.pack(side="left", padx=2)
        
        # Кнопка "Предыдущая"
        if self.current_page > 1:
            prev_btn = tk.Button(self.pagination_frame, text="‹", 
                                font=("Courier", 10), bg=t["btn2"], fg=t["fg"],
                                bd=0, padx=10, pady=4, command=self.prev_page)
            prev_btn.pack(side="left", padx=2)
        
        # Номера страниц
        start_page = max(1, self.current_page - 2)
        end_page = min(self.total_pages, start_page + 4)
        
        for page in range(start_page, end_page + 1):
            if page == self.current_page:
                btn = tk.Button(self.pagination_frame, text=str(page), 
                              font=("Courier", 10, "bold"), bg=t["btn"], fg="white",
                              bd=0, padx=12, pady=4)
            else:
                btn = tk.Button(self.pagination_frame, text=str(page), 
                              font=("Courier", 10), bg=t["btn2"], fg=t["fg"],
                              bd=0, padx=12, pady=4, 
                              command=lambda p=page: self.go_to_page(p))
            btn.pack(side="left", padx=2)
        
        # Кнопка "Следующая"
        if self.current_page < self.total_pages:
            next_btn = tk.Button(self.pagination_frame, text="›", 
                                font=("Courier", 10), bg=t["btn2"], fg=t["fg"],
                                bd=0, padx=10, pady=4, command=self.next_page)
            next_btn.pack(side="left", padx=2)
        
        # Кнопка "Последняя"
        if self.current_page < self.total_pages:
            last_btn = tk.Button(self.pagination_frame, text="»", 
                                font=("Courier", 10), bg=t["btn2"], fg=t["fg"],
                                bd=0, padx=10, pady=4, command=self.last_page)
            last_btn.pack(side="left", padx=2)
        
        # Информация о странице
        info_text = f"Страница {self.current_page} из {self.total_pages} (всего {len(self.current_history)} записей)"
        tk.Label(self.pagination_frame, text=info_text, font=("Courier", 9), 
                bg=t["bg"], fg=t["fg2"]).pack(side="right", padx=10)
    
    def first_page(self):
        """Переход на первую страницу"""
        self.current_page = 1
        self.refresh()
    
    def prev_page(self):
        """Предыдущая страница"""
        if self.current_page > 1:
            self.current_page -= 1
            self.refresh()
    
    def next_page(self):
        """Следующая страница"""
        if self.current_page < self.total_pages:
            self.current_page += 1
            self.refresh()
    
    def last_page(self):
        """Последняя страница"""
        self.current_page = self.total_pages
        self.refresh()
    
    def go_to_page(self, page):
        """Переход на определенную страницу"""
        self.current_page = page
        self.refresh()
    
    def show_login_message(self):
        """Показать сообщение о необходимости входа"""
        t = T()
        for w in self.history_container.winfo_children():
            w.destroy()
        tk.Label(self.history_container, text=TR("login_for_history"), 
                font=("Courier", 14), bg=t["bg"], fg=t["fg2"]).pack(pady=50)
