import java.util.Scanner;

public class FuelCalculator {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        System.out.println("Калькулятор расхода топлива");

        // Ввод данных
        System.out.print("Введите пройденное расстояние (км): ");
        double distance = scanner.nextDouble();

        System.out.print("Введите израсходованное топливо (литры): ");
        double fuel = scanner.nextDouble();

        // Проверка
        if (distance <= 0 || fuel <= 0) {
            System.out.println("Ошибка: значения должны быть больше 0");
            return;
        }

        // Расчет (литры на 100 км)
        double consumption = (fuel / distance) * 100;

        // Вывод результата
        System.out.printf("Расход топлива: %.2f л/100 км\n", consumption);

        scanner.close();
    }
}
