header = tk.Frame(main, bg=ACCENT, height=70)
header.pack(fill="x")
header.pack_propagate(False)

tk.Button(header,
          text="☰",
          bg=ACCENT,
          fg="white",
          font=("Arial", 16, "bold"),
          bd=0,
          activebackground=ACCENT,
          activeforeground="white",
          command=toggle_menu).pack(side="left", padx=15)

title = tk.Label(header,
                 text="CalculatCar Pro 🚗",
                 bg=ACCENT,
                 fg="white",
                 font=("Arial", 20, "bold"))
title.pack(side="left")

user_label = tk.Label(header,
                      text="",
                      bg=ACCENT,
                      fg="white",
                      font=("Arial", 11))
user_label.pack(side="right", padx=15)
