def _render_history_entries(self):
    t = T()
    if not hasattr(self, '_hist_inner'):
        return
    for w in self._hist_inner.winfo_children():
        w.destroy()
    if not self.current_user:
        return
    history = self.data["users"][self.current_user].get("history", [])
    if not history:
        tk.Label(self._hist_inner, text=TR("history_empty"),
                 font=("Courier", 12), bg=t["bg"], fg=t["fg2"]).pack(pady=40)
        return

    for i, entry in enumerate(history):
        # Основная карточка записи
        card = tk.Frame(self._hist_inner, bg=t["bg2"], pady=8, padx=8)
        card.pack(fill="x", pady=6)

        # Верхняя строка с датой и кнопками
        top_row = tk.Frame(card, bg=t["bg2"])
        top_row.pack(fill="x", pady=(0, 8))
        
        # Дата слева
        tk.Label(top_row, text=f"📅 {entry['date']}",
                 font=("Courier", 10, "bold"), bg=t["bg2"], fg=t["fg"]).pack(side="left")
        
        # Кнопки справа
        btn_frame = tk.Frame(top_row, bg=t["bg2"])
        btn_frame.pack(side="right")
        
        # Текст для копирования
        sym = entry.get("currency", "₽")
        copy_text = (
            f"Дата: {entry['date']}\n"
            f"Авто: {entry.get('car', '—')}\n"
            f"Расстояние: {entry['distance']} км\n"
            f"Топливо: {entry['fuel']} л\n"
            f"Цена: {entry['price']} {sym}/л\n"
            f"Расход: {entry['consumption']} л/100км\n"
            f"Стоимость: {entry['cost']:,.2f} {sym}"
        )
        
        # Кнопка Копировать
        def copy_entry(txt=copy_text):
            self.clipboard_clear()
            self.clipboard_append(txt)
            messagebox.showinfo(TR("copied"), TR("copied_msg"))
        
        copy_btn = tk.Button(btn_frame, text="📋 Копировать", 
                            font=("Courier", 9),
                            bg=t["green"], fg="white", bd=0, 
                            padx=8, pady=4, cursor="hand2",
                            command=copy_entry)
        copy_btn.pack(side="left", padx=(0, 8))
        
        # Кнопка Удалить
        def delete_entry(idx=i):
            if messagebox.askyesno(TR("delete_confirm"), 
                                   "Вы уверены, что хотите удалить эту запись?"):
                del self.data["users"][self.current_user]["history"][idx]
                save_data(self.data)
                self._render_history_entries()
        
        delete_btn = tk.Button(btn_frame, text="🗑 Удалить", 
                              font=("Courier", 9),
                              bg=t["red"], fg="white", bd=0, 
                              padx=8, pady=4, cursor="hand2",
                              command=delete_entry)
        delete_btn.pack(side="left")
        
        # Фото автомобиля
        ph = None
        car_name = entry.get("car", "—")
        if car_name not in (TR("no_car"), "— Без авто —", "— No car —"):
            for car in self.data["users"].get(self.current_user, {}).get("cars", []):
                if car["name"] == car_name and car.get("photo"):
                    ph = car["photo"]
                    break

        # Контент записи
        content = tk.Frame(card, bg=t["bg2"])
        content.pack(fill="x")
        
        # Фото слева
        if ph or car_name not in (TR("no_car"), "— Без авто —", "— No car —"):
            img_box = tk.Frame(content, bg=t["bg2"], width=150, height=90)
            img_box.pack_propagate(False)
            img_box.pack(side="left", padx=(0, 12))
            
            if ph:
                img = load_car_image(ph, 150, 90)
                photo = ImageTk.PhotoImage(img)
                self._photo_refs[f"hist_car_{i}"] = photo
                tk.Label(img_box, image=photo, bg=t["bg2"]).place(relx=0.5, rely=0.5, anchor="center")
            else:
                placeholder = make_placeholder(150, 90, "🚗")
                photo = ImageTk.PhotoImage(placeholder)
                self._photo_refs[f"hist_car_{i}"] = photo
                tk.Label(img_box, image=photo, bg=t["bg2"]).place(relx=0.5, rely=0.5, anchor="center")
        
        # Информация справа
        info = tk.Frame(content, bg=t["bg2"])
        info.pack(side="left", fill="x", expand=True)
        
        # Автомобиль
        tk.Label(info, text=f"🚗 {car_name}",
                 font=("Courier", 11, "bold"), bg=t["bg2"], fg=t["accent"]).pack(anchor="w")
        
        # Основные данные
        data_text = f"📏 Расстояние: {entry['distance']} км"
        tk.Label(info, text=data_text,
                 font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w", pady=(4, 0))
        
        data_text = f"⛽ Топливо: {entry['fuel']} л"
        tk.Label(info, text=data_text,
                 font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w")
        
        data_text = f"💰 Цена: {entry['price']} {sym}/л"
        tk.Label(info, text=data_text,
                 font=("Courier", 10), bg=t["bg2"], fg=t["fg2"]).pack(anchor="w")
        
        # Результаты расчета
        results_frame = tk.Frame(info, bg=t["bg3"], padx=8, pady=4)
        results_frame.pack(fill="x", pady=(8, 0))
        
        result_text = f"💧 Расход: {entry['consumption']} л/100км"
        tk.Label(results_frame, text=result_text,
                 font=("Courier", 11, "bold"), bg=t["bg3"], fg=t["green"]).pack(anchor="w")
        
        result_text = f"💳 Стоимость: {entry['cost']:,.2f} {sym}"
        tk.Label(results_frame, text=result_text,
                 font=("Courier", 11, "bold"), bg=t["bg3"], fg=t["yellow"]).pack(anchor="w")
        
        # Разделитель между записями
        if i < len(history) - 1:
            separator = tk.Frame(self._hist_inner, bg=t["border"], height=1)
            separator.pack(fill="x", pady=4)

def _delete_history_entry(self, idx):
    if messagebox.askyesno(TR("delete_confirm"), "Удалить эту запись?"):
        del self.data["users"][self.current_user]["history"][idx]
        save_data(self.data)
        self._render_history_entries()

def _clear_all_history(self):
    if messagebox.askyesno(TR("clear_history_confirm"), TR("clear_history_msg")):
        self.data["users"][self.current_user]["history"] = []
        save_data(self.data)
        self._render_history_entries()
