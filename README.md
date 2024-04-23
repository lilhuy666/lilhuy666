Вот пример кода на Kotlin и макета разметки XML для решения этой задачи в Android Studio:

1. Создайте макет разметки в файле activity_main.xml следующим образом:

<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <EditText
        android:id="@+id/editTextTicketNumber"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="Введите 6 цифр"
        android:inputType="number"/>

    <Button
        android:id="@+id/buttonCheckTicket"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_below="@id/editTextTicketNumber"
        android:text="Проверить билет"/>
    
    <TextView
        android:id="@+id/textViewResult"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_below="@id/buttonCheckTicket"
        android:layout_centerHorizontal="true"/>
</RelativeLayout>


2. В вашем файле MainActivity.kt добавьте следующий код:

import android.os.Bundle
import android.widget.Button
import android.widget.EditText
import android.widget.TextView
import androidx.appcompat.app.AppCompatActivity

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val editTextTicketNumber = findViewById<EditText>(R.id.editTextTicketNumber)
        val buttonCheckTicket = findViewById<Button>(R.id.buttonCheckTicket)
        val textViewResult = findViewById<TextView>(R.id.textViewResult)

        buttonCheckTicket.setOnClickListener {
            val input = editTextTicketNumber.text.toString()

            if (input.length == 6) {
                val firstHalf = input.substring(0, 3)
                val secondHalf = input.substring(3)

                val sumFirstHalf = firstHalf.map { it.toString().toInt() }.sum()
                val sumSecondHalf = secondHalf.map { it.toString().toInt() }.sum()

                if (sumFirstHalf == sumSecondHalf) {
                    textViewResult.text = "Билет счастливый!"
                } else {
                    textViewResult.text = "Билет не счастливый."
                }
            } else {
                textViewResult.text = "Некорректный ввод."
            }
        }
    }
}


Этот код позволяет пользователю ввести номер билета, после чего проверяет его на счастливость (сумма первых трех цифр равна сумме оставшихся трех цифр).

Убедитесь, что у вас есть корректный макет разметки и код MainActivity, затем запустите приложение в Android Studio.
