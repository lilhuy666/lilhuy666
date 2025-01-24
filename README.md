▎ Класс «Туристический автобус»

Вот реализация класса «туристический автобус» с добавлением необходимой логики для создания объектов и ввода данных с клавиатуры:

import java.util.Scanner;

public class TouristBus {
    private int totalSeats; // Общее количество мест
    private double seatPrice; // Стоимость одного места
    private int occupiedSeats; // Занятые места

    // Конструктор по умолчанию
    public TouristBus() {
        this(50, 100.0); // Значения по умолчанию
    }

    // Конструктор с параметрами
    public TouristBus(int totalSeats, double seatPrice) {
        this.totalSeats = totalSeats;
        this.seatPrice = seatPrice;
        this.occupiedSeats = 0; // Автобус изначально пуст
    }

    // Конструктор копирования
    public TouristBus(TouristBus bus) {
        this.totalSeats = bus.totalSeats;
        this.seatPrice = bus.seatPrice;
        this.occupiedSeats = bus.occupiedSeats;
    }

    // Метод для занятия мест
    public void occupySeats(int seats) {
        if (occupiedSeats + seats <= totalSeats) {
            occupiedSeats += seats;
        }
    }

    // Метод для получения занятых мест
    public int getOccupiedSeats() {
        return occupiedSeats;
    }

    // Метод для получения свободных мест
    public int getFreeSeats() {
        return totalSeats - occupiedSeats;
    }

    // Метод для проверки, пуст ли автобус
    public boolean isEmpty() {
        return occupiedSeats == 0;
    }

    // Метод для проверки, заполнен ли автобус
    public boolean isFull() {
        return occupiedSeats == totalSeats;
    }

    // Метод для расчета общей стоимости занятых мест
    public double calculateTotalCost() {
        return occupiedSeats * seatPrice;
    }

    // Метод для установки количества занятых мест
    public void setOccupiedSeats(int seats) {
        if (seats <= totalSeats) {
            occupiedSeats = seats;
        }
    }

    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        // Создаем два объекта «Туристический автобус»
        TouristBus bus1 = new TouristBus(50, 500);
        TouristBus bus2 = new TouristBus(50, 600);

        // Вводим количество людей для каждого автобуса
        System.out.print("Введите количество людей для первого автобуса: ");
        int numPeople1 = scanner.nextInt();
        bus1.occupySeats(numPeople1);

        System.out.print("Введите количество людей для второго автобуса: ");
        int numPeople2 = scanner.nextInt();
        bus2.occupySeats(numPeople2);

        // Выводим количество оставшихся свободных мест и цену за поездку
        System.out.println("Свободные места в первом автобусе: " + bus1.getFreeSeats());
        System.out.println("Цена за поездку для первой группы: " + bus1.calculateTotalCost());

        System.out.println("Свободные места во втором автобусе: " + bus2.getFreeSeats());
        System.out.println("Цена за поездку для второй группы: " + bus2.calculateTotalCost());

        scanner.close();
    }
}


В этой реализации:

1. Создан класс TouristBus с необходимыми конструкторами и методами.
2. В методе main() создаются два объекта автобусов с заданными ценами.
3. Считывается количество людей для каждого автобуса.
4. Выводится количество свободных мест и стоимость поездки для групп туристов.
