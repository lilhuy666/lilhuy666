import javax.swing.*;
import java.awt.*;
import java.awt.event.ActionEvent;
import java.awt.event.ActionListener;

public class Main {

    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> new FuelCalculatorApp());
    }
}

class FuelCalculatorApp extends JFrame {

    private JTextField distanceField;
    private JTextField fuelField;
    private JTextField priceField;

    private JLabel consumptionResult;
    private JLabel costResult;
    private JLabel costPerKmResult;

    public FuelCalculatorApp() {
        setTitle("Калькулятор расхода топлива");
        setSize(400, 400);
        setDefaultCloseOperation(EXIT_ON_CLOSE);
        setLocationRelativeTo(null);

        setLayout(new BorderLayout());

        JPanel mainPanel = new JPanel();
        mainPanel.setLayout(new BoxLayout(mainPanel, BoxLayout.Y_AXIS));
        mainPanel.setBorder(BorderFactory.createEmptyBorder(15, 15, 15, 15));

        // Ввод данных
        mainPanel.add(createLabel("Ввод данных"));

        distanceField = createInput(mainPanel, "Пробег (км):");
        fuelField = createInput(mainPanel, "Топливо (л):");
        priceField = createInput(mainPanel, "Цена за литр:");

        // Кнопка
        JButton calculateButton = new JButton("Рассчитать");
        calculateButton.setAlignmentX(Component.CENTER_ALIGNMENT);
        calculateButton.setBackground(Color.ORANGE);
        calculateButton.setFocusPainted(false);
        calculateButton.setFont(new Font("Arial", Font.BOLD, 16));
        calculateButton.addActionListener(new CalculateAction());

        mainPanel.add(Box.createVerticalStrut(10));
        mainPanel.add(calculateButton);

        // Результаты
        mainPanel.add(Box.createVerticalStrut(20));
        mainPanel.add(createLabel("Результаты"));

        consumptionResult = new JLabel("Расход топлива: -");
        costResult = new JLabel("Стоимость поездки: -");
        costPerKmResult = new JLabel("Цена за 1 км: -");

        mainPanel.add(consumptionResult);
        mainPanel.add(costResult);
        mainPanel.add(costPerKmResult);

        add(mainPanel);
        setVisible(true);
    }

    private JLabel createLabel(String text) {
        JLabel label = new JLabel(text);
        label.setFont(new Font("Arial", Font.BOLD, 16));
        label.setAlignmentX(Component.CENTER_ALIGNMENT);
        return label;
    }

    private JTextField createInput(JPanel panel, String labelText) {
        JPanel row = new JPanel(new BorderLayout());
        JLabel label = new JLabel(labelText);
        JTextField field = new JTextField();

        row.add(label, BorderLayout.WEST);
        row.add(field, BorderLayout.CENTER);

        panel.add(Box.createVerticalStrut(10));
        panel.add(row);

        return field;
    }

    class CalculateAction implements ActionListener {
        @Override
        public void actionPerformed(ActionEvent e) {
            try {
                double distance = Double.parseDouble(distanceField.getText());
                double fuel = Double.parseDouble(fuelField.getText());
                double price = Double.parseDouble(priceField.getText());

                double consumption = (fuel / distance) * 100;
                double totalCost = fuel * price;
                double costPerKm = totalCost / distance;

                consumptionResult.setText("Расход топлива: " + String.format("%.2f", consumption) + " л/100км");
                costResult.setText("Стоимость поездки: " + String.format("%.2f", totalCost) + " руб.");
                costPerKmResult.setText("Цена за 1 км: " + String.format("%.2f", costPerKm) + " руб/км");

            } catch (Exception ex) {
                JOptionPane.showMessageDialog(null, "Введите корректные данные!");
            }
        }
    }
}
