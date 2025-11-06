using System;
using System.Windows.Forms;

namespace MatrixApp
{
    public partial class Form1 : Form
    {
        // Создаем матрицу 3x3
        int[,] A = new int[3, 3]
        {
            { 2, 3, 4 },
            { 1, 5, 6 },
            { 7, 8, 9 }
        };

        public Form1()
        {
            InitializeComponent();
        }

        private void Form1_Load(object sender, EventArgs e)
        {
            // Создаем элемент DataGridView для отображения матрицы
            DataGridView dgv = new DataGridView();
            dgv.RowCount = 3;
            dgv.ColumnCount = 3;
            dgv.Width = 250;
            dgv.Height = 120;
            dgv.Location = new System.Drawing.Point(20, 20);

            // Заполняем таблицу значениями матрицы
            for (int i = 0; i < 3; i++)
                for (int j = 0; j < 3; j++)
                    dgv[j, i].Value = A[i, j];

            Controls.Add(dgv);

            // Вычисляем сумму второй строки
            int sumSecondRow = 0;
            for (int j = 0; j < 3; j++)
                sumSecondRow += A[1, j];

            // Вычисляем произведение первого столбца
            int productFirstColumn = 1;
            for (int i = 0; i < 3; i++)
                productFirstColumn *= A[i, 0];

            // Выводим результаты
            Label lblResults = new Label();
            lblResults.AutoSize = true;
            lblResults.Location = new System.Drawing.Point(20, 160);
            lblResults.Text =
                $"Сумма второй строки = {sumSecondRow}\n" +
                $"Произведение первого столбца = {productFirstColumn}";
            Controls.Add(lblResults);
        }
    }
}
