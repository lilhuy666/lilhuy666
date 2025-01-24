▎ Класс «Туристический автобус»

Вот более интересная версия реализации класса, описывающего сущность «туристический автобус» на языке Java. Я добавил несколько дополнительных методов и комментариев для улучшения функциональности и читаемости кода:

public class TouristBus {
    private int totalSeats; // Общее количество мест
    private double seatPrice; // Стоимость одного места
    private int occupiedSeats; // Количество занятых мест

    // Конструктор по умолчанию
    public TouristBus() {
        this.totalSeats = 50; // Предположим, что по умолчанию 50 мест
        this.seatPrice = 100.0; // Предположим, что стоимость одного места 100
        this.occupiedSeats = 0; // Изначально автобус пуст
    }

    // Конструктор с параметрами
    public TouristBus(int totalSeats, double seatPrice) {
        this.totalSeats = totalSeats;
        this.seatPrice = seatPrice;
        this.occupiedSeats = 0; // Изначально автобус пуст
    }

    // Конструктор копирования
    public TouristBus(TouristBus bus) {
        this.totalSeats = bus.totalSeats;
        this.seatPrice = bus.seatPrice;
        this.occupiedSeats = bus.occupiedSeats;
    }

    // Метод для изменения количества занятых мест
    public void occupySeats(int seats) {
        if (occupiedSeats + seats <= totalSeats) {
            occupiedSeats += seats;
            System.out.println(seats + " seats occupied. Total occupied: " + occupiedSeats);
        } else {
            System.out.println("Недостаточно свободных мест!");
        }
    }

    // Метод для освобождения мест
    public void freeSeats(int seats) {
        if (occupiedSeats - seats >= 0) {
            occupiedSeats -= seats;
            System.out.println(seats + " seats freed. Total occupied: " + occupiedSeats);
        } else {
            System.out.println("Невозможно освободить больше мест, чем занято!");
        }
    }

    // Метод для получения количества занятых мест
    public int getOccupiedSeats() {
        return occupiedSeats;
    }

    // Метод для получения количества свободных мест
    public int getFreeSeats() {
        return totalSeats - occupiedSeats
