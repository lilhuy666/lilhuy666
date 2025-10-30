namespace NumberProcessing
{
    partial class Form1
    {
        private System.ComponentModel.IContainer components = null;

        protected override void Dispose(bool disposing)
        {
            if (disposing && (components != null))
            {
                components.Dispose();
            }
            base.Dispose(disposing);
        }

        #region Код формы

        private void InitializeComponent()
        {
            this.labelA = new System.Windows.Forms.Label();
            this.labelB = new System.Windows.Forms.Label();
            this.labelC = new System.Windows.Forms.Label();
            this.labelD = new System.Windows.Forms.Label();
            this.textBoxA = new System.Windows.Forms.TextBox();
            this.textBoxB = new System.Windows.Forms.TextBox();
            this.textBoxC = new System.Windows.Forms.TextBox();
            this.textBoxD = new System.Windows.Forms.TextBox();
            this.buttonCalculate = new System.Windows.Forms.Button();
            this.labelResult = new System.Windows.Forms.Label();
            this.SuspendLayout();
            // 
            // labelA
            // 
            this.labelA.AutoSize = true;
            this.labelA.Location = new System.Drawing.Point(30, 30);
            this.labelA.Name = "labelA";
            this.labelA.Size = new System.Drawing.Size(20, 15);
            this.labelA.TabIndex = 0;
            this.labelA.Text = "A:";
            // 
            // labelB
            // 
            this.labelB.AutoSize = true;
            this.labelB.Location = new System.Drawing.Point(30, 70);
            this.labelB.Name = "labelB";
            this.labelB.Size = new System.Drawing.Size(19, 15);
            this.labelB.TabIndex = 1;
            this.labelB.Text = "B:";
            // 
            // labelC
            // 
            this.labelC.AutoSize = true;
            this.labelC.Location = new System.Drawing.Point(30, 110);
            this.labelC.Name = "labelC";
            this.labelC.Size = new System.Drawing.Size(20, 15);
            this.labelC.TabIndex = 2;
            this.labelC.Text = "C:";
            // 
            // labelD
            // 
            this.labelD.AutoSize = true;
            this.labelD.Location = new System.Drawing.Point(30, 150);
            this.labelD.Name = "labelD";
            this.labelD.Size = new System.Drawing.Size(21, 15);
            this.labelD.TabIndex = 3;
            this.labelD.Text = "D:";
            // 
            // textBoxA
            // 
            this.textBoxA.Location = new System.Drawing.Point(70, 27);
            this.textBoxA.Name = "textBoxA";
            this.textBoxA.Size = new System.Drawing.Size(100, 23);
            this.textBoxA.TabIndex = 4;
            // 
            // textBoxB
            // 
            this.textBoxB.Location = new System.Drawing.Point(70, 67);
            this.textBoxB.Name = "textBoxB";
            this.textBoxB.Size = new System.Drawing.Size(100, 23);
            this.textBoxB.TabIndex = 5;
            // 
            // textBoxC
            // 
            this.textBoxC.Location = new System.Drawing.Point(70, 107);
            this.textBoxC.Name = "textBoxC";
            this.textBoxC.Size = new System.Drawing.Size(100, 23);
            this.textBoxC.TabIndex = 6;
            // 
            // textBoxD
            // 
            this.textBoxD.Location = new System.Drawing.Point(70, 147);
            this.textBoxD.Name = "textBoxD";
            this.textBoxD.Size = new System.Drawing.Size(100, 23);
            this.textBoxD.TabIndex = 7;
            // 
            // buttonCalculate
            // 
            this.buttonCalculate.Location = new System.Drawing.Point(30, 190);
            this.buttonCalculate.Name = "buttonCalculate";
            this.buttonCalculate.Size = new System.Drawing.Size(140, 30);
            this.buttonCalculate.TabIndex = 8;
            this.buttonCalculate.Text = "Выполнить";
            this.buttonCalculate.UseVisualStyleBackColor = true;
            this.buttonCalculate.Click += new System.EventHandler(this.buttonCalculate_Click);
            // 
            // labelResult
            // 
            this.labelResult.AutoSize = true;
            this.labelResult.Font = new System.Drawing.Font("Segoe UI", 9F, System.Drawing.FontStyle.Bold, System.Drawing.GraphicsUnit.Point);
            this.labelResult.ForeColor = System.Drawing.Color.Blue;
            this.labelResult.Location = new System.Drawing.Point(30, 240);
            this.labelResult.Name = "labelResult";
            this.labelResult.Size = new System.Drawing.Size(0, 15);
            this.labelResult.TabIndex = 9;
            // 
            // Form1
            // 
            this.ClientSize = new System.Drawing.Size(250, 300);
            this.Controls.Add(this.labelResult);
            this.Controls.Add(this.buttonCalculate);
            this.Controls.Add(this.textBoxD);
            this.Controls.Add(this.textBoxC);
            this.Controls.Add(this.textBoxB);
            this.Controls.Add(this.textBoxA);
            this.Controls.Add(this.labelD);
            this.Controls.Add(this.labelC);
            this.Controls.Add(this.labelB);
            this.Controls.Add(this.labelA);
            this.Name = "Form1";
            this.Text = "Преобразование чисел";
            this.ResumeLayout(false);
            this.PerformLayout();
        }

        #endregion

        private System.Windows.Forms.Label labelA;
        private System.Windows.Forms.Label labelB;
        private System.Windows.Forms.Label labelC;
        private System.Windows.Forms.Label labelD;
        private System.Windows.Forms.TextBox textBoxA;
        private System.Windows.Forms.TextBox textBoxB;
        private System.Windows.Forms.TextBox textBoxC;
        private System.Windows.Forms.TextBox textBoxD;
        private System.Windows.Forms.Button buttonCalculate;
        private System.Windows.Forms.Label labelResult;
    }
}
