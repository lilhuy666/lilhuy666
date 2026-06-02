1 Модуль: отредактируй таблицы, импартируй их, нарисуй бд, напиши отчёт(сохранять в пдф). 
Алгоритм разработки приложения(сохрани в пдф):
                                                                Начало (в овале)
                                                                Анализ требований и документов предоставленных заказчиком, для проектирования и создания БД(в прямоугольнику)
                                                                Проектирование ER диаграммы в 3 НФ (в прямоугольнике)
Данные предоставленные заказчиком(в бочке без дна)              Создание БД и импорт данных (в прямоугольнике)
                                                                Создание БД и импорт данных выполнен успешно (в ромбе) (соедине со стрелкой идущей изз начала)
                                                                Анализ требований к разработке приложения - оформление кода - руководство по стилю - требования к функционалу                                                                 БД - требования к функционалу приложения.(в прямоугольнике)
                                                                Проектирование архитектуры приложения(в прямоугольнике)
                                                                Создание пустого приложения(в прямоугольнике)
База данных(в бочке)                                            Подключение БД к приложению. Создание контекста данных и моделей данных(в прямоугольнике)
                                                                Разработка модуля авторизации(в прямоугольнике)
                                                                Разработка модуля "Список товаров"(в прямоугольнике)
                                                                Реализация логики отображения: - заглушка для отсутствующих изображений - подсветка строк по скидкам >15% -                                                                   выделение соответствующих товаров(в прямоугольнике)
                                                                Разработка интерфейсов для всех ролей(в прямоугольнике)
                                                                Выполнение отладки приложения(в прямоугольнике)
                                                                Составление документации со скриншотами(в прямоугольнике)
                                                                Конец(в овале)
                                                                
                
2 Модуль: 
1.Удали маин виндоус, создай папки Codes, Pages, Resources, Windows. В папке Windows создай окно Desktop. Импартируй рисунки из задания в парку Resources.
Выдели экран приловения и в свойствах замени иконку.
Код для App: 
<Application x:Class="WpfApp1.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:local="clr-namespace:WpfApp1"
             StartupUri="Windows/Desktop.xaml">
    <Application.Resources>
        <Style x:Key="MainBtn" TargetType="Button"
            <Setter Property="Width" Value="200"/>
            <Setter Property="Height" Value="30"/>
            <Setter Property="Background" Value="#00FA9A"/>
            <Setter Property="Foreground" Value="#FFF"/>
            <Setter Property="FontFamily" Value="Times New Roman"/>
            <Setter Property="FontSize" Value="17"/>
            <Setter Property="Margin" Value="2"/>
        </Style>
        <Style x:Key="MainLbl" TargetType="Label">
            <Setter Property="Width" Value="200"/>
            <Setter Property="Height" Value="30"/>
            <Setter Property="FontFamily" Value="Times New Roman"/>
            <Setter Property="FontSize" Value="17"/>
            <Setter Property="Margin" Value="2"/>
        </Style>
    </Application.Resources>
</Application>


Код для Desktop:
<Window x:Class="WpfApp1.Windows.Desktop"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        xmlns:local="clr-namespace:WpfApp1.Windows"
        mc:Ignorable="d"
        Title="ООО Обувь" MinHeight="900" MinWidth="1140" MaxHeight="900" MaxWidth="1140" Icon="/Windows/Icon.png" WindowStartupLocation="CenterScreen">
    <Grid HorizontalAlignment="Right" VerticalAlignment="Top">
        <Grid.RowDefinitions>
            <RowDefinition Height="159*"/>
            <RowDefinition Height="725*"/>
        </Grid.RowDefinitions>
        <Image Grid.Row="0" Margin="10,10,989,8" Source="/Resources/Icon.JPG" Stretch="Fill" RenderTransformOrigin="0.5,0.5"/>
        <Button x:Name="BackBtn" Content="Назад" Click="BackBtn_Click" Style="{StaticResource MainBtn}" Margin="666,102,0,27" Background="#7FF000" HorizontalAlignment="Left" Width="200"/>
        <Button x:Name="ToDesktopBtn" Content="На главную" Click="ToDesktopBtn_Click" Style="{StaticResource MainBtn}" Margin="0,102,25,27" HorizontalAlignment="Right" Width="200"/>
        <Label x:Name="LoginInfo" Grid.Row="0" Style="{StaticResource MainLbl}" Margin="715,33,25,96" Width="400"/>
        <Label Content="000 Обувь" Grid.Row="0" FontSize="40" FontWeight="Bold" FontFamily="Times New Roman" Margin="180,54,675,35"/>
        <Frame x:Name="DesktopFrame" Grid.Row="1" NavigationUIVisibility="Hidden" VerticalAlignment="Center" HorizontalAlignment="Center"/>
    </Grid>
</Window>




Код с окном входа в систему:
<Window x:Class="WpfApp1.Windows.Desktop"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        mc:Ignorable="d"
        Title="ООО Обувь" 
        MinHeight="900" MinWidth="1140" 
        MaxHeight="900" MaxWidth="1140" 
        Icon="/WpfApp1;component/Windows/Icon.png" 
        WindowStartupLocation="CenterScreen">

    <Grid>
        <!-- 1. ОПРЕДЕЛЕНИЕ СТРОК ДЛЯ ГЛАВНОЙ СЕТКИ -->
        <Grid.RowDefinitions>
            <RowDefinition Height="159*"/>
            <!-- Для верхней панели -->
            <RowDefinition Height="725*"/>
            <!-- Для зоны входа и фрейма -->
        </Grid.RowDefinitions>

        <!-- Верхняя часть (Логотип, кнопки, информация) -->
        <Grid Grid.Row="0">
            <Image Source="/WpfApp1;component/Resources/Icon.JPG" Stretch="Fill" Margin="23,6,966,17"/>

            <StackPanel Orientation="Horizontal" HorizontalAlignment="Right" VerticalAlignment="Top">
                <Button x:Name="BackBtn" Content="Назад" Click="BackBtn_Click" Style="{StaticResource MainBtn}" Margin="0,102,25,27" Width="200"/>
                <Button x:Name="ToDesktopBtn" Content="На главную" Click="ToDesktopBtn_Click" Style="{StaticResource MainBtn}" Margin="25,102,25,27" Width="200"/>
            </StackPanel>

            <StackPanel Orientation="Horizontal" HorizontalAlignment="Left">
                <Label x:Name="LoginInfo" Style="{StaticResource MainLbl}" Margin="715,33,25,96" Width="400"/>
                <Label Content="ООО Обувь" FontSize="40" FontWeight="Bold" FontFamily="Times New Roman" Margin="180,54,675,35"/>
            </StackPanel>
        </Grid>

        <!-- Нижняя часть (Зона входа или основной фрейм) -->
        <Grid Grid.Row="1">
            <!-- Зона входа в систему (по умолчанию видима) -->
            <Border x:Name="LoginZone"
                    Background="#AAFFFFFF"
                    CornerRadius="15"
                    Padding="30"
                    VerticalAlignment="Center"
                    HorizontalAlignment="Center">

                <StackPanel HorizontalAlignment="Center">
                    <TextBlock Text="Выполнение входа в систему"
                               FontSize="24"
                               FontWeight="Bold"
                               Margin="0,0,0,20"
                               HorizontalAlignment="Center"/>

                    <Label Content="Логин:" Margin="0,10,0,5"/>
                    <TextBox x:Name="LoginBox" Width="300" Height="35" Margin="0,0,0,15"/>

                    <Label Content="Пароль:" Margin="0,10,0,5"/>
                    <PasswordBox x:Name="PasswordBox" Width="300" Height="35" Margin="0,0,0,25"/>

                    <StackPanel Orientation="Horizontal" HorizontalAlignment="Center">
                        <Button x:Name="LoginBtn"
                                Content="Войти"
                                Width="120"
                                Height="40"
                                Margin="0,0,15,0"
                                Click="LoginBtn_Click"/>
                        <Button x:Name="GuestBtn"
                                Content="Войти как гость"
                                Width="180"
                                Height="40"
                                Click="GuestBtn_Click"/>
                    </StackPanel>
                </StackPanel>
            </Border>

            <!-- Основной фрейм приложения (по умолчанию скрыт) -->
            <Frame x:Name="DesktopFrame"
                   NavigationUIVisibility="Hidden"
                   VerticalAlignment="Center"
                   HorizontalAlignment="Center"
                   Visibility="Collapsed"/>
        </Grid>
    </Grid>
</Window>
