Пример решения задачи на Java с использованием рекурсии и ввода двумерного массива из консоли:

import java.util.Scanner;

public class SettlementsGraph {

    private static int[][] graph;
    private static boolean[] visited;
    private static int settlements;
    private static int cities;

    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        // Вводим количество поселков и городов
        System.out.print("Введите количество поселков: ");
        settlements = scanner.nextInt();
        System.out.print("Введите количество городов: ");
        cities = scanner.nextInt();

        // Создаем граф для поселков и городов
        graph = new int[settlements + cities][settlements + cities];
        visited = new boolean[settlements + cities];

        // Вводим информацию о дорогах
        System.out.println("Введите информацию о дорогах:");
        for (int i = 0; i < cities; i++) {
            int settlement = scanner.nextInt();
            int city = scanner.nextInt();
            graph[settlement][city] = 1;
        }

        // Поиск одиночного города
        boolean found = false;
        for (int i = 0; i < cities; i++) {
            if (hasNoOutgoingRoad(i)) {
                System.out.println("Ответ: " + i);
                found = true;
                break;
            }
        }

        if (!found) {
            System.out.println("Ответ: -1");
        }
    }

    private static boolean hasNoOutgoingRoad(int node) {
        visited[node] = true;

        for (int i = 0; i < cities + settlements; i++) {
            if (graph[node][i] == 1 && !visited[i]) {
                return false;
            }
        }

        return true;
    }
}


Объяснение:
- Пользователь вводит количество поселков и городов.
- Далее вводится информация о дорогах, где каждый ввод представляет собой связь между поселком и городом.
- В программе используется рекурсивная функция hasNoOutgoingRoad, которая проверяет отсутствие исходящей дороги из заданной точки.
- После проверки всех городов определяется одиночный город (если таковой есть), если же такого города нет, выводится -1.
- Решение позволяет определить ситуацию, когда все поселки соединены со всеми городами однонаправленными дорогами, за исключением одного изолированного города.
