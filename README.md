header = tk.Frame(main, bg=PANEL, height=60)
header.pack(fill="x")
header.pack_propagate(False)

tk.Button(header,
          text="☰",
          bg=PANEL,
          fg=TEXT,
          font=("Arial", 16, "bold"),
          bd=0,
          command=toggle_menu).pack(side="left", padx=15)

title = tk.Label(header,
                 text="CalculatCar Pro 🚗",
                 bg=PANEL,
                 fg="white",
                 font=("Arial", 18, "bold"))
title.pack(side="left")

user_label = tk.Label(header,
                      text="",
                      bg=PANEL,
                      fg=SUB,
                      font=("Arial", 11))
user_label.pack(side="right", padx=15)
