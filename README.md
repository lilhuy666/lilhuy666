Разметка activity_main.xml для предоставленного кода MainActivity.java может выглядеть следующим образом:

```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <Button
        android:id="@+id/set_alarm_button"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Установить будильник"
        android:layout_centerHorizontal="true"
        android:layout_marginTop="16dp" />

    <TextView
        android:id="@+id/selected_time_textview"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Установленный будильник: Будильник не установлен (Ежедневно)"
        android:layout_below="@id/set_alarm_button"
        android:layout_centerHorizontal="true"
        android:layout_marginTop="16dp" />

    <TextView
        android:id="@+id/selected_days_textview"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Выбранные дни: Понедельник, Вторник, Среда, Четверг, Пятница"
        android:layout_below="@id/selected_time_textview"
        android:layout_centerHorizontal="true"
        android:layout_marginTop="16dp" />

</RelativeLayout>
```

Этот макет содержит кнопку "Установить будильник" и два текстовых поля для отображения выбранного времени и выбранных дней недели. При нажатии на кнопку "Установить будильник", открывается диалог выбора времени с помощью TimePickerDialog, а затем диалог выбора дней недели с помощью AlertDialog с чекбоксами. Выбранные значения отображаются в соответствующих текстовых полях на экране.
