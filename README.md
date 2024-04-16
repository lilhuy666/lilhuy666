Чтобы сделать так, чтобы ваше приложение в указанное время играло звуковой сигнал, вам необходимо добавить подходящую реализацию намеченного будильника. Вам также понадобится код, который будет запускать звуковой сигнал в момент, когда наступит заданное время. Вот пример, как это можно сделать:

1. Добавьте следующее разрешение в ваш файл манифеста AndroidManifest.xml:

<uses-permission android:name="android.permission.WAKE_LOCK" />


2. Создайте новый класс AlarmReceiver, который будет отвечать за воспроизведение звукового сигнала:

public class AlarmReceiver extends BroadcastReceiver {

    @Override
    public void onReceive(Context context, Intent intent) {
        // Воспроизведение звукового сигнала
        Toast.makeText(context, "Alarm! Wake up!", Toast.LENGTH_LONG).show();
    }
}


3. Измените ваш MainActivity.java следующим образом, чтобы установить будильник на выбранное время:

private void showDaysSelectionDialog() {
    // Ваш существующий код

    .setPositiveButton("OK", new DialogInterface.OnClickListener() {
        @Override
        public void onClick(DialogInterface dialog, int which) {
            // Ваш существующий код
            setAlarm();
        }
    })
}

private void setAlarm() {
    AlarmManager alarmManager = (AlarmManager) getSystemService(Context.ALARM_SERVICE);
    Intent alarmIntent = new Intent(this, AlarmReceiver.class);
    PendingIntent pendingIntent = PendingIntent.getBroadcast(this, 0, alarmIntent, 0);

    Calendar calendar = Calendar.getInstance();
    // Установка времени срабатывания будильника
    calendar.setTimeInMillis(System.currentTimeMillis());
    calendar.set(Calendar.HOUR_OF_DAY, hourOfDay);
    calendar.set(Calendar.MINUTE, minute);

    // Конвертация выбранных дней недели в формат, подходящий для setRepeating
    // Этот код предполагает, что выбранные дни хранятся в формате строк selectedDays (например, "Monday,Tuesday")
    
    // В этом примере срабатывание будильника будет происходить каждый день
    alarmManager.setRepeating(AlarmManager.RTC_WAKEUP, calendar.getTimeInMillis(), AlarmManager.INTERVAL_DAY, pendingIntent);
}


Обратите внимание, что данный пример устанавливает будильник на каждый день в выбранное время. Все подробности, такие как выбор конкретных дней недели и точная реализация воспроизведения звукового сигнала, могут потребовать дополнительной обработки и проработки вашей логики. Важно убедиться, что ваше приложение имеет не только установленный будильник, но и последующий функционал для воспроизведения звукового сигнала в нужное время.
