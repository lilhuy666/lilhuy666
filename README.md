import javax.swing.*;
import javax.swing.table.DefaultTableModel;
import java.awt.*;
import java.awt.event.*;
import java.sql.*;
import java.util.ArrayList;
import org.jfree.chart.ChartFactory;
import org.jfree.chart.ChartPanel;
import org.jfree.chart.JFreeChart;
import org.jfree.data.category.DefaultCategoryDataset;

public class FuelApp {

    static Connection conn;
    static String currentUser = null;

    public static void main(String[] args) {
        connectDB();
        createTables();
        new LoginWindow();
    }

    // ---------------- БАЗА ----------------
    static void connectDB() {
        try {
            conn = DriverManager.getConnection("jdbc:sqlite:fuel_app.db");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    static void createTables() {
        try {
            Statement st = conn.createStatement();

            st.execute("""
                CREATE TABLE IF NOT EXISTS users (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    username TEXT,
                    password TEXT
                )
            """);

            st.execute("""
                CREATE TABLE IF NOT EXISTS history (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    user TEXT,
                    transport TEXT,
                    distance REAL,
                    fuel REAL,
                    price REAL,
                    result REAL,
                    cost REAL
                )
            """);

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // ---------------- LOGIN ----------------
    static class LoginWindow extends JFrame {

        JTextField loginField = new JTextField(15);
        JPasswordField passField = new JPasswordField(15);

        LoginWindow() {
            setTitle("Вход");
            setSize(300, 200);
            setLayout(new FlowLayout());

            add(new JLabel("Логин"));
            add(loginField);
            add(new JLabel("Пароль"));
            add(passField);

            JButton loginBtn = new JButton("Войти");
            JButton regBtn = new JButton("Регистрация");

            add(loginBtn);
            add(regBtn);

            loginBtn.addActionListener(e -> login());
            regBtn.addActionListener(e -> register());

            setDefaultCloseOperation(EXIT_ON_CLOSE);
            setVisible(true);
        }

        void register() {
            try {
                PreparedStatement ps = conn.prepareStatement(
                        "SELECT * FROM users WHERE username=?");
                ps.setString(1, loginField.getText());

                if (ps.executeQuery().next()) {
                    JOptionPane.showMessageDialog(this, "Пользователь существует");
                    return;
                }

                ps = conn.prepareStatement(
                        "INSERT INTO users(username, password) VALUES (?, ?)");
                ps.setString(1, loginField.getText());
                ps.setString(2, new String(passField.getPassword()));
                ps.executeUpdate();

                JOptionPane.showMessageDialog(this, "Успех");

            } catch (Exception ex) {
                ex.printStackTrace();
            }
        }

        void login() {
            try {
                PreparedStatement ps = conn.prepareStatement(
                        "SELECT * FROM users WHERE username=? AND password=?");

                ps.setString(1, loginField.getText());
                ps.setString(2, new String(passField.getPassword()));

                if (ps.executeQuery().next()) {
                    currentUser = loginField.getText();
                    new MainWindow();
                    dispose();
                } else {
                    JOptionPane.showMessageDialog(this, "Ошибка входа");
                }

            } catch (Exception ex) {
                ex.printStackTrace();
            }
        }
    }

    // ---------------- MAIN ----------------
    static class MainWindow extends JFrame {

        JTextField distanceField = new JTextField(10);
        JTextField fuelField = new JTextField(10);
        JTextField priceField = new JTextField(10);

        JComboBox<String> transportBox =
                new JComboBox<>(new String[]{"Легковой", "Грузовой", "Мотоцикл"});

        JLabel resultLabel = new JLabel("");

        MainWindow() {
            setTitle("Fuel Analytics");
            setSize(400, 350);
            setLayout(new FlowLayout());

            add(new JLabel("Пользователь: " + currentUser));

            add(new JLabel("Транспорт"));
            add(transportBox);

            add(new JLabel("Дистанция"));
            add(distanceField);

            add(new JLabel("Топливо"));
            add(fuelField);

            add(new JLabel("Цена"));
            add(priceField);

            JButton calcBtn = new JButton("Рассчитать");
            JButton histBtn = new JButton("История");
            JButton analBtn = new JButton("Аналитика");

            add(calcBtn);
            add(histBtn);
            add(analBtn);
            add(resultLabel);

            calcBtn.addActionListener(e -> calculate());
            histBtn.addActionListener(e -> showHistory());
            analBtn.addActionListener(e -> showAnalytics());

            setDefaultCloseOperation(EXIT_ON_CLOSE);
            setVisible(true);
        }

        void calculate() {
            try {
                double distance = Double.parseDouble(distanceField.getText());
                double fuel = Double.parseDouble(fuelField.getText());
                double price = Double.parseDouble(priceField.getText());

                double consumption = (fuel / distance) * 100;
                double cost = fuel * price;

                resultLabel.setText(
                        String.format("%.2f л/100км | %.2f руб", consumption, cost));

                PreparedStatement ps = conn.prepareStatement("""
                        INSERT INTO history(user, transport, distance, fuel, price, result, cost)
                        VALUES (?, ?, ?, ?, ?, ?, ?)
                """);

                ps.setString(1, currentUser);
                ps.setString(2, transportBox.getSelectedItem().toString());
                ps.setDouble(3, distance);
                ps.setDouble(4, fuel);
                ps.setDouble(5, price);
                ps.setDouble(6, consumption);
                ps.setDouble(7, cost);

                ps.executeUpdate();

            } catch (Exception ex) {
                JOptionPane.showMessageDialog(this, "Ошибка ввода");
            }
        }

        void showHistory() {
            JFrame win = new JFrame("История");
            win.setSize(400, 300);

            DefaultTableModel model = new DefaultTableModel();
            model.addColumn("Тип");
            model.addColumn("Расход");
            model.addColumn("Стоимость");

            JTable table = new JTable(model);

            try {
                PreparedStatement ps = conn.prepareStatement(
                        "SELECT transport, result, cost FROM history WHERE user=?");
                ps.setString(1, currentUser);

                ResultSet rs = ps.executeQuery();
                while (rs.next()) {
                    model.addRow(new Object[]{
                            rs.getString(1),
                            rs.getDouble(2),
                            rs.getDouble(3)
                    });
                }

            } catch (Exception e) {
                e.printStackTrace();
            }

            win.add(new JScrollPane(table));
            win.setVisible(true);
        }

        void showAnalytics() {
            try {
                PreparedStatement ps = conn.prepareStatement(
                        "SELECT result FROM history WHERE user=?");
                ps.setString(1, currentUser);

                ResultSet rs = ps.executeQuery();

                ArrayList<Double> values = new ArrayList<>();

                while (rs.next()) {
                    values.add(rs.getDouble(1));
                }

                if (values.isEmpty()) {
                    JOptionPane.showMessageDialog(this, "Нет данных");
                    return;
                }

                double avg = values.stream().mapToDouble(a -> a).average().getAsDouble();
                JOptionPane.showMessageDialog(this,
                        "Средний расход: " + avg);

                // график
                DefaultCategoryDataset dataset = new DefaultCategoryDataset();

                for (int i = 0; i < values.size(); i++) {
                    dataset.addValue(values.get(i), "Расход", String.valueOf(i));
                }

                JFreeChart chart = ChartFactory.createLineChart(
                        "История", "Поездки", "л/100км", dataset);

                JFrame chartFrame = new JFrame("График");
                chartFrame.add(new ChartPanel(chart));
                chartFrame.setSize(500, 400);
                chartFrame.setVisible(true);

            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
