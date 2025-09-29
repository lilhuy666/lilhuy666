
class Dog:
    def __init__(self, color, breed, age):
        self._color = color
        self._breed = breed
        self._age = age

    # Свойство color
    @property
    def color(self):
        return self._color

    @color.setter
    def color(self, value):
        if not isinstance(value, str):
            raise ValueError("Цвет должен быть строкой")
        self._color = value

    # Свойство breed
    @property
    def breed(self):
        return self._breed

    @breed.setter
    def breed(self, value):
        if not isinstance(value, str):
            raise ValueError("Порода должна быть строкой")
        self._breed = value

    # Свойство age
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

    # Свойство name
    @property
    def name(self):
        return self._name

    @name.setter
    def name(self, value):
        if not isinstance(value, str):
            raise ValueError("Имя должно быть строкой")
        self._name = value

    # Свойство owner
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

    # Свойство habitat (место жительства)
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
    dom_dog = DomesticDog("черный", "дворняга", 4, "Барон", "Алексей")
    print(f"Домашняя собака: {dom_dog.name}, хозяин: {dom_dog.owner}, цвет: {dom_dog.color}, порода: {dom_dog.breed}, возраст: {dom_dog.age}")

    wild_dog = WildDog("серый", "волк", 6, "тайга")
    print(f"Дикая собака, место жительства: {wild_dog.habitat}, цвет: {wild_dog.color}, порода: {wild_dog.breed}, возраст: {wild_dog.age}")


- Класс Dog включает три метода (свойства): цвет, порода, возраст.
- Классы DomesticDog и WildDog наследуют Dog и добавляют свои характеристики:
- DomesticDog — имя собаки и её хозяина
- WildDog — место жительства

- Использованы свойства (property) с проверкой типов для удобного и безопасного доступа к атрибутам.
- Код готов для запуска в Visual Studio Code.
