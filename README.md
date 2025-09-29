class Dog:
    def __init__(self, color, breed, age):
        self.__color = color
        self.__breed = breed
        self.__age = age

    @property
    def color(self):
        return self.__color

    @property
    def breed(self):
        return self.__breed

    @property
    def age(self):
        return self.__age

    @age.setter
    def age(self, value):
        if value >= 0:
            self.__age = value
        else:
            raise ValueError("Age cannot be negative")

class WildDog(Dog):
    def __init__(self, color, breed, age, sleep_address):
        super().__init__(color, breed, age)
        self.__sleep_address = sleep_address

    @property
    def sleep_address(self):
        return self.__sleep_address

class DomesticDog(Dog):
    def __init__(self, color, breed, age, owner, name):
        super().__init__(color, breed, age)
        self.__owner = owner
        self.__name = name

    @property
    def owner(self):
        return self.__owner

    @owner.setter
    def owner(self, value):
        self.__owner = value

    @property
    def name(self):
        return self.__name

if __name__ == "__main__":
    wild_dog = WildDog("white", "wolf", 4, "forest")
    domestic_dog = DomesticDog("black", "chihuahua", 3, "Leha Stena", "Molly")

    # Вывод информации о дикой собаке
    print("Wild Dog Info:")
    print(f"Breed: {wild_dog.breed}")
    print(f"Color: {wild_dog.color}")
    print(f"Age: {wild_dog.age} years")
    print(f"Sleep Address: {wild_dog.sleep_address}")

    print()

    # Вывод информации о домашней собаке
    print("Domestic Dog Info:")
    print(f"Breed: {domestic_dog.breed}")
    print(f"Color: {domestic_dog.color}")
    print(f"Age: {domestic_dog.age} years")
    print(f"Name: {domestic_dog.name}")
    print(f"Owner: {domestic_dog.owner}")
