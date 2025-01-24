 int numPeople1 = -1;
        while (numPeople1 > 50)
        while (numPeople1 < 0) {
            System.out.print("kollichestvo lud 1 avtobusa: ");
            try {
                numPeople1 = Integer.parseInt(scanner.nextLine());
                if (numPeople1 > 50)
                if (numPeople1 < 0) {
                    System.out.println("ne mojet bit otricatelnim");
                }
            } catch (NumberFormatException e) {
                System.out.println("Oshibka!!!");
            }
        }
        bus1.occupySeats(numPeople1);

        // Вводим количество людей для второго автобуса
        int numPeople2 = -1;
        while (numPeople2 < 0) {
            System.out.print("kollichestvo lud 1 avtobusa: ");
            try {
                numPeople2 = Integer.parseInt(scanner.nextLine());
                if (numPeople2 < 0) {
                    System.out.println("ne mojet bit otricatelnim");
                }
            } catch (NumberFormatException e) {
                System.out.println("Oshibka!!!");
            }
        }
        bus2.occupySeats(numPeople2);
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
        while (numPeople1 < 0) {
            System.out.print("Введите количество людей для первого автобуса: ");
            try {
                numPeople1 = Integer.parseInt(scanner.nextLine());
                if (numPeople1 < 0) {
                    System.out.println("Количество людей не может быть отрицательным. Пожалуйста, попробуйте снова.");
                }
            } catch (NumberFormatException e) {
                System.out.println("Ошибка: введите целое число.");
            }
        }
        bus1.occupySeats(numPeople1);

        // Вводим количество людей для второго автобуса
        int numPeople2 = -1;
        while (numPeople2 < 0) {
            System.out.print("Введите количество людей для второго автобуса: ");
            try {
                numPeople2 = Integer.parseInt(scanner.nextLine());
                if (numPeople2 < 0) {
                    System.out.println("Количество людей не может быть отрицательным. Пожалуйста, попробуйте снова.");
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
        System.out.println("Цена за поездку для второй группы: " + bus2.calculateTotalCost());

        scanner.close();
    }
}


▎ Изменения в коде

1. Добавлены циклы while для повторного запроса ввода, если введены некорректные данные.
2. Используется try-catch блок для обработки исключений, выбрасываемых при неверном формате ввода (например, когда вводятся буквы).
