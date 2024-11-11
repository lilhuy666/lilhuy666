Для создания инструментированных тестов в Android вы можете использовать фреймворк Espresso, который позволяет писать тесты пользовательского интерфейса. Ниже приведён пример кода для запуска инструментированного теста в вашем приложении.

### Пример инструментарного теста с использованием Espresso

Создайте новый класс теста в каталоге src/androidTest/java/ваш_пакет/:

package com.example.raspisanieurokov; // Замените на ваш пакет

import androidx.test.ext.junit.runners.AndroidJUnit4;
import androidx.test.rule.ActivityTestRule;

import org.junit.Rule;
import org.junit.Test;
import org.junit.runner.RunWith;

import static androidx.test.espresso.Espresso.onView;
import static androidx.test.espresso.action.ViewActions.click;
import static androidx.test.espresso.action.ViewActions.typeText;
import static androidx.test.espresso.assertion.ViewAssertions.matches;
import static androidx.test.espresso.matcher.ViewMatchers.withId;
import static androidx.test.espresso.matcher.ViewMatchers.withText;

@RunWith(AndroidJUnit4.class)
public class ExampleInstrumentedTest {

    @Rule
    public ActivityTestRule<MainActivity> activityRule =
            new ActivityTestRule<>(MainActivity.class);

    @Test
    public void testButtonClick() {
        // Ввод текста в EditText
        onView(withId(R.id.editTextName)) // Замените на ID вашего EditText
                .perform(typeText("Hello, World!"));

        // Нажатие на кнопку
        onView(withId(R.id.buttonSubmit)) // Замените на ID вашей кнопки
                .perform(click());

        // Проверка текста на TextView
        onView(withId(R.id.textViewResult)) // Замените на ID вашего TextView
                .check(matches(withText("Hello, World!"))); // Ожидаемый результат
    }
}


### Описание кода:

1. Импорт необходимых классов: Импортируются классы, необходимые для написания теста и использования Espresso.

2. Анотация @RunWith: Указывает, что тест будет запускаться с использованием JUnit4.

3. ActivityTestRule: Это специальный класс, который управляет жизненным циклом активности, позволяя запускать тесты в контексте MainActivity.

4. Тест: 
   - onView(withId(R.id.editTextName)): Находит элемент EditText по его ID и вводит текст "Hello, World!".
   - onView(withId(R.id.buttonSubmit)): Находит кнопку по её ID и выполняет клик.
   - onView(withId(R.id.textViewResult)): Находит элемент TextView и проверяет, соответствует ли его текст ожидаемому.

### Замечания:

- Не забудьте заменить R.id.editTextName, R.id.buttonSubmit и R.id.textViewResult на реальные идентификаторы ваших элементов пользовательского интерфейса.
- Убедитесь, что у вас установлены необходимые зависимости для Espresso в вашем файле build.gradle:
  
androidTestImplementation 'androidx.test.espresso:espresso-core:3.4.0'
androidTestImplementation 'androidx.test.ext:junit:1.1.3'


Этот пример является базовым и может быть адаптирован в зависимости от функциональности вашего приложения.
