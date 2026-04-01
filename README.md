import javax.swing.*;
import javax.swing.table.DefaultTableModel;
import java.awt.*;
import java.awt.event.ActionEvent;
import java.util.ArrayList;

public class Main {

    public static void main(String[] args) {
        SwingUtilities.invokeLater(FuelCalculatorUI::new);
    }
}

// ================= ХРАНЕНИЕ В ПАМЯТИ =================
class Record {
    double distance, fuel, price, consumption, totalCost;

    public Record(double distance, double fuel, double price, double consumption, double totalCost) {
        this.distance = distance;
        this.fuel = fuel;
        this.price = price;
        this.consumption = consumption;
        this.totalCost = totalCost;
    }
}

class DataStore {
    public static ArrayList<Record> history = new ArrayList<>();
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

        // 🔷 TOP
        JPanel topPanel = new JPanel(new BorderLayout());
        topPanel.setBackground(new Color(40, 60, 80));

        JLabel title = new JLabel("Калькулятор расхода топлива ТС");
        title.setForeground(Color.WHITE);
        title.setBorder(BorderFactory.createEmptyBorder(10, 10, 10, 10));

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
        calcBtn.setBackground(new Color(255, 140, 0));
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

            // 💾 сохраняем в память
            DataStore.history.add(new Record(distance, fuelUsed, price, avgConsumption, totalCost));

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

        String[] cols = {"Км", "Литры", "Цена", "Расход", "Стоимость"};
        DefaultTableModel model = new DefaultTableModel(cols, 0);

        JTable table = new JTable(model);

        for (Record r : DataStore.history) {
            model.addRow(new Object[]{
                    r.distance,
                    r.fuel,
                    r.price,
                    r.consumption,
                    r.totalCost
            });
        }

        add(new JScrollPane(table));
        setVisible(true);
    }
}
