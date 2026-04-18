Traceback (most recent call last):
  File "C:\Users\Кирилл\PycharmProjects\PythonProject3\.venv\Lib\site-packages\numpy\_core\__init__.py", line 24, in <module>
    from . import multiarray
  File "C:\Users\Кирилл\PycharmProjects\PythonProject3\.venv\Lib\site-packages\numpy\_core\multiarray.py", line 11, in <module>
    from . import _multiarray_umath, overrides
ImportError: DLL load failed while importing _multiarray_umath: Произошел сбой в программе инициализации библиотеки динамической компоновки (DLL).

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "C:\Users\Кирилл\PycharmProjects\PythonProject3\main.py", line 6, in <module>
    import matplotlib.pyplot as plt
  File "C:\Users\Кирилл\PycharmProjects\PythonProject3\.venv\Lib\site-packages\matplotlib\__init__.py", line 161, in <module>
    from . import _api, _version, cbook, _docstring, rcsetup
  File "C:\Users\Кирилл\PycharmProjects\PythonProject3\.venv\Lib\site-packages\matplotlib\cbook.py", line 24, in <module>
    import numpy as np
  File "C:\Users\Кирилл\PycharmProjects\PythonProject3\.venv\Lib\site-packages\numpy\__init__.py", line 125, in <module>
    from numpy.__config__ import show_config
  File "C:\Users\Кирилл\PycharmProjects\PythonProject3\.venv\Lib\site-packages\numpy\__config__.py", line 4, in <module>
    from numpy._core._multiarray_umath import (
  File "C:\Users\Кирилл\PycharmProjects\PythonProject3\.venv\Lib\site-packages\numpy\_core\__init__.py", line 85, in <module>
    raise ImportError(msg) from exc
ImportError: 

IMPORTANT: PLEASE READ THIS FOR ADVICE ON HOW TO SOLVE THIS ISSUE!

Importing the numpy C-extensions failed. This error can happen for
many reasons, often due to issues with your setup or how NumPy was
installed.

We have compiled some common reasons and troubleshooting tips at:

    https://numpy.org/devdocs/user/troubleshooting-importerror.html

Please note and check the following:

  * The Python version is: Python 3.12 from "C:\Users\Кирилл\PycharmProjects\PythonProject3\.venv\Scripts\python.exe"
  * The NumPy version is: "2.4.4"

and make sure that they are the versions you expect.

Please carefully study the information and documentation linked above.
This is unlikely to be a NumPy issue but will be caused by a bad install
or environment on your machine.

Original error was: DLL load failed while importing _multiarray_umath: Произошел сбой в программе инициализации библиотеки динамической компоновки (DLL).

