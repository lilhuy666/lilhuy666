
class Dog:
    def __init__(self, color, breed, age):
        self.color = color
        self.breed = breed
        self.age = age

    def get_color(self):
        return self.color

    def get_breed(self):
        return self.breed

    def get_age(self):
        return self.age

class DomesticDog(Dog):
    def __init__(self, color, breed, age, name, owner):
        super().__init__(color, breed, age)
        self.name = name
        self.owner = owner

    def get_name(self):
        return self.name

    def get_owner(self):
        return self.owner

class WildDog(Dog):
    def __init__(self, color, breed, age, habitat):
        super().__init__(color, breed, age)
        self.habitat = habitat

    def get_habitat(self):
        return self.habitat

# Пример использования
domestic = DomesticDog("черный", "овчарка", 5, "Шарик", "Иван")
wild = WildDog("рыжий", "дворняга", 3, "лес")

print(domestic.get_name(), domestic.get_owner(), domestic.get_color(), domestic.get_breed(), domestic.get_age())
print(wild.get_habitat(), wild.get_color(), wild.get_breed(), wild.get_age())
