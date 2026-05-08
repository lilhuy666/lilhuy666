Traceback (most recent call last):
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 2234, in <module>
    app = FuelApp()
          ^^^^^^^^^
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 524, in __init__
    self._build_ui()
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 575, in _build_ui
    self._build_history()
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 1568, in _build_history
    self._render_history_entries()
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 1596, in _render_history_entries
    img = load_car_image(ph, 200, 120) if ph and os.path.exists(ph) else make_placeholder(200, 120)
          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\Кирилл\PycharmProjects\PythonProject6\main.py", line 478, in load_car_image
    img = Image.open(path).convert("RGB")
          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\Кирилл\PycharmProjects\PythonProject7\.venv\Lib\site-packages\PIL\Image.py", line 1070, in convert
    self.load()
  File "C:\Users\Кирилл\PycharmProjects\PythonProject7\.venv\Lib\site-packages\PIL\WebPImagePlugin.py", line 138, in load
    return super().load()
           ^^^^^^^^^^^^^^
  File "C:\Users\Кирилл\PycharmProjects\PythonProject7\.venv\Lib\site-packages\PIL\ImageFile.py", line 412, in load
    n, err_code = decoder.decode(b)
                  ^^^^^^^^^^^^^^^^^
KeyboardInterrupt
