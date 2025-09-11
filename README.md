<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:padding="16dp"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <EditText
        android:id="@+id/editTextNumber1"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:inputType="number"
        android:hint="Число 1" />

    <EditText
        android:id="@+id/editTextNumber2"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:inputType="number"
        android:hint="Число 2" />

    <LinearLayout
        android:orientation="horizontal"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="8dp">

        <Button
            android:id="@+id/buttonAdd"
            android:layout_weight="1"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:text="+" />

        <Button
            android:id="@+id/buttonSubtract"
            android:layout_weight="1"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:text="−" />
    </LinearLayout>

    <TextView
        android:id="@+id/textViewResult"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Результат:"
        android:textSize="18sp"
        android:layout_marginTop="12dp" />
</LinearLayout>


MainActivity.kt
package com.example.simplecalculator

import android.os.Bundle
import android.widget.Button
import android.widget.EditText
import android.widget.TextView
import androidx.appcompat.app.AppCompatActivity

class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val et1 = findViewById<EditText>(R.id.editTextNumber1)
        val et2 = findViewById<EditText>(R.id.editTextNumber2)
        val btnAdd = findViewById<Button>(R.id.buttonAdd)
        val btnSub = findViewById<Button>(R.id.buttonSubtract)
        val tvResult = findViewById<TextView>(R.id.textViewResult)

        fun parseIntSafe(edit: EditText): Int? {
            val txt = edit.text.toString().trim()
            if (txt.isEmpty()) return null
            return try {
                // Используем требуемую конструкцию с Integer.parseInt
                val N1: Int? = 0
                Integer.parseInt(txt)
            } catch (e: NumberFormatException) {
                null
            }
        }

        btnAdd.setOnClickListener {
            val a = parseIntSafe(et1)
            val b = parseIntSafe(et2)
            if (a == null || b == null) {
                tvResult.text = "Ошибка: введите целые числа"
            } else {
                val res = a + b
                tvResult.text = "Результат: $res"
            }
        }

        btnSub.setOnClickListener {
            val a = parseIntSafe(et1)
            val b = parseIntSafe(et2)
            if (a == null || b == null) {
                tvResult.text = "Ошибка: введите целые числа"
            } else {
                val res = a - b
                tvResult.text = "Результат: $res"
            }
        }
    }
}
