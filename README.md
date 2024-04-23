Для создания приложения "Счастливый билет" вам необходимо добавить в файл AndroidManifest.xml две активити, одна из которых будет отображать результат проверки билета, а другая будет предоставлять интерфейс для ввода номера билета. 

Пример кода для добавления двух активити в AndroidManifest.xml:

```xml
<application
    ...
    >
    <activity android:name=".MainActivity">
        <intent-filter>
            <action android:name="android.intent.action.MAIN"/>
            <category android:name="android.intent.category.LAUNCHER"/>
        </intent-filter>
    </activity>
    
    <activity android:name=".TicketCheckActivity"/>
</application>
```

В этом примере предполагается, что MainActivity будет активностью, запускаемой при старте приложения, а TicketCheckActivity будет активностью, отображающей результат проверки билета. 

Помимо этого, вам также понадобится создать layout файлы для обеих активити (activity_main.xml и activity_ticket_check.xml) и написать соответствующий Java (или Kotlin) код для обработки логики вашего приложения "Счастливый билет". 

Пожалуйста, убедитесь, что соответствующие layout файлы и классы активити также созданы и правильно настроены в вашем проекте.
