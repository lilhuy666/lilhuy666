import com.sun.net.httpserver.HttpServer;
import com.sun.net.httpserver.HttpExchange;

import java.io.*;
import java.net.InetSocketAddress;
import java.nio.file.*;
import java.util.*;

public class Main {

    static final String FILE = "history.txt";

    public static void main(String[] args) throws Exception {
        HttpServer server = HttpServer.create(new InetSocketAddress(8080), 0);

        server.createContext("/", Main::index);
        server.createContext("/calculate", Main::calculate);
        server.createContext("/history", Main::history);
        server.createContext("/profile", Main::profile);

        server.setExecutor(null);
        server.start();

        System.out.println("http://localhost:8080");
    }

    static void index(HttpExchange ex) throws IOException {
        send(ex, htmlPage("", ""));
    }

    static void calculate(HttpExchange ex) throws IOException {
        if ("POST".equals(ex.getRequestMethod())) {
            String body = new String(ex.getRequestBody().readAllBytes());
            Map<String, String> data = parse(body);

            double d = Double.parseDouble(data.get("distance"));
            double f = Double.parseDouble(data.get("fuel"));

            double result = (f / d) * 100;

            String record = d + " км | " + f + " л | " + result + " л/100км\n";
            Files.writeString(Paths.get(FILE), record, StandardOpenOption.CREATE, StandardOpenOption.APPEND);

            send(ex, htmlPage("<h2>Расход: " + result + " л/100км</h2>", ""));
        }
    }

    static void history(HttpExchange ex) throws IOException {
        String content = "";

        if (Files.exists(Paths.get(FILE))) {
            content = Files.readString(Paths.get(FILE)).replace("\n", "<br>");
        }

        send(ex, htmlPage("<h2>История</h2><div class='card'>" + content + "</div>", ""));
    }

    static void profile(HttpExchange ex) throws IOException {
        String profile = """
        <h2>Личный кабинет</h2>
        <div class='card'>
            <p><b>Имя:</b> Студент</p>
            <p><b>Email:</b> student@mail.com</p>
            <p><b>Проект:</b> Калькулятор расхода топлива</p>
        </div>
        """;

        send(ex, htmlPage(profile, ""));
    }

    static Map<String, String> parse(String body) {
        Map<String, String> map = new HashMap<>();
        for (String p : body.split("&")) {
            String[] kv = p.split("=");
            map.put(kv[0], kv[1]);
        }
        return map;
    }

    static void send(HttpExchange ex, String response) throws IOException {
        ex.sendResponseHeaders(200, response.getBytes().length);
        OutputStream os = ex.getResponseBody();
        os.write(response.getBytes());
        os.close();
    }

    static String htmlPage(String dynamic, String script) {
        return """
        <html>
        <head>
        <meta charset="UTF-8">
        <title>Fuel App</title>

        <style>
        body {
            margin:0;
            font-family: Arial;
            background: linear-gradient(135deg,#1e3c72,#2a5298);
            color:white;
        }

        .menu {
            display:flex;
            justify-content:center;
            gap:20px;
            padding:15px;
            background: rgba(0,0,0,0.3);
        }

        .menu a {
            color:white;
            text-decoration:none;
            font-weight:bold;
        }

        .container {
            text-align:center;
            margin-top:50px;
        }

        .card {
            background: rgba(255,255,255,0.1);
            padding:20px;
            margin:20px auto;
            width:300px;
            border-radius:15px;
            backdrop-filter: blur(10px);
        }

        input {
            padding:10px;
            margin:5px;
            width:200px;
            border:none;
            border-radius:5px;
        }

        button {
            padding:10px 20px;
            background:#00c6ff;
            border:none;
            border-radius:5px;
            color:white;
            cursor:pointer;
        }
        </style>

        </head>

        <body>

        <div class="menu">
            <a href="/">Главная</a>
            <a href="/history">История</a>
            <a href="/profile">Профиль</a>
        </div>

        <div class="container">
            <h1>Калькулятор расхода топлива</h1>

            <form method="post" action="/calculate">
                <input name="distance" placeholder="Км"><br>
                <input name="fuel" placeholder="Литры"><br>
                <button>Рассчитать</button>
            </form>

            """ + dynamic + """

        </div>

        </body>
        </html>
        """;
    }
}
