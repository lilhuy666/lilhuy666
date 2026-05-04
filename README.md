Traceback (most recent call last):
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 1180, in <module>
    main()
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 1158, in main
    app = FuelCalcPro()
          ^^^^^^^^^^^^^
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 283, in __init__
    self._build()
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 355, in _build
    self._show_section("calculator")
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 606, in _show_section
    self._build_calculator()
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 646, in _build_calculator
    self._build_result_card(right)
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 747, in _build_result_card
    self.gauge = Gauge(gf, size=150, C=C)
                 ^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 163, in __init__
    self._draw_base()
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 171, in _draw_base
    self.create_oval(cx-r-i*3, cy-r-i*3, cx+r+i*3, cy+r+i*3,
  File "C:\Users\Кирилл\AppData\Local\Programs\Python\Python312\Lib\tkinter\__init__.py", line 2885, in create_oval
    return self._create('oval', args, kw)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\Кирилл\AppData\Local\Programs\Python\Python312\Lib\tkinter\__init__.py", line 2863, in _create
    return self.tk.getint(self.tk.call(
                          ^^^^^^^^^^^^^
_tkinter.TclError: invalid color name "#00E5FF18"
