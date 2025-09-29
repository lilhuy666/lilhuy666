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
        self._color = value

    @property
    def breed(self):
        return self._breed

    @breed.setter
    def breed(self, value):
        self._breed = value

    @property
    def age(self):
        return self._age

    @age.setter
    def age(self, value):
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
        self._name = value

    @property
    def owner(self):
        return self._owner

    @owner.setter
    def owner(self, value):
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
        self._habitat = value

# Пример использования
domestic_dog = DomesticDog("brown", "Labrador", 5, "Buddy", "Alice")
wild_dog = WildDog("gray", "Wolf", 3, "Forests")

print(f"Domestic Dog - Name: {domestic_dog.name}, Owner: {domestic_dog.owner}, Color: {domestic_dog.color}, Breed: {domestic_dog.breed}, Age: {domestic_dog.age}")
print(f"Wild Dog - Habitat: {wild_dog.habitat}, Color: {wild_dog.color}, Breed: {wild_dog.breed}, Age: {wild_dog.age}")
