using System;
using System.Windows.Forms;

namespace WinFormsApp1
{
    public partial class Form1 : Form
    {
        public Form1()
        {
            InitializeComponent();
        }

        private void button1_Click(object sender, EventArgs e)
        {
            // Проверяем ввод данных
            if (!double.TryParse(textBox1.Text, out double a) ||
                !double.TryParse(textBox2.Text, out double b) ||
                !double.TryParse(textBox3.Text, out double c) ||
                !double.TryParse(textBox4.Text, out double d))
            {
                MessageBox.Show("Введите корректные числовые значения для a, b, c, d!",
                                "Ошибка ввода", MessageBoxButtons.OK, MessageBoxIcon.Warning);
                return;
            }

            // Логика из условия
            if (a <= b && b <= c && c <= d)
            {
                // Находим наибольшее
                double max = Math.Max(Math.Max(a, b), Math.Max(c, d));

                a = b = c = d = max;
            }
            else if (a > b && b > c && c > d)
            {
                // Оставляем без изменений
            }
            else
            {
                // Заменяем квадратами
                a = Math.Pow(a, 2);
                b = Math.Pow(b, 2);
                c = Math.Pow(c, 2);
                d = Math.Pow(d, 2);
            }

            // Вывод результата
            label5.Text = $"a = {a:F3}";
            label6.Text = $"b = {b:F3}";
            label7.Text = $"c = {c:F3}";
            label8.Text = $"d = {d:F3}";
        }
    }
}
