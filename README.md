import java.util.Scanner;
import java.util.ArrayList;
import java.util.List;

public class Main {
    public static void main(String\[] args) {
        Scanner scanner = new Scanner(System.in);

        System.out.print("Введите количество населенных пунктов: ");
        int numberOfTowns = scanner.nextInt();

        System.out.print("Введите количество дорог: ");
        int numberOfRoads = scanner.nextInt();

        if (numberOfTowns < 2 || numberOfRoads < 2) {
            System.out.println("Ошибка: должно быть минимум 2 населенных пункта и 2 дороги.");
            return;
        }

        // Список для хранения количества дорог, выходящих из каждого пункта
        int\[] outDegrees = new int\[numberOfTowns];
        // Список для хранения количества дорог, входящих в каждый пункт
        int\[] inDegrees = new int\[numberOfTowns];

        System.out.println("Введите дороги в формате 'начало конец':");
        for (int i = 0; i < numberOfRoads; i++) {
            int start = scanner.nextInt() - 1;
            int end = scanner.nextInt() - 1;

            outDegrees\[start]++;
            inDegrees\[end]++;
        }

        List<Integer> potentialCities = new ArrayList<>();
        for (int i = 0; i < numberOfTowns; i++) {
            if (outDegrees\[i] == 0 && inDegrees\[i] > 0) {
                potentialCities.add(i + 1);
            }
        }

        if (potentialCities.size() == 1) {
            System.out.println("Город: " + potentialCities.get(0));
        } else {
            System.out.println("-1");
        }

        scanner.close();
    }
}
