import javafx.application.Application;
import javafx.geometry.Insets;
import javafx.scene.Scene;
import javafx.scene.control.*;
import javafx.scene.layout.VBox;
import javafx.stage.Stage;

public class Main extends Application {

    private User currentUser = new User("Гость");

    @Override
    public void start(Stage stage) {
        stage.setTitle("Fuel Calculator App");

        TextField distanceField = new TextField();
        distanceField.setPromptText("Км");

        TextField fuelField = new TextField();
        fuelField.setPromptText("Литры");

        TextField priceField = new TextField();
        priceField.setPromptText("Цена за литр");

        Label resultLabel = new Label();
        Label historyLabel = new Label("История:");

        Button calcBtn = new Button("Рассчитать");
        Button saveBtn = new Button("Сохранить");

        calcBtn.setOnAction(e -> {
            try {
                double d = Double.parseDouble(distanceField.getText());
                double f = Double.parseDouble(fuelField.getText());
                double p = Double.parseDouble(priceField.getText());

                double consumption = (f / d) * 100;
                double cost = f * p;

                String result = String.format(
                        "Расход: %.2f л/100км | Стоимость: %.2f",
                        consumption, cost
                );

                resultLabel.setText(result);
                currentUser.addHistory(result);

            } catch (Exception ex) {
                resultLabel.setText("Ошибка ввода");
            }
        });

        saveBtn.setOnAction(e -> {
            DataManager.save(currentUser);
            historyLabel.setText("Сохранено!");
        });

        VBox root = new VBox(10,
                new Label("Пользователь: " + currentUser.getName()),
                distanceField,
                fuelField,
                priceField,
                calcBtn,
                resultLabel,
                saveBtn,
                historyLabel
        );

        root.setPadding(new Insets(20));

        Scene scene = new Scene(root, 400, 400);
        scene.getStylesheets().add("style.css");

        stage.setScene(scene);
        stage.show();
    }

    public static void main(String[] args) {
        launch();
    }
}
