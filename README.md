Вот пример кода для класса MainActivity, который реализует основную логику вашего приложения "Расписание уроков":

### MainActivity.java

package com.example.raspisanieurokov; // Замените на ваш пакет

import androidx.appcompat.app.AppCompatActivity;
import androidx.recyclerview.widget.LinearLayoutManager;
import androidx.recyclerview.widget.RecyclerView;

import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.EditText;

import java.util.ArrayList;
import java.util.List;

public class MainActivity extends AppCompatActivity {

    private List<Lesson> lessonList; // Список уроков
    private LessonAdapter adapter; // Адаптер для RecyclerView
    private EditText editTextSubject; // Поле ввода названия предмета
    private EditText editTextTime; // Поле ввода времени
    private Button buttonAdd; // Кнопка для добавления урока

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // Инициализация интерфейса
        editTextSubject = findViewById(R.id.editTextSubject);
        editTextTime = findViewById(R.id.editTextTime);
        buttonAdd = findViewById(R.id.buttonAdd);

        // Инициализация списка и адаптера
        lessonList = new ArrayList<>();
        adapter = new LessonAdapter(lessonList);
        
        RecyclerView recyclerView = findViewById(R.id.recyclerView);
        recyclerView.setLayoutManager(new LinearLayoutManager(this));
        recyclerView.setAdapter(adapter);

        // Обработчик нажатия кнопки "Добавить урок"
        buttonAdd.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                String subject = editTextSubject.getText().toString().trim(); // Получаем название предмета
                String time = editTextTime.getText().toString().trim(); // Получаем время

                // Проверяем, что поля не пустые
                if (!subject.isEmpty() && !time.isEmpty()) {
                    Lesson lesson = new Lesson(subject, time); // Создаем новый урок
                    lessonList.add(lesson); // Добавляем его в список
                    adapter.notifyItemInserted(lessonList.size() - 1); // Уведомляем адаптер о добавлении элемента

                    // Очищаем поля ввода
                    editTextSubject.setText("");
                    editTextTime.setText("");
                }
            }
        });
    }
}


### Объяснение кода:

1. Объявление переменных:
    - lessonList — список для хранения объектов Lesson.
    - adapter — адаптер для RecyclerView, который будет управлять отображением данных.
    - editTextSubject и editTextTime — поля ввода для названия предмета и времени.
    - buttonAdd — кнопка для добавления нового урока.

2. Метод onCreate:
    - Устанавливает разметку для MainActivity (setContentView).
    - Инициализирует элементы интерфейса с помощью findViewById.
    - Создает новый экземпляр списка и адаптера.
    - Настраивает RecyclerView с помощью менеджера компоновки и адаптера.
    
3. Обработчик нажатия на кнопку:
    - При нажатии на кнопку извлекаются значения из полей ввода.
    - Проверяется, что оба поля не пустые. Если они заполнены, создается новый объект Lesson, добавляется в список и обновляется адаптер.
    - Поля ввода очищаются после добавления урока.

Не забудьте, что вы также должны создать LessonAdapter и Lesson классы, а также сделать соответствующую разметку для activity_main.xml и lesson_item.xml, как я описывал в предыдущем ответе. 

Также убедитесь, что в файле build.gradle у вас есть необходимые зависимости для работы с RecyclerView.
