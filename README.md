Вот пример кода для реализации такого приложения на Kotlin в Android Studio:

MainActivity.kt:

import android.content.Intent
import androidx.appcompat.app.AppCompatActivity
import android.os.Bundle
import kotlinx.android.synthetic.main.activity_main.*
import org.json.JSONObject

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        btnSubmit.setOnClickListener {
            val jsonObject = JSONObject()
            jsonObject.put("param1", editText1.text.toString())
            jsonObject.put("param2", editText2.text.toString())
            jsonObject.put("param3", editText3.text.toString())
            jsonObject.put("param4", editText4.text.toString())
            jsonObject.put("param5", editText5.text.toString())

            val intent = Intent(this, SecondActivity::class.java)
            intent.putExtra("jsonObject", jsonObject.toString())
            startActivity(intent)
        }
    }
}


SecondActivity.kt:

import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import kotlinx.android.synthetic.main.activity_second.*
import org.json.JSONObject

class SecondActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_second)

        val jsonObjectString = intent.getStringExtra("jsonObject")
        val jsonObject = JSONObject(jsonObjectString)

        textView1.text = jsonObject.getString("param1")
        textView2.text = jsonObject.getString("param2")
        textView3.text = jsonObject.getString("param3")
        textView4.text = jsonObject.getString("param4")
        textView5.text = jsonObject.getString("param5")

        btnBack.setOnClickListener {
            val intent = Intent(this, MainActivity::class.java)
            intent.putExtra("jsonObject", jsonObject.toString())
            startActivity(intent)
        }
    }
}


activitymain.xml:

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
