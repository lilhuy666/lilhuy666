Пример программы на Java, соответствующей вашим требованиям:

import java.util.*;

public class TownsFinder {

    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        // Ввод количества населенных пунктов и дорог
        System.out.print("Введите количество населенных пунктов: ");
        int towns = scanner.nextInt();
        System.out.print("Введите количество дорог: ");
        int roads = scanner.nextInt();

        List<Integer> destinationTowns = new ArrayList<>();
        int[] outDegrees = new int[towns + 1];

        // Ввод информации о дорогах
        for (int i = 0; i < roads; i++) {
            int start = scanner.nextInt();
            int end = scanner.nextInt();
            outDegrees[start]++;
            destinationTowns.add(end);
        }

        List<Integer> possibleTowns = new ArrayList<>();
        for (int i = 1; i <= towns; i++) {
            if (outDegrees[i] == 0 && destinationTowns.contains(i)) {
                possibleTowns.add(i);
            }
        }

        // Вывод результата
        if (possibleTowns.isEmpty()) {
            System.out.println("Ответ: -1");
        } else if (possibleTowns.size() == 1) {
            System.out.println("Ответ: " + possibleTowns.get(0));
        } else {
            System.out.println("Ответ: -1");
        }
    }
}


Объяснение:
- В программе пользователь вводит количество населенных пунктов и дорог.
- Затем пользователь вводит информацию о дорогах (начало и конец дороги).
- Для каждого населенного пункта вычисляется количество исходящих дорог.
- Затем программа ищет города, из которых нет исходящих дорог и которые являются конечными пунктами некоторых дорог.
- Если таких городов нет, программа выводит -1.
- Если найден только один такой город, программа выводит его номер.
- Если найдено несколько таких городов, программа также выводит -1.
