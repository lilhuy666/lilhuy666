package main.java;

import javafx.application.Application;
import javafx.geometry.Insets;
import javafx.geometry.Pos;
import javafx.scene.Scene;
import javafx.scene.control.*;
import javafx.scene.layout.*;
import javafx.stage.Stage;

public class FuelCalculatorApp extends Application {

    @Override
    public void start(Stage primaryStage) {
        // Заголовок
        Label title = new Label("Калькулятор расхода топлива");
        title.getStyleClass().add("title");

        // Поля ввода
        Label distLabel = new Label("Пройденное расстояние (км):");
        TextField distanceField = new TextField();
        distanceField.setPromptText("например, 450");

        Label fuelLabel = new Label("Потрачено топлива (л):");
        TextField fuelField = new TextField();
        fuelField.setPromptText("например, 36");

        // Кнопка расчёта
        Button calcButton = new Button("Рассчитать расход");
        calcButton.getStyleClass().add("main-button");

        // Результат
        Label resultLabel = new Label();
        resultLabel.getStyleClass().add("result-text");

        // Дополнительный расчёт
        CheckBox extraCalc = new CheckBox("Рассчитать топливо для другой дистанции");
        TextField extraDistanceField = new TextField();
        extraDistanceField.setPromptText("новая дистанция (км)");
        extraDistanceField.setDisable(true);
        Label extraResultLabel = new Label();
        extraResultLabel.getStyleClass().add("result-text");

        // Логика интерфейса
        extraCalc.selectedProperty().addListener((obs, wasSelected, isNowSelected) -> {
            extraDistanceField.setDisable(!isNowSelected);
            if (!isNowSelected) {
                extraResultLabel.setText("");
                extraDistanceField.clear();
            }
        });

        calcButton.setOnAction(e -> {
            try {
                double distance = Double.parseDouble(distanceField.getText());
                double fuel = Double.parseDouble(fuelField.getText());
                double consumption = (fuel / distance) * 100;
                resultLabel.setText(String.format("Расход: %.1f л/100 км", consumption));

                if (extraCalc.isSelected()) {
                    double newDist = Double.parseDouble(extraDistanceField.getText());
                    double fuelForDist = (consumption / 100) * newDist;
                    extraResultLabel.setText(String.format("Для %.0f км потребуется %.1f л", newDist, fuelForDist));
                }
            } catch (Exception ex) {
                resultLabel.setText("Введите корректные числа!");
                extraResultLabel.setText("");
            }
        });

        // Компоновка интерфейса (VBox)
        VBox root = new VBox(15);
        root.setAlignment(Pos.CENTER);
        root.setPadding(new Insets(30));
        root.getStyleClass().add("root");

        root.getChildren().addAll(
            title,
            distLabel, distanceField,
            fuelLabel, fuelField,
            calcButton,
            resultLabel,
            extraCalc,
            extraDistanceField,
            extraResultLabel
        );

        // Сцена и стили
        Scene scene = new Scene(root, 450, 550);
        scene.getStylesheets().add(getClass().getResource("/styles.css").toExternalForm());

        primaryStage.setTitle("Калькулятор топлива");
        primaryStage.setScene(scene);
        primaryStage.show();
    }

    public static void main(String[] args) {
        launch(args);
    }
}
