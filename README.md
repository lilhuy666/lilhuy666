import javax.swing.*;
import java.awt.*;
import java.awt.event.ActionEvent;
import java.io.FileWriter;

public class FuelAppSwing {

    public static void main(String[] args) {
        JFrame frame = new JFrame("Fuel Calculator");
        frame.setSize(400, 400);
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);

        JPanel panel = new JPanel();
        panel.setLayout(new GridLayout(8, 1, 10, 10));
        panel.setBackground(new Color(232, 245, 233)); // светло-зеленый

        JTextField distanceField = new JTextField();
        JTextField fuelField = new JTextField();
        JTextField priceField = new JTextField();

        JLabel resultLabel = new JLabel("Результат:");
        JLabel userLabel = new JLabel("Пользователь: Гость");

        JButton calcBtn = new JButton("Рассчитать");
        JButton saveBtn = new JButton("Сохранить");

        // стиль кнопок
        calcBtn.setBackground(new Color(76, 175, 80));
        calcBtn.setForeground(Color.WHITE);

        saveBtn.setBackground(new Color(56, 142, 60));
        saveBtn.setForeground(Color.WHITE);

        // расчет
        calcBtn.addActionListener((ActionEvent e) -> {
            try {
                double d = Double.parseDouble(distanceField.getText());
                double f = Double.parseDouble(fuelField.getText());
                double p = Double.parseDouble(priceField.getText());

                double consumption = (f / d) * 100;
                double cost = f * p;

                resultLabel.setText(
                        String.format("Расход: %.2f | Стоимость: %.2f", consumption, cost)
                );

            } catch (Exception ex) {
                resultLabel.setText("Ошибка ввода");
            }
        });

        // сохранение
        saveBtn.addActionListener(e -> {
            try (FileWriter writer = new FileWriter("history.txt", true)) {
                writer.write(resultLabel.getText() + "\n");
            } catch (Exception ex) {
                resultLabel.setText("Ошибка сохранения");
            }
        });

        panel.add(userLabel);
        panel.add(new JLabel("Км:"));
        panel.add(distanceField);
        panel.add(new JLabel("Литры:"));
        panel.add(fuelField);
        panel.add(new JLabel("Цена за литр:"));
        panel.add(priceField);
        panel.add(calcBtn);
        panel.add(saveBtn);
        panel.add(resultLabel);

        frame.add(panel);
        frame.setVisible(true);
    }
}
