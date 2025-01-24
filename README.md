Вот пример реализации класса "Туристический автобус" на Java, а также основной программы для работы с этим классом в среде Java IDEA.

▎ Класс TouristBus

public class TouristBus {
    private int totalSeats;        // Общее количество мест
    private double seatPrice;      // Стоимость одного места
    private int occupiedSeats;      // Количество занятых мест

    // Конструктор по умолчанию
    public TouristBus() {
        this.totalSeats = 50;      // Предположим, что общее количество мест по умолчанию 50
        this.seatPrice = 0;
        this.occupiedSeats = 0;
    }

    // Конструктор с параметрами
    public TouristBus(int totalSeats, double seatPrice) {
        this.totalSeats = totalSeats;
        this.seatPrice = seatPrice;
        this.occupiedSeats = 0;
    }

    // Конструктор копирования
    public TouristBus(TouristBus other) {
        this.totalSeats = other.totalSeats;
        this.seatPrice = other.seatPrice;
        this.occupiedSeats = other.occupiedSeats;
    }

    // Метод для изменения количества занятых мест
    public void occupySeats(int numberOfPeople) {
        if (numberOfPeople + occupiedSeats <= totalSeats) {
            occupiedSeats += numberOfPeople;
        } else {
            System.out.println("Недостаточно свободных мест!");
        }
    }

    // Метод для получения количества свободных мест
    public int getAvailableSeats() {
        return totalSeats - occupiedSeats;
    }

    // Метод для проверки, пуст ли или заполнен автобус
    public boolean isEmpty() {
        return occupiedSeats == 0;
    }

    public boolean isFull() {
        return occupiedSeats == totalSeats;
    }

    // Метод для расчета общей стоимости занятых мест
    public double calculateTotalCost() {
        return occupiedSeats * seatPrice;
    }

    // Геттеры для получения значений
    public double getSeatPrice() {
        return seatPrice;
    }
}


▎ Основной класс

import java.util.Scanner;

public class Main {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        // Создание объектов "Туристический автобус"
        TouristBus bus1 = new TouristBus(50, 500); // 50 мест, цена 500
        TouristBus bus2 = new TouristBus(50, 600); // 50 мест, цена 600

        // Ввод количества людей для первого автобуса
        System.out.print("Введите количество людей для первого автобуса: ");
        int peopleBus1 = scanner.nextInt();
        bus1.occupySeats(peopleBus1);

        // Ввод количества людей для второго автобуса
        System.out.print("Введите количество людей для второго автобуса: ");
        int peopleBus2 = scanner.nextInt();
        bus2.occupySeats(peopleBus2);

        // Вывод информации о количестве свободных мест и ценах
        int availableSeatsBus1 = bus1.getAvailableSeats();
        int availableSeatsBus2 = bus2.getAvailableSeats();

        System.out.println("Свободные места в первом автобусе: " + availableSeatsBus1);
        System.out.println("Свободные места во втором автобусе: " + availableSeatsBus2);
        
        double totalCostBus1 = bus1.calculateTotalCost();
        double totalCostBus2 = bus2.calculateTotalCost();

        System.out.println("Стоимость поездки для группы в первом автобусе: " + totalCostBus1);
        System.out.println("Стоимость поездки для группы во втором автобусе: " + totalCostBus2);

        scanner.close();
    }
}


▎ Объяснение работы программы

1. Класс TouristBus: описывает автобус с необходимыми полями и методами. 
   - Конструкторы позволяют создать экземпляры автобусов с разными параметрами. 
   - Метод occupySeats позволяет занять определенное количество мест. 
   - Метод getAvailableSeats возвращает количество свободных мест.
   - Методы isEmpty и isFull проверяют, пуст ли или заполнен автобус.
   - Метод calculateTotalCost рассчитывает общую стоимость занятых мест.

2. Основной класс Main: 
   - Создает два объекта автобус.
   - Запрашивает у пользователя количество людей для каждого автобуса.
