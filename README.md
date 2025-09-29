
class Dog:
    def __init__(self, color, breed, age):
        self._color = color
        self._breed = breed
        self._age = age

    @property
    def color(self):
        return self._color

    @color.setter
    def color(self, value):
        if not isinstance(value, str):
            raise ValueError("Цвет должен быть строкой")
        self._color = value

    @property
    def breed(self):
        return self._breed

    @breed.setter
    def breed(self, value):
        if not isinstance(value, str):
            raise ValueError("Порода должна быть строкой")
        self._breed = value

    @property
    def age(self):
        return self._age

    @age.setter
    def age(self, value):
        if not isinstance(value, int) or value < 0:
            raise ValueError("Возраст должен быть неотрицательным целым числом")
        self._age = value

class DomesticDog(Dog):
    def __init__(self, color, breed, age, name, owner):
        super().__init__(color, breed, age)
        self._name = name
        self._owner = owner

    @property
    def name(self):
        return self._name

    @name.setter
    def name(self, value):
        if not isinstance(value, str):
            raise ValueError("Имя должно быть строкой")
        self._name = value

    @property
    def owner(self):
        return self._owner

    @owner.setter
    def owner(self, value):
        if not isinstance(value, str):
            raise ValueError("Имя хозяина должно быть строкой")
        self._owner = value

class WildDog(Dog):
    def __init__(self, color, breed, age, habitat):
        super().__init__(color, breed, age)
        self._habitat = habitat

    @property
    def habitat(self):
        return self._habitat

    @habitat.setter
    def habitat(self, value):
        if not isinstance(value, str):
            raise ValueError("Место жительства должно быть строкой")
        self._habitat = value

# Пример использования
if __name__ == "__main__":
    domestic = DomesticDog("коричневый", "лайка", 5, "Рекс", "Иван")
    print(domestic.name, domestic.owner, domestic.color, domestic.breed, domestic.age)

    wild = WildDog("серый", "волкособ", 3, "лес")
    print(wild.habitat, wild.color, wild.breed, wild.age)


- В коде 3 класса: Dog (базовый), DomesticDog (домашняя собака) и WildDog (дикая собака)
- Добавлены свойства (property) для всех полей с валидацией
- Классы удобно использовать и расширять в VS Code или любом другом IDE
- В конце приведён пример создания объектов и доступа к свойствам
