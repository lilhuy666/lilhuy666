▎ Класс «Туристический автобус»

Вот сокращенная версия класса «туристический автобус», который включает все указанные вами характеристики и методы:

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
}


Этот класс содержит все необходимые конструкторы и методы для работы с туристическим автобусом.
