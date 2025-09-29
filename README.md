Конечно, вот улучшенный вариант кода на Python, который соответствует вашему запросу, оптимизирован для Visual Studio Code и включает использование декоратора `@property` для более "питонического" доступа к атрибутам.

class Dog:
    """
    Базовый класс для собак.
    Содержит общие атрибуты: цвет, порода, возраст.
    """
    def __init__(self, color: str, breed: str, age: int):
        """
        Инициализатор класса Dog.

        Args:
            color (str): Цвет собаки.
            breed (str): Порода собаки.
            age (int): Возраст собаки.
        """
        self._color = color # Используем префикс _ для обозначения "приватности"
        self._breed = breed
        self._age = age

    @property
    def color(self) -&gt; str:
        """ Свойство для получения цвета собаки. """
        return self._color

    @property
    def breed(self) -&gt; str:
        """ Свойство для получения породы собаки. """
        return self._breed

    @property
    def age(self) -&gt; int:
        """ Свойство для получения возраста собаки. """
        return self._age

    # Для демонстрации, можно добавить сеттеры, если нужно менять атрибуты после создания объекта
    @color.setter
    def color(self, value: str):
        """ Устанавливает новый цвет собаки. """
        if not isinstance(value, str) or not value:
            raise ValueError("Цвет должен быть непустой строкой.")
        self._color = value

    @age.setter
    def age(self, value: int):
        """ Устанавливает новый возраст собаки. """
        if not isinstance(value, int) or value &lt; 0:
            raise ValueError("Возраст должен быть неотрицательным целым числом.")
        self._age = value

    def __str__(self) -&gt; str:
        """ Возвращает строковое представление объекта Dog. """
        return f"Собака: Порода - {self.breed}, Цвет - {self.color}, Возраст - {self.age} лет."

class Pet(Dog):
    """
    Класс для домашних собак. Наследует от Dog.
    Добавляет атрибуты: имя, хозяин.
    """
    def __init__(self, color: str, breed: str, age: int, name: str, owner: str):
        """
        Инициализатор класса Pet.

        Args:
            color (str): Цвет собаки.
            breed (str): Порода собаки.
            age (int): Возраст собаки.
            name (str): Имя домашней собаки.
            owner (str): Имя хозяина.
        """
        super().__init__(color, breed, age) # Вызов инициализатора родительского класса
        self._name = name
        self._owner = owner

    @property
    def name(self) -&gt; str:
        """ Свойство для получения имени домашней собаки. """
        return self._name

    @name.setter
    def name(self, value: str):
        """ Устанавливает новое имя собаки. """
        if not isinstance(value, str) or not value:
            raise ValueError("Имя должно быть непустой строкой.")
        self._name = value

    @property
    def owner(self) -&gt; str:
        """ Свойство для получения имени хозяина. """
        return self._owner

    @owner.setter
    def owner(self, value: str):
        """ Устанавливает нового хозяина. """
        if not isinstance(value, str) or not value:
            raise ValueError("Имя хозяина должно быть непустой строкой.")
        self._owner = value

    def __str__(self) -&gt; str:
        """ Возвращает строковое представление объекта Pet. """
        # Используем свойства родительского класса через self.color, self.breed, self.age
        return (f"Домашняя собака: Имя - {self.name}, Хозяин - {self.owner}, "
                f"Порода - {self.breed}, Цвет - {self.color}, Возраст - {self.age} лет.")

class WildDog(Dog):
    """
    Класс для диких собак. Наследует от Dog.
    Добавляет атрибут: место жительства.
    """
    def __init__(self, color: str, breed: str, age: int, habitat: str):
        """
        Инициализатор класса WildDog.

        Args:
            color (str): Цвет собаки.
            breed (str): Порода собаки.
            age (int): Возраст
