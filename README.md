using System;
using System.Linq;
using System.Windows.Forms;

namespace NumberReplacement
{
    public partial class Form1 : Form
    {
        public Form1()
        {
            InitializeComponent();
        }

        private void buttonCalculate_Click(object sender, EventArgs e)
        {
            try
            {
                double a = double.Parse(textBoxA.Text);
                double b = double.Parse(textBoxB.Text);
                double c = double.Parse(textBoxC.Text);
                double d = double.Parse(textBoxD.Text);

                double[] numbers = { a, b, c, d };

                // Проверяем условия
                if (a <= b && b <= c && c >= d)
                {
                    double max = numbers.Max();
                    a = b = c = d = max;
                }
                else if (a > b && b > c && c > d)
                {
                    // Оставляем без изменений
                }
                else
                {
                    a = Math.Pow(a, 2);
                    b = Math.Pow(b, 2);
                    c = Math.Pow(c, 2);
                    d = Math.Pow(d, 2);
                }

                // Выводим результат
                textBoxResult.Text = $"a = {a}, b = {b}, c = {c}, d = {d}";
            }
            catch
            {
                MessageBox.Show("Введите корректные числа!", "Ошибка ввода", MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }
    }
}
