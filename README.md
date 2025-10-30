using System;
using System.Windows.Forms;

namespace NumberTransform
{
    public partial class Form1 : Form
    {
        public Form1()
        {
            InitializeComponent();
        }

        private void button1_Click(object sender, EventArgs e)
        {
            try
            {
                double a = double.Parse(textBoxA.Text);
                double b = double.Parse(textBoxB.Text);
                double c = double.Parse(textBoxC.Text);
                double d = double.Parse(textBoxD.Text);

                string resultText = "";

                if (a <= b && b <= c && c <= d)
                {
                    double max = Math.Max(Math.Max(a, b), Math.Max(c, d));
                    a = b = c = d = max;
                    resultText = "Все числа заменены на наибольшее.";
                }
                else if (a > b && b > c && c > d)
                {
                    resultText = "Числа остались без изменений.";
                }
                else
                {
                    a = a * a;
                    b = b * b;
                    c = c * c;
                    d = d * d;
                    resultText = "Все числа заменены их квадратами.";
                }

                textBoxA.Text = a.ToString();
                textBoxB.Text = b.ToString();
                textBoxC.Text = c.ToString();
                textBoxD.Text = d.ToString();

                labelResult.Text = resultText;
            }
            catch
            {
                MessageBox.Show("Введите корректные числа!");
            }
        }
    }
}
