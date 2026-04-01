import javax.swing.*;
import java.awt.*;
import java.awt.event.ActionEvent;

public class Main {
    public static void main(String[] args) {
        SwingUtilities.invokeLater(FuelCalculatorUI::new);
    }
}

class FuelCalculatorUI extends JFrame {

    private JTextField distanceField, fuelField, priceField;
    private JLabel consumptionLabel, costLabel, perKmLabel;

    public FuelCalculatorUI() {
        setTitle("Калькулятор расхода топлива");
        setSize(700, 450);
        setDefaultCloseOperation(EXIT_ON_CLOSE);
        setLocationRelativeTo(null);
        setLayout(new BorderLayout());

        // 🔷 Верхняя панель
        JPanel topPanel = new JPanel(new BorderLayout());
        topPanel.setBackground(new Color(30, 60, 90));
        topPanel.setPreferredSize(new Dimension(0, 50));

        JLabel title = new JLabel("Калькулятор расхода топлива");
        title.setForeground(Color.WHITE);
        title.setFont(new Font("Arial", Font.BOLD, 18));
        title.setBorder(BorderFactory.createEmptyBorder(0, 15, 0, 0));

        topPanel.add(title, BorderLayout.WEST);
        add(topPanel, BorderLayout.NORTH);

        // 🔷 Правая панель (меню)
        JPanel rightMenu = new JPanel();
        rightMenu.setLayout(new GridLayout(3, 1, 10, 10));
        rightMenu.setBorder(BorderFactory.createEmptyBorder(10, 10, 10, 10));
        rightMenu.setPreferredSize(new Dimension(180, 0));

        JButton profileBtn = new JButton("Личный кабинет");
        JButton homeBtn = new JButton("Главная");
        JButton historyBtn = new JButton("История");

        rightMenu.add(profileBtn);
        rightMenu.add(homeBtn);
        rightMenu.add(historyBtn);

        add(rightMenu, BorderLayout.EAST);

        // 🔷 Центральная часть
        JPanel centerPanel = new JPanel(new GridLayout(1, 2, 15, 0));
        centerPanel.setBorder(BorderFactory.createEmptyBorder(15, 15, 15, 15));

        // 🟦 Панель ввода
        JPanel inputPanel = new JPanel();
        inputPanel.setLayout(new BoxLayout(inputPanel, BoxLayout.Y_AXIS));
        inputPanel.setBorder(BorderFactory.createTitledBorder("Ввод данных"));

        distanceField = createField(inputPanel, "Пробег (км):");
        fuelField = createField(inputPanel, "Топливо (л):");
        priceField = createField(inputPanel, "Цена (руб/л):");

        JButton calcButton = new JButton("Рассчитать");
        calcButton.setAlignmentX(Component.CENTER_ALIGNMENT);
        calcButton.setBackground(Color.ORANGE);
        calcButton.addActionListener(this::calculate);

        inputPanel.add(Box.createVerticalStrut(15));
        inputPanel.add(calcButton);

        // 🟦 Панель результатов
        JPanel resultPanel = new JPanel();
        resultPanel.setLayout(new BoxLayout(resultPanel, BoxLayout.Y_AXIS));
        resultPanel.setBorder(BorderFactory.createTitledBorder("Результаты"));

        consumptionLabel = new JLabel("Расход: -");
        costLabel = new JLabel("Стоимость: -");
        perKmLabel = new JLabel("Цена за км: -");

        resultPanel.add(Box.createVerticalStrut(20));
        resultPanel.add(consumptionLabel);
        resultPanel.add(Box.createVerticalStrut(10));
        resultPanel.add(costLabel);
        resultPanel.add(Box.createVerticalStrut(10));
        resultPanel.add(perKmLabel);

        // 👉 одинаковый размер
        inputPanel.setPreferredSize(new Dimension(250, 300));
        resultPanel.setPreferredSize(new Dimension(250, 300));

        centerPanel.add(inputPanel);
        centerPanel.add(resultPanel);

        add(centerPanel, BorderLayout.CENTER);

        setVisible(true);
    }

    private JTextField createField(JPanel panel, String labelText) {
        JPanel row = new JPanel(new BorderLayout(5, 5));
        JLabel label = new JLabel(labelText);
        JTextField field = new JTextField();

        row.add(label, BorderLayout.WEST);
        row.add(field, BorderLayout.CENTER);

        panel.add(Box.createVerticalStrut(10));
        panel.add(row);

        return field;
    }

    private void calculate(ActionEvent e) {
        try {
            double distance = Double.parseDouble(distanceField.getText());
            double fuel = Double.parseDouble(fuelField.getText());
            double price = Double.parseDouble(priceField.getText());

            double consumption = (fuel / distance) * 100;
            double totalCost = fuel * price;
            double costPerKm = totalCost / distance;

            consumptionLabel.setText("Расход: " + String.format("%.2f", consumption) + " л/100км");
            costLabel.setText("Стоимость: " + String.format("%.2f", totalCost) + " руб.");
            perKmLabel.setText("Цена за км: " + String.format("%.2f", costPerKm) + " руб/км");

        } catch (Exception ex) {
            JOptionPane.showMessageDialog(this, "Ошибка ввода данных!");
        }
    }
}
