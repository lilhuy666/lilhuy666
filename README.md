package com.example.fuelcalc;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.*;

import java.util.ArrayList;
import java.util.List;

@SpringBootApplication
@RestController
public class FuelCalcApplication {

    public static void main(String[] args) {
        SpringApplication.run(FuelCalcApplication.class, args);
    }

    static class Calculation {
        public double distance;
        public double consumption;
        public double price;
        public double total;
    }

    private final List<Calculation> history = new ArrayList<>();

    // 🔹 Главная страница
    @GetMapping("/")
    public String index() {
        return htmlPage("", "");
    }

    // 🔹 Расчет
    @PostMapping("/calc")
    public String calc(@RequestParam double distance,
                       @RequestParam double consumption,
                       @RequestParam double price) {

        double liters = (distance / 100) * consumption;
        double total = liters * price;

        Calculation c = new Calculation();
        c.distance = distance;
        c.consumption = consumption;
        c.price = price;
        c.total = total;
        history.add(c);

        return htmlPage("<h2>Результат: " + total + " ₽</h2>", "");
    }

    // 🔹 Профиль (история)
    @GetMapping("/profile")
    public String profile() {

        StringBuilder table = new StringBuilder();
        table.append("<h2>История</h2><table><tr><th>Км</th><th>Расход</th><th>Цена</th><th>Итого</th></tr>");

        for (Calculation c : history) {
            table.append("<tr>")
                    .append("<td>").append(c.distance).append("</td>")
                    .append("<td>").append(c.consumption).append("</td>")
                    .append("<td>").append(c.price).append("</td>")
                    .append("<td>").append(c.total).append("</td>")
                    .append("</tr>");
        }

        table.append("</table>");

        return htmlPage("", table.toString());
    }

    // 🔥 ВЕСЬ HTML + CSS ВНУТРИ
    private String htmlPage(String result, String content) {
        return """
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Fuel Calculator</title>

<style>
body {
    margin: 0;
    font-family: Arial;
    background: #0f1f14;
    color: white;
    text-align: center;
}

.container {
    margin-top: 80px;
}

h1 {
    color: #00ff88;
}

form {
    display: flex;
    flex-direction: column;
    width: 300px;
    margin: auto;
    gap: 10px;
}

input {
    padding: 10px;
    border-radius: 10px;
    border: none;
}

button {
    padding: 10px;
    border: none;
    border-radius: 10px;
    background: #00ff88;
    cursor: pointer;
}

.burger {
    position: absolute;
    top: 15px;
    left: 15px;
    font-size: 30px;
    cursor: pointer;
}

.menu {
    position: absolute;
    top: 50px;
    left: -200px;
    width: 200px;
    background: #1a3a24;
    transition: 0.3s;
    display: flex;
    flex-direction: column;
}

.menu a {
    padding: 15px;
    color: white;
    text-decoration: none;
}

.menu.active {
    left: 0;
}

table {
    margin: auto;
    border-collapse: collapse;
    margin-top: 20px;
}

td, th {
    border: 1px solid #00ff88;
    padding: 10px;
}
</style>

</head>

<body>

<div class="burger" onclick="toggleMenu()">☰</div>

<div class="menu" id="menu">
    <a href="/">Главная</a>
    <a href="/profile">Личный кабинет</a>
</div>

<div class="container">
    <h1>Калькулятор топлива</h1>

    <form method="post" action="/calc">
        <input name="distance" placeholder="Км" required>
        <input name="consumption" placeholder="л/100км" required>
        <input name="price" placeholder="Цена" required>
        <button type="submit">Рассчитать</button>
    </form>

    """ + result + content + """

</div>

<script>
function toggleMenu() {
    document.getElementById("menu").classList.toggle("active");
}
</script>

</body>
</html>
""";
    }
}
