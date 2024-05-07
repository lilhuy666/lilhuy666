```xml
<RelativeLayout>
    <EditText
        android:id="@+id/editText1"/>
    <EditText
        android:id="@+id/editText2"/>
    <EditText
        android:id="@+id/editText3"/>
    <EditText
        android:id="@+id/editText4"/>
    <EditText
        android:id="@+id/editText5"/>
    <Button
        android:id="@+id/btnSubmit"/>
</RelativeLayout>
```

activitysecond.xml:

<RelativeLayout>
    <TextView
        android:id="@+id/textView1"/>
    <TextView
        android:id="@+id/textView2"/>
    <TextView
        android:id="@+id/textView3"/>
    <TextView
        android:id="@+id/textView4"/>
    <TextView
        android:id="@+id/textView5"/>
    <Button
        android:id="@+id/btnBack"/>
</RelativeLayout>


Этот код создает две активности - MainActivity и SecondActivity. Пользователь вводит значения в EditText на MainActivity, нажимает на кнопку Submit, данные передаются в виде JSON объекта на SecondActivity, где они выводятся в TextView. На SecondActivity есть кнопка Back, которая возвращает пользователя на MainActivity с введенными ранее данными.
