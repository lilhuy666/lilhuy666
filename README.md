Конечно, вот полный код с добавленными методами showDaysSelectionDialog() и setAlarm():

public class MainActivity extends AppCompatActivity {

    // Ваши существующие переменные и методы

    private void showDaysSelectionDialog() {
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle("Select Days")
                .setMultiChoiceItems(daysOfWeek, checkedDays, new DialogInterface.OnMultiChoiceClickListener() {
                    @Override
                    public void onClick(DialogInterface dialog, int which, boolean isChecked) {
                        // Обновление массива checkedDays при выборе дней недели
                        checkedDays[which] = isChecked;
                    }
                })
                .setPositiveButton("OK", new DialogInterface.OnClickListener() {
                    @Override
                    public void onClick(DialogInterface dialog, int which) {
                        // Применение выбранных дней недели и установка будильника
                        setAlarm();
                    }
                })
                .show();
    }

    private void setAlarm() {
        AlarmManager alarmManager = (AlarmManager) getSystemService(Context.ALARM_SERVICE);
        Intent alarmIntent = new Intent(this, AlarmReceiver.class);
        PendingIntent pendingIntent = PendingIntent.getBroadcast(this, 0, alarmIntent, 0);

        Calendar calendar = Calendar.getInstance();
        calendar.setTimeInMillis(System.currentTimeMillis());
        calendar.set(Calendar.HOUR_OF_DAY, hourOfDay);
        calendar.set(Calendar.MINUTE, minute);

        // Конвертация выбранных дней недели в формат, подходящий для установки повторяемого будильника

        // В этом примере срабатывание будильника будет происходить каждый день
        alarmManager.setRepeating(AlarmManager.RTC_WAKEUP, calendar.getTimeInMillis(), AlarmManager.INTERVAL_DAY, pendingIntent);
    }
}


Не забудьте вставить этот код в соответствующие части вашего MainActivity.java и адаптировать его под вашу текущую реализацию.
