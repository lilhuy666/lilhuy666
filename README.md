Для создания будильника с возможностью выбора дня недели и времени, можно использовать TimePickerDialog для выбора времени и AlertDialog с чекбоксами для выбора дней недели. Вот пример кода для активности MainActivity.java:

import android.app.TimePickerDialog;
import android.content.DialogInterface;
import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;
import android.widget.TimePicker;

import androidx.appcompat.app.AlertDialog;
import androidx.appcompat.app.AppCompatActivity;

import java.util.Calendar;

public class MainActivity extends AppCompatActivity {

    private String selectedTime = "Будильник не установлен";
    private String selectedDays = "Ежедневно"; // По умолчанию будильник настроен на каждый день
    private TextView selectedTimeTextView;
    private TextView selectedDaysTextView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        selectedTimeTextView = findViewById(R.id.selected_time_textview);
        selectedDaysTextView = findViewById(R.id.selected_days_textview);
        Button setAlarmButton = findViewById(R.id.set_alarm_button);

        setAlarmButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                final Calendar c = Calendar.getInstance();
                int hour = c.get(Calendar.HOUR_OF_DAY);
                int minute = c.get(Calendar.MINUTE);

                TimePickerDialog timePickerDialog = new TimePickerDialog(MainActivity.this,
                        new TimePickerDialog.OnTimeSetListener() {
                            @Override
                            public void onTimeSet(TimePicker view, int hourOfDay, int minute) {
                                selectedTime = String.format("%02d:%02d", hourOfDay, minute);
                                selectedTimeTextView.setText("Установленный будильник: " + selectedTime + " (" + selectedDays + ")");
                            }
                        }, hour, minute, true);
                timePickerDialog.show();
                
                final String[] daysArray = {"Понедельник", "Вторник", "Среда", "Четверг", "Пятница", "Суббота", "Воскресенье"};
                final boolean[] checkedItems = {true, true, true, true, true, false, false}; // Выбираем Пн-Пт по умолчанию
                
                AlertDialog.Builder builder = new AlertDialog.Builder(MainActivity.this);
                builder.setTitle("Выберите дни недели");
                builder.setMultiChoiceItems(daysArray, checkedItems, new DialogInterface.OnMultiChoiceClickListener() {
                    @Override
                    public void onClick(DialogInterface dialogInterface, int i, boolean b) {
                        checkedItems[i] = b;
                    }
                });
                builder.setPositiveButton("OK", new DialogInterface.OnClickListener() {
                    @Override
                    public void onClick(DialogInterface dialogInterface, int i) {
                        StringBuilder selectedDaysBuilder = new StringBuilder();
                        for (int j = 0; j < daysArray.length; j++) {
                            if (checkedItems[j]) {
                                selectedDaysBuilder.append(daysArray[j]);
                                selectedDaysBuilder.append(", ");
                            }
                        }
                        selectedDays = selectedDaysBuilder.toString();
                        selectedDays = selectedDays.substring(0, selectedDays.length() - 2); // Убираем лишнюю запятую и пробел в конце
                        selectedDaysTextView.setText("Выбранные дни: " + selectedDays);
                    }
                });
                builder.setNegativeButton("Отмена", null);
                builder.show();
            }
        });
    }
}


В этом примере, когда нажимается кнопка "Установить будильни
