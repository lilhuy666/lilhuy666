 ▎ Ограничение ввода

Чтобы ограничить пользователя вводить количество мест не больше 50, необходимо добавить проверку после считывания данных. Если введенное число превышает 50, программа должна сообщать об ошибке и запрашивать ввод снова. Вот обновленный код:

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

    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        // Создаем два объекта «Туристический автобус»
        TouristBus bus1 = new TouristBus(50, 500);
        TouristBus bus2 = new TouristBus(50, 600);

        // Вводим количество людей для первого автобуса
        int numPeople1 = -1;
        while (numPeople1 < 0 || numPeople1 > 50) {
            System.out.print("Введите количество людей для первого автобуса (макс. 50): ");
            try {
                numPeople1 = Integer.parseInt(scanner.nextLine());
                if (numPeople1 < 0) {
                    System.out.println("Количество людей не может быть отрицательным. Пожалуйста, попробуйте снова.");
                } else if (numPeople1 > 50) {
                    System.out.println("Количество людей не может превышать 50. Пожалуйста, попробуйте снова.");
                }
            } catch (NumberFormatException e) {
                System.out.println("Ошибка: введите целое число.");
            }
        }
        bus1.occupySeats(numPeople1);

        // Вводим количество людей для второго автобуса
        int numPeople2 = -1;
        while (numPeople2 < 0 || numPeople2 > 50) {
            System.out.print("Введите количество людей для второго автобуса (макс. 50): ");
            try {
                numPeople2 = Integer.parseInt(scanner.nextLine());
                if (numPeople2 < 0) {
                    System.out.println("Количество людей не может быть отрицательным. Пожалуйста, попробуйте снова.");
                } else if (numPeople2 > 50) {
                    System.out.println("Количество людей не может превышать 50. Пожалуйста, попробуйте снова.");
                }
            } catch (NumberFormatException e) {
                System.out.println("Ошибка: введите целое число.");
            }
        }
        bus2.occupySeats(numPeople2);

        // Выводим количество оставшихся свободных мест и цену за поездку
        System.out.println("Свободные места в первом автобусе: " + bus1.getFreeSeats());
        System.out.println("Цена за поездку для первой группы: " + bus1.calculateTotalCost());

        System.out.println("Свободные места во втором автобусе: " + bus2.getFreeSeats());
