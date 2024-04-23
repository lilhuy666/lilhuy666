Конечно, вот пример кода для приложения Android Studio, которое определяет, является ли введенный пользователем номер билета счастливым:

<!-- activity_main.xml -->
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <EditText
        android:id="@+id/editText"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="Введите номер билета" />

    <Button
        android:id="@+id/button"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Проверить" />

    <TextView
        android:id="@+id/textView"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Результат проверки" />

</LinearLayout>


// MainActivity.kt
import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import kotlinx.android.synthetic.main.activity_main.*

class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        button.setOnClickListener {
            val ticketNumber = editText.text.toString()

            val numbers = ticketNumber.map { it.toString().toInt() }
            val middleIndex = numbers.size / 2

            val firstHalfSum = numbers.take(middleIndex).sum()
            val secondHalfSum = numbers.drop(middleIndex).sum()

            if (firstHalfSum == secondHalfSum) {
                textView.text = "Это счастливый билет!"
            } else {
                textView.text = "Это обычный билет :("
            }
        }
    }
}


Не забудьте добавить зависимости и импорты в свой проект. Данный код позволит пользователю ввести номер билета в EditText, нажать на кнопку "Проверить", и затем отобразит результат проверки в TextView. Пожалуйста, дайте знать, если вам нужна любая дополнительная помощь!
