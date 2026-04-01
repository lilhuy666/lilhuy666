package com.fuelapp;

import javax.swing.*;
import java.awt.*;
import java.awt.event.ActionEvent;
import java.io.FileWriter;

public class FuelAppSwing {

    public static void main(String[] args) {

        JFrame frame = new JFrame("⛽ Fuel Calculator");
        frame.setSize(420, 420);
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        frame.setLocationRelativeTo(null); // центр экрана

        JPanel panel = new JPanel();
        panel.setLayout(new GridLayout(0, 1, 8, 8));
        panel.setBorder(BorderFactory.createEmptyBorder(15, 20, 15, 20));
        panel.setBackground(new Color(245, 250, 245));

        Font font = new Font("Arial", Font.PLAIN, 14);

        JLabel userLabel = new JLabel("👤 Пользователь: Гость", JLabel.CENTER);
        userLabel.setFont(new Font("Arial", Font.BOLD, 14));

        JTextField distanceField = new JTextField();
        JTextField fuelField = new JTextField();
        JTextField priceField = new JTextField();

        distanceField.setFont(font);
        fuelField.setFont(font);
        priceField.setFont(font);

        JLabel resultLabel = new JLabel("Результат появится здесь", JLabel.CENTER);
        resultLabel.setFont(new Font("Arial", Font.BOLD, 14));

        JButton calcBtn = new JButton("🧮 Рассчитать");
        JButton saveBtn = new JButton("💾 Сохранить");

        styleButton(calcBtn, new Color(76, 175, 80));
        styleButton(saveBtn, new Color(56, 142, 60));

        // расчет
        calcBtn.addActionListener((ActionEvent e) -> {
            try {
                double d = Double.parseDouble(distanceField.getText());
                double f = Double.parseDouble(fuelField.getText());
                double p = Double.parseDouble(priceField.getText());

                if (d == 0) {
                    resultLabel.setText("❌ Дистанция не может быть 0");
                    return;
                }

                double consumption = (f / d) * 100;
                double cost = f * p;

                resultLabel.setText(String.format(
                        "<html>⛽ Расход: %.2f л/100км<br>💰 Стоимость: %.2f</html>",
                        consumption, cost
                ));

            } catch (Exception ex) {
                resultLabel.setText("❌ Проверь ввод данных");
            }
        });

        // сохранение
        saveBtn.addActionListener(e -> {
            try (FileWriter writer = new FileWriter("history.txt", true)) {

                writer.write(
                        "Км: " + distanceField.getText() +
                        ", Литры: " + fuelField.getText() +
                        ", Цена: " + priceField.getText() +
                        " -> " + resultLabel.getText().replaceAll("<.*?>", "") + "\n"
                );

                resultLabel.setText("✅ Сохранено");

            } catch (Exception ex) {
                resultLabel.setText("❌ Ошибка сохранения");
            }
        });

        // добавление элементов
        panel.add(userLabel);
        panel.add(createLabel("🚗 Километры:"));
        panel.add(distanceField);
        panel.add(createLabel("⛽ Литры:"));
        panel.add(fuelField);
        panel.add(createLabel("💵 Цена за литр:"));
        panel.add(priceField);
        panel.add(calcBtn);
        panel.add(saveBtn);
        panel.add(resultLabel);

        frame.add(panel);
        frame.setVisible(true);
    }

    private static JLabel createLabel(String text) {
        JLabel label = new JLabel(text);
        label.setFont(new Font("Arial", Font.PLAIN, 13));
        return label;
    }

    private static void styleButton(JButton button, Color color) {
        button.setBackground(color);
        button.setForeground(Color.WHITE);
        button.setFocusPainted(false);
        button.setFont(new Font("Arial", Font.BOLD, 13));
    }
}
