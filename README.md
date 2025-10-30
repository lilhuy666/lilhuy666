
// Создайте проект Windows Forms в Visual Studio
// Добавьте на форму 4 TextBox для ввода a, b, c, d (textBoxA, textBoxB, textBoxC, textBoxD)
// Добавьте кнопку (buttonProcess)
// Добавьте 4 Label для вывода результатов (labelA, labelB, labelC, labelD)

using System;
using System.Windows.Forms;

namespace NumberProcessingApp
{
    public partial class Form1 : Form
    {
        public Form1()
        {
            InitializeComponent();
        }

        private void buttonProcess_Click(object sender, EventArgs e)
        {
            // Считываем значения и проверяем корректность ввода
            if (double.TryParse(textBoxA.Text, out double a) &&
                double.TryParse(textBoxB.Text, out double b) &&
                double.TryParse(textBoxC.Text, out double c) &&
                double.TryParse(textBoxD.Text, out double d))
            {
                // Условие 1: a ≤ b ≤ c ≤ d
                if (a <= b && b <= c && c <= d)
                {
                    double max = d; // d - самое большое число в этом случае
                    a = b = c = d = max;
                }
                // Условие 2: a > b > c > d
                else if (a > b && b > c && c > d)
                {
                    // Ничего не меняем
                }
                else
                {
                    // Иначе заменяем все числа их квадратами
                    a = a * a;
                    b = b * b;
                    c = c * c;
                    d = d * d;
                }

                // Вывод результатов в метки
                labelA.Text = $"a = {a}";
                labelB.Text = $"b = {b}";
                labelC.Text = $"c = {c}";
                labelD.Text = $"d = {d}";
            }
            else
            {
                MessageBox.Show("Введите корректные действительные числа для a, b, c, d.", "Ошибка ввода", MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }
    }
}


Коротко

- При a ≤ b ≤ c ≤ d все числа заменяются максимальным (d).
- При строго убывающей последовательности a > b > c > d ничего не меняется.
- В остальных случаях числа заменяются на их квадраты.

Подключите этот код к вашей форме, чтобы реализовать указанную логику.
