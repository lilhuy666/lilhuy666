import javax.swing.*;
import java.awt.*;
import java.awt.event.ActionEvent;

public class Main {
    public static void main(String[] args) {
        SwingUtilities.invokeLater(FuelCalculatorUI::new);
    }
}

class FuelCalculatorUI extends JFrame {

    private JTextField consumptionField, distanceField, priceField;
    private JTextField fuelUsedField, costField, costPerKmField;

    private JRadioButton calcTotalBtn, calcAvgBtn;

    public FuelCalculatorUI() {
        setTitle("Калькулятор расхода топлива");
        setSize(750, 500);
        setDefaultCloseOperation(EXIT_ON_CLOSE);
        setLocationRelativeTo(null);

        setLayout(new BorderLayout());

        // 🔷 Заголовок
        JLabel title = new JLabel("Калькулятор расхода топлива ТС");
        title.setFont(new Font("Arial", Font.BOLD, 18));
        title.setBorder(BorderFactory.createEmptyBorder(10, 10, 10, 10));
        add(title, BorderLayout.NORTH);

        JPanel mainPanel = new JPanel(new GridLayout(1, 2, 30, 0));
        mainPanel.setBorder(BorderFactory.createEmptyBorder(15, 15, 15, 15));

        // 🟦 Левая часть (ввод)
        JPanel leftPanel = new JPanel();
        leftPanel.setLayout(new BoxLayout(leftPanel, BoxLayout.Y_AXIS));

        calcTotalBtn = new JRadioButton("Рассчитать расход и стоимость", true);
        calcAvgBtn = new JRadioButton("Рассчитать средний расход на 100 км");

        ButtonGroup group = new ButtonGroup();
        group.add(calcTotalBtn);
        group.add(calcAvgBtn);

        leftPanel.add(calcTotalBtn);
        leftPanel.add(calcAvgBtn);

        consumptionField = createField(leftPanel, "Средний расход топлива (л/100 км):");
        distanceField = createField(leftPanel, "Расстояние (км):");
        priceField = createField(leftPanel, "Цена за литр (руб):");

        JButton calcBtn = new JButton("РАССЧИТАТЬ");
        calcBtn.setAlignmentX(Component.LEFT_ALIGNMENT);
        calcBtn.addActionListener(this::calculate);

        leftPanel.add(Box.createVerticalStrut(15));
        leftPanel.add(calcBtn);

        // 🟦 Правая часть (результаты)
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

    private JTextField createField(JPanel panel, String labelText) {
        JLabel label = new JLabel(labelText);
        JTextField field = new JTextField();

        label.setAlignmentX(Component.LEFT_ALIGNMENT);
        field.setMaximumSize(new Dimension(Integer.MAX_VALUE, 30));

        panel.add(Box.createVerticalStrut(10));
        panel.add(label);
        panel.add(field);

        return field;
    }

    private JTextField createResultField(JPanel panel, String labelText) {
        JLabel label = new JLabel(labelText);
        JTextField field = new JTextField();
        field.setEditable(false);

        label.setAlignmentX(Component.LEFT_ALIGNMENT);
        field.setMaximumSize(new Dimension(Integer.MAX_VALUE, 30));

        panel.add(Box.createVerticalStrut(15));
        panel.add(label);
        panel.add(field);

        return field;
    }

    private void calculate(ActionEvent e) {
        try {
            double consumption = Double.parseDouble(consumptionField.getText());
            double distance = Double.parseDouble(distanceField.getText());
            double price = Double.parseDouble(priceField.getText());

            double fuelUsed = (consumption / 100) * distance;
            double totalCost = fuelUsed * price;
            double costPerKm = totalCost / distance;

            fuelUsedField.setText(String.format("%.2f", fuelUsed));
            costField.setText(String.format("%.2f", totalCost));
            costPerKmField.setText(String.format("%.2f", costPerKm));

        } catch (Exception ex) {
            JOptionPane.showMessageDialog(this, "Ошибка ввода!");
        }
    }
}
