import com.formdev.flatlaf.FlatDarkLaf;

import javax.swing.*;
import javax.swing.table.DefaultTableModel;
import java.awt.*;
import java.awt.event.ActionEvent;
import java.sql.*;

public class Main {

    public static void main(String[] args) {
        try {
            FlatDarkLaf.setup();
        } catch (Exception e) {
            e.printStackTrace();
        }

        Database.init();
        SwingUtilities.invokeLater(FuelCalculatorUI::new);
    }
}

// ================= БАЗА ДАННЫХ =================
class Database {
    private static final String URL = "jdbc:sqlite:fuel.db";

    public static Connection connect() throws SQLException {
        return DriverManager.getConnection(URL);
    }

    public static void init() {
        String sql = """
                CREATE TABLE IF NOT EXISTS history (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    distance REAL,
                    fuel REAL,
                    price REAL,
                    consumption REAL,
                    total_cost REAL,
                    cost_per_km REAL,
                    date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                );
                """;

        try (Connection conn = connect();
             Statement stmt = conn.createStatement()) {
            stmt.execute(sql);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static void save(double distance, double fuel, double price,
                            double consumption, double totalCost, double costPerKm) {

        String sql = "INSERT INTO history(distance,fuel,price,consption,total_cost,cost_per_km) VALUES(?,?,?,?,?,?)";

        try (Connection conn = connect();
             PreparedStatement ps = conn.prepareStatement(sql)) {

            ps.setDouble(1, distance);
            ps.setDouble(2, fuel);
            ps.setDouble(3, price);
            ps.setDouble(4, consumption);
            ps.setDouble(5, totalCost);
            ps.setDouble(6, costPerKm);
            ps.executeUpdate();

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

// ================= UI =================
class FuelCalculatorUI extends JFrame {

    private JTextField consumptionField, distanceField, priceField;
    private JTextField fuelUsedField, costField, costPerKmField;

    private JLabel consumptionLabel, distanceLabel;

    private JRadioButton calcTotalBtn, calcAvgBtn;

    public FuelCalculatorUI() {
        setTitle("Калькулятор расхода топлива");
        setSize(750, 500);
        setDefaultCloseOperation(EXIT_ON_CLOSE);
        setLocationRelativeTo(null);

        setLayout(new BorderLayout());

        // 🔷 TOP BAR
        JPanel topPanel = new JPanel(new BorderLayout());
        JLabel title = new JLabel("Калькулятор расхода топлива ТС");
        title.setBorder(BorderFactory.createEmptyBorder(10, 10, 10, 10));
        title.setFont(new Font("Arial", Font.BOLD, 16));

        // ☰ меню
        JPopupMenu menu = new JPopupMenu();
        JMenuItem home = new JMenuItem("Главная");
        JMenuItem history = new JMenuItem("История");
        JMenuItem profile = new JMenuItem("Личный кабинет");

        menu.add(home);
        menu.add(history);
        menu.add(profile);

        JButton menuBtn = new JButton("☰");
        menuBtn.addActionListener(e -> menu.show(menuBtn, 0, menuBtn.getHeight()));

        history.addActionListener(e -> new HistoryWindow());

        topPanel.add(title, BorderLayout.WEST);
        topPanel.add(menuBtn, BorderLayout.EAST);

        add(topPanel, BorderLayout.NORTH);

        // 🔷 MAIN
        JPanel mainPanel = new JPanel(new GridLayout(1, 2, 30, 0));
        mainPanel.setBorder(BorderFactory.createEmptyBorder(15, 15, 15, 15));

        // 🟦 LEFT
        JPanel leftPanel = new JPanel();
        leftPanel.setLayout(new BoxLayout(leftPanel, BoxLayout.Y_AXIS));

        calcTotalBtn = new JRadioButton("Рассчитать расход и стоимость", true);
        calcAvgBtn = new JRadioButton("Рассчитать средний расход на 100 км");

        ButtonGroup group = new ButtonGroup();
        group.add(calcTotalBtn);
        group.add(calcAvgBtn);

        calcTotalBtn.addActionListener(e -> switchMode());
        calcAvgBtn.addActionListener(e -> switchMode());

        leftPanel.add(calcTotalBtn);
        leftPanel.add(calcAvgBtn);

        consumptionLabel = new JLabel("Средний расход (л/100 км):");
        consumptionField = createField(leftPanel, consumptionLabel);

        distanceLabel = new JLabel("Расстояние (км):");
        distanceField = createField(leftPanel, distanceLabel);

        priceField = createField(leftPanel, new JLabel("Цена за литр (руб):"));

        JButton calcBtn = new JButton("Рассчитать");
        calcBtn.addActionListener(this::calculate);

        leftPanel.add(Box.createVerticalStrut(15));
        leftPanel.add(calcBtn);

        // 🟦 RIGHT
        JPanel rightPanel = new JPanel();
        rightPanel.setLayout(new BoxLayout(rightPanel, BoxLayout.Y_AXIS));

        fuelUsedField = createResultField(rightPanel, "Израсходовано топлива (л):");
        costField = createResultField(rightPanel, "Стоимость (руб):");
        costPerKmField = createResultField(rightPanel, "Цена за км:");

        mainPanel.add(leftPanel);
        mainPanel.add(rightPanel);

        add(mainPanel, BorderLayout.CENTER);

        setVisible(true);
    }

    private JTextField createField(JPanel panel, JLabel label) {
        JTextField field = new JTextField();
        field.setMaximumSize(new Dimension(Integer.MAX_VALUE, 30));

        panel.add(Box.createVerticalStrut(10));
        panel.add(label);
        panel.add(field);

        return field;
    }

    private JTextField createResultField(JPanel panel, String text) {
        JLabel label = new JLabel(text);
        JTextField field = new JTextField();
        field.setEditable(false);

        field.setMaximumSize(new Dimension(Integer.MAX_VALUE, 30));

        panel.add(Box.createVerticalStrut(15));
        panel.add(label);
        panel.add(field);

        return field;
    }

    // 🔄 смена режима
    private void switchMode() {
        if (calcAvgBtn.isSelected()) {
            consumptionLabel.setText("Израсходовано топлива (л):");
            distanceLabel.setText("Пройденное расстояние (км):");
        } else {
            consumptionLabel.setText("Средний расход (л/100 км):");
            distanceLabel.setText("Расстояние (км):");
        }
    }

    // 🧮 расчет
    private void calculate(ActionEvent e) {
        try {
            double consumption = Double.parseDouble(consumptionField.getText());
            double distance = Double.parseDouble(distanceField.getText());
            double price = Double.parseDouble(priceField.getText());

            double fuelUsed;
            double avgConsumption;

            if (calcAvgBtn.isSelected()) {
                fuelUsed = consumption;
                avgConsumption = (fuelUsed / distance) * 100;
            } else {
                avgConsumption = consumption;
                fuelUsed = (avgConsumption / 100) * distance;
            }

            double totalCost = fuelUsed * price;
            double costPerKm = totalCost / distance;

            fuelUsedField.setText(String.format("%.2f", fuelUsed));
            costField.setText(String.format("%.2f", totalCost));
            costPerKmField.setText(String.format("%.2f", costPerKm));

            Database.save(distance, fuelUsed, price, avgConsumption, totalCost, costPerKm);

        } catch (Exception ex) {
            JOptionPane.showMessageDialog(this, "Ошибка ввода!");
        }
    }
}

// ================= ИСТОРИЯ =================
class HistoryWindow extends JFrame {

    public HistoryWindow() {
        setTitle("История");
        setSize(600, 400);
        setLocationRelativeTo(null);

        String[] cols = {"Дата", "Км", "Литры", "Цена", "Расход", "Стоимость"};
        DefaultTableModel model = new DefaultTableModel(cols, 0);

        JTable table = new JTable(model);

        try (Connection conn = Database.connect();
             Statement st = conn.createStatement();
             ResultSet rs = st.executeQuery("SELECT * FROM history")) {

            while (rs.next()) {
                model.addRow(new Object[]{
                        rs.getString("date"),
                        rs.getDouble("distance"),
                        rs.getDouble("fuel"),
                        rs.getDouble("price"),
                        rs.getDouble("consumption"),
                        rs.getDouble("total_cost")
                });
            }

        } catch (Exception e) {
            e.printStackTrace();
        }

        add(new JScrollPane(table));
        setVisible(true);
    }
}
