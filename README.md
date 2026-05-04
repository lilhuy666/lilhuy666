Traceback (most recent call last):
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 1178, in <module>
    main()
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 1156, in main
    app = FuelCalcPro()
          ^^^^^^^^^^^^^
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 281, in __init__
    self._build()
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 353, in _build
    self._show_section("calculator")
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 604, in _show_section
    self._build_calculator()
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 644, in _build_calculator
    self._build_result_card(right)
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 745, in _build_result_card
    self.gauge = Gauge(gf, size=150, C=C)
                 ^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 161, in __init__
    self._draw_base()
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 169, in _draw_base
    self.create_oval(cx-r-i*3, cy-r-i*3, cx+r+i*3, cy+r+i*3,
  File "C:\Users\Кирилл\AppData\Local\Programs\Python\Python312\Lib\tkinter\__init__.py", line 2885, in create_oval
    return self._create('oval', args, kw)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\Кирилл\AppData\Local\Programs\Python\Python312\Lib\tkinter\__init__.py", line 2863, in _create
    return self.tk.getint(self.tk.call(
                          ^^^^^^^^^^^^^
_tkinter.TclError: invalid color name "#00E5FF18"
