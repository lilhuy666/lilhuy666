using System;
using System.Windows.Forms;

namespace NumberTransformApp
{
    public partial class Form1 : Form
    {
        public Form1()
        {
            InitializeComponent();
        }

        private void buttonExecute_Click(object sender, EventArgs e)
        {
            try
            {
                double a = double.Parse(textBoxA.Text);
                double b = double.Parse(textBoxB.Text);
                double c = double.Parse(textBoxC.Text);
                double d = double.Parse(textBoxD.Text);

                string resultMessage;

                if (a <= b && b <= c && c <= d)
                {
                    double max = Math.Max(Math.Max(a, b), Math.Max(c, d));
                    a = b = c = d = max;
                    resultMessage = "Все числа заменены на наибольшее.";
                }
                else if (a > b && b > c && c > d)
                {
                    resultMessage = "Числа остались без изменений.";
                }
                else
                {
                    a *= a;
                    b *= b;
                    c *= c;
                    d *= d;
                    resultMessage = "Все числа заменены их квадратами.";
                }

                // Обновляем текстовые поля
                textBoxA.Text = a.ToString();
                textBoxB.Text = b.ToString();
                textBoxC.Text = c.ToString();
                textBoxD.Text = d.ToString();

                // Показываем результат
                labelResult.Text = resultMessage;
            }
            catch
            {
                MessageBox.Show("Введите корректные числа!", "Ошибка", MessageBoxButtons.OK, MessageBoxIcon.Error);
            }
        }
    }
}
