//Core/BaseViewModel.cs
using System.ComponentModel;
using System.Runtime.CompilerServices;

namespace SchoolApp.Core
{
    public abstract class BaseViewModel : INotifyPropertyChanged
    {
        public event PropertyChangedEventHandler PropertyChanged;
        protected void OnPropertyChanged([CallerMemberName] string name = null) =>
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));

        protected bool Set<T>(ref T field, T value, [CallerMemberName] string name = null)
        {
            if (Equals(field, value)) return false;
            field = value;
            OnPropertyChanged(name);
            return true;
        }
    }
}
//Core/RelayCommand.cs
using System;
using System.Windows.Input;

namespace SchoolApp.Core
{
    public class RelayCommand : ICommand
    {
        private readonly Action<object> _execute;
        private readonly Predicate<object> _canExecute;

        public RelayCommand(Action<object> execute, Predicate<object> canExecute = null)
        {
            _execute = execute ?? throw new ArgumentNullException(nameof(execute));
            _canExecute = canExecute;
        }

        public bool CanExecute(object parameter) => _canExecute?.Invoke(parameter) ?? true;
        public void Execute(object parameter) => _execute(parameter);
        public event EventHandler CanExecuteChanged
        {
            add => CommandManager.RequerySuggested += value;
            remove => CommandManager.RequerySuggested -= value;
        }
    }
}
//Models/Teacher.cs
namespace SchoolApp.Models
{
    public class Teacher
    {
        public int Id { get; set; }
        public string FullName { get; set; }
        public string Subject { get; set; }
        public int? ClassId { get; set; }
    }
}
//Models/Class.cs
namespace SchoolApp.Models
{
    public class Class
    {
        public int Id { get; set; }
        public string Number { get; set; }
        public int Floor { get; set; }
    }
}
//Models/TeacherClassView.cs
namespace SchoolApp.Models
{
    public class TeacherClassView
    {
        public int TeacherId { get; set; }
        public string TeacherFullName { get; set; }
        public string TeacherSubject { get; set; }
        public string ClassNumber { get; set; }
        public int ClassFloor { get; set; }
    }
}
//Services/DatabaseService.cs
 using Microsoft.Data.Sqlite;
using SchoolApp.Models;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

namespace SchoolApp.Services
{
    public class DatabaseService
    {
        private readonly string _connStr = "Data Source=school.db";

        public DatabaseService() => InitializeDb().Wait();

        private async Task InitializeDb()
        {
            using var conn = new SqliteConnection(_connStr);
            await conn.OpenAsync();
            string sql = @"
                CREATE TABLE IF NOT EXISTS Classes (
                    Id INTEGER PRIMARY KEY AUTOINCREMENT, Number TEXT NOT NULL, Floor INTEGER NOT NULL);
                CREATE TABLE IF NOT EXISTS Teachers (
                    Id INTEGER PRIMARY KEY AUTOINCREMENT, FullName TEXT NOT NULL, Subject TEXT NOT NULL, ClassId INTEGER,
                    FOREIGN KEY(ClassId) REFERENCES Classes(Id));";
            using var cmd = new SqliteCommand(sql, conn);
            await cmd.ExecuteNonQueryAsync();
        }

        public async Task<List<TeacherClassView>> GetPageAsync(int page, int pageSize, string sortColumn, bool asc, string search)
        {
            var list = new List<TeacherClassView>();
            using var conn = new SqliteConnection(_connStr);
            await conn.OpenAsync();

            var allowedSorts = new[] { "t.FullName", "t.Subject", "c.Number", "c.Floor" };
            string safeSort = allowedSorts.Contains(sortColumn) ? sortColumn : "t.FullName";
            string direction = asc ? "ASC" : "DESC";
            int offset = (page - 1) * pageSize;

            string sql = $@"
                SELECT t.Id, t.FullName, t.Subject, c.Number, c.Floor
                FROM Teachers t
                LEFT JOIN Classes c ON t.ClassId = c.Id
                WHERE (t.FullName LIKE @search OR t.Subject LIKE @search OR c.Number LIKE @search)
                ORDER BY {safeSort} {direction}
                LIMIT @limit OFFSET @offset;";

            using var cmd = new SqliteCommand(sql, conn);
            cmd.Parameters.AddWithValue("@search", $"%{search}%");
            cmd.Parameters.AddWithValue("@limit", pageSize);
            cmd.Parameters.AddWithValue("@offset", offset);

            using var reader = await cmd.ExecuteReaderAsync();
            while (await reader.ReadAsync())
            {
                list.Add(new TeacherClassView
                {
                    TeacherId = reader.GetInt32(0),
                    TeacherFullName = reader.GetString(1),
                    TeacherSubject = reader.GetString(2),
                    ClassNumber = reader.IsDBNull(3) ? "—" : reader.GetString(3),
                    ClassFloor = reader.IsDBNull(4) ? 0 : reader.GetInt32(4)
                });
            }
            return list;
        }

        public async Task<int> GetTotalCountAsync(string search)
        {
            using var conn = new SqliteConnection(_connStr);
            await conn.OpenAsync();
            using var cmd = new SqliteCommand("SELECT COUNT(*) FROM Teachers t LEFT JOIN Classes c ON t.ClassId = c.Id WHERE t.FullName LIKE @s OR t.Subject LIKE @s OR c.Number LIKE @s", conn);
            cmd.Parameters.AddWithValue("@s", $"%{search}%");
            return (int)(long)await cmd.ExecuteScalarAsync();
        }

        public async Task<List<Class>> GetClassesAsync()
        {
            var list = new List<Class>();
            using var conn = new SqliteConnection(_connStr);
            await conn.OpenAsync();
            using var cmd = new SqliteCommand("SELECT Id, Number, Floor FROM Classes ORDER BY Number", conn);
            using var r = await cmd.ExecuteReaderAsync();
            while (await r.ReadAsync()) list.Add(new Class { Id = r.GetInt32(0), Number = r.GetString(1), Floor = r.GetInt32(2) });
            return list;
        }

        public async Task<int?> FindClassIdAsync(string number)
        {
            using var conn = new SqliteConnection(_connStr);
            await conn.OpenAsync();
            using var cmd = new SqliteCommand("SELECT Id FROM Classes WHERE Number = @num", conn);
            cmd.Parameters.AddWithValue("@num", number);
            var res = await cmd.ExecuteScalarAsync();
            return res != null ? Convert.ToInt32(res) : null;
        }

        public async Task<int> InsertClassAsync(string number, int floor)
        {
            using var conn = new SqliteConnection(_connStr);
            await conn.OpenAsync();
            using var cmd = new SqliteCommand("INSERT INTO Classes (Number, Floor) VALUES (@num, @floor); SELECT last_insert_rowid();", conn);
            cmd.Parameters.AddWithValue("@num", number);
            cmd.Parameters.AddWithValue("@floor", floor);
            return Convert.ToInt32(await cmd.ExecuteScalarAsync());
        }

        public async Task InsertTeacherAsync(string name, string subject, int? classId)
        {
            using var conn = new SqliteConnection(_connStr);
            await conn.OpenAsync();
            using var cmd = new SqliteCommand("INSERT INTO Teachers (FullName, Subject, ClassId) VALUES (@n, @s, @cid)", conn);
            cmd.Parameters.AddWithValue("@n", name);
            cmd.Parameters.AddWithValue("@s", subject);
            cmd.Parameters.AddWithValue("@cid", classId.HasValue ? (object)classId.Value : DBNull.Value);
            await cmd.ExecuteNonQueryAsync();
        }

        public async Task UpdateTeacherAsync(int id, string name, string subject, int? classId)
        {
            using var conn = new SqliteConnection(_connStr);
            await conn.OpenAsync();
            using var cmd = new SqliteCommand("UPDATE Teachers SET FullName=@n, Subject=@s, ClassId=@cid WHERE Id=@id", conn);
            cmd.Parameters.AddWithValue("@id", id);
            cmd.Parameters.AddWithValue("@n", name);
            cmd.Parameters.AddWithValue("@s", subject);
            cmd.Parameters.AddWithValue("@cid", classId.HasValue ? (object)classId.Value : DBNull.Value);
            await cmd.ExecuteNonQueryAsync();
        }

        public async Task DeleteTeacherAsync(int id)
        {
            using var conn = new SqliteConnection(_connStr);
            await conn.OpenAsync();
            using var cmd = new SqliteCommand("DELETE FROM Teachers WHERE Id=@id", conn);
            cmd.Parameters.AddWithValue("@id", id);
            await cmd.ExecuteNonQueryAsync();
        }
    }
}
//ViewModels/MainViewModel.cs
using SchoolApp.Core;
using SchoolApp.Models;
using SchoolApp.Services;
using System.Collections.ObjectModel;
using System.Windows;

namespace SchoolApp.ViewModels
{
    public class MainViewModel : BaseViewModel
    {
        private readonly DatabaseService _db = new();
        public ObservableCollection<TeacherClassView> Items { get; } = new();

        private string _searchText;
        public string SearchText
        {
            get => _searchText;
            set { Set(ref _searchText, value); LoadData(); }
        }

        private string _selectedSort = "t.FullName";
        public string SelectedSort
        {
            get => _selectedSort;
            set { Set(ref _selectedSort, value); LoadData(); }
        }

        private int _currentPage = 1;
        public int CurrentPage
        {
            get => _currentPage;
            set { Set(ref _currentPage, value); LoadData(); }
        }

        private int _totalPages = 1;
        public int TotalPages
        {
            get => _totalPages;
            set { Set(ref _totalPages, value); }
        }

        private TeacherClassView _selectedItem;
        public TeacherClassView SelectedItem
        {
            get => _selectedItem;
            set { Set(ref _selectedItem, value); }
        }

        public int PageSize { get; set; } = 10;

        public RelayCommand AddCommand { get; }
        public RelayCommand EditCommand { get; }
        public RelayCommand DeleteCommand { get; }
        public RelayCommand NextPageCommand { get; }
        public RelayCommand PrevPageCommand { get; }

        public MainViewModel()
        {
            NextPageCommand = new RelayCommand(_ => { if (CurrentPage < TotalPages) CurrentPage++; }, _ => CurrentPage < TotalPages);
            PrevPageCommand = new RelayCommand(_ => { if (CurrentPage > 1) CurrentPage--; }, _ => CurrentPage > 1);
            AddCommand = new RelayCommand(async _ => await OpenDialog(null));
            EditCommand = new RelayCommand(async _ => await OpenDialog(SelectedItem), _ => SelectedItem != null);
            DeleteCommand = new RelayCommand(async _ =>
            {
                if (MessageBox.Show("Удалить запись?", "Подтверждение", MessageBoxButton.YesNo) == MessageBoxResult.Yes)
                {
                    await _db.DeleteTeacherAsync(SelectedItem.TeacherId);
                    LoadData();
                }
            }, _ => SelectedItem != null);

            LoadData();
        }

        private async void LoadData()
        {
            var total = await _db.GetTotalCountAsync(SearchText);
            TotalPages = System.Math.Max(1, (int)System.Math.Ceiling((double)total / PageSize));
            if (CurrentPage > TotalPages) CurrentPage = TotalPages;

            var pageData = await _db.GetPageAsync(CurrentPage, PageSize, SelectedSort, true, SearchText);
            Items.Clear();
            foreach (var item in pageData) Items.Add(item);
        }

        private async Task OpenDialog(TeacherClassView item)
        {
            var dlg = new EditWindow(item?.TeacherId);
            dlg.ShowDialog();
            if (dlg.DialogResult == true) LoadData();
        }
    }
}
//ViewModels/EditViewModel.cs
using SchoolApp.Core;
using SchoolApp.Models;
using SchoolApp.Services;
using System.Collections.ObjectModel;
using System.Linq;
using System.Windows;

namespace SchoolApp.ViewModels
{
    public class EditViewModel : BaseViewModel
    {
        private readonly DatabaseService _db = new();
        public int? TeacherId { get; }

        private string _fullName;
        public string FullName { get => _fullName; set => Set(ref _fullName, value); }

        private string _subject;
        public string Subject { get => _subject; set => Set(ref _subject, value); }

        private ObservableCollection<Class> _classes = new();
        public ObservableCollection<Class> Classes { get => _classes; set => Set(ref _classes, value); }

        private Class _selectedClass;
        public Class SelectedClass
        {
            get => _selectedClass;
            set => Set(ref _selectedClass, value);
        }

        private string _newClassNumber;
        public string NewClassNumber
        {
            get => _newClassNumber;
            set { Set(ref _newClassNumber, value); SearchExistingClass(); }
        }

        private int _newClassFloor = 1;
        public int NewClassFloor { get => _newClassFloor; set => Set(ref _newClassFloor, value); }

        private bool _isCreatingNewClass;
        public bool IsCreatingNewClass
        {
            get => _isCreatingNewClass;
            set => Set(ref _isCreatingNewClass, value);
        }

        public RelayCommand SaveCommand { get; }
        public RelayCommand CancelCommand { get; }

        public EditViewModel(int? teacherId)
        {
            TeacherId = teacherId;
            LoadClasses();
            if (teacherId.HasValue) LoadTeacher();
            else IsCreatingNewClass = true;

            SaveCommand = new RelayCommand(async _ => await SaveAsync());
            CancelCommand = new RelayCommand(_ => Application.Current.Windows.OfType<Window>().FirstOrDefault(w => w.DataContext == this)?.DialogResult = false);
        }

        private async void LoadClasses()
        {
            var list = await _db.GetClassesAsync();
            Classes.Clear();
            foreach (var c in list) Classes.Add(c);
        }

        private async void SearchExistingClass()
        {
            if (string.IsNullOrWhiteSpace(NewClassNumber)) { IsCreatingNewClass = false; return; }
            var id = await _db.FindClassIdAsync(NewClassNumber);
            if (id.HasValue)
            {
                SelectedClass = Classes.First(c => c.Id == id.Value);
                IsCreatingNewClass = false;
            }
            else IsCreatingNewClass = true;
        }

        private async void LoadTeacher()
        {
            var item = (await _db.GetPageAsync(1, 100, "t.Id", true, ""))
                .FirstOrDefault(t => t.TeacherId == TeacherId);
            if (item != null)
            {
                FullName = item.TeacherFullName;
                Subject = item.TeacherSubject;
                SelectedClass = Classes.FirstOrDefault(c => c.Number == item.ClassNumber);
                if (SelectedClass != null) IsCreatingNewClass = false;
            }
        }

        private async Task SaveAsync()
        {
            if (string.IsNullOrWhiteSpace(FullName) || string.IsNullOrWhiteSpace(Subject))
            {
                MessageBox.Show("Заполните ФИО и Предмет");
                return;
            }

            int? classId = null;
            if (!IsCreatingNewClass && SelectedClass != null)
                classId = SelectedClass.Id;
            else if (IsCreatingNewClass && !string.IsNullOrWhiteSpace(NewClassNumber))
            {
                classId = await _db.InsertClassAsync(NewClassNumber, NewClassFloor);
                Classes.Add(new Class { Id = classId.Value, Number = NewClassNumber, Floor = NewClassFloor });
            }

            if (TeacherId.HasValue)
                await _db.UpdateTeacherAsync(TeacherId.Value, FullName, Subject, classId);
            else
                await _db.InsertTeacherAsync(FullName, Subject, classId);

            Application.Current.Windows.OfType<Window>().FirstOrDefault(w => w.DataContext == this)?.DialogResult = true;
        }
    }
}
//InverseBooleanConverter.cs
using System;
using System.Globalization;
using System.Windows.Data;

namespace SchoolApp
{
    public class InverseBooleanConverter : IValueConverter
    {
        public object Convert(object value, Type targetType, object parameter, CultureInfo culture) => !(bool)value;
        public object ConvertBack(object value, Type targetType, object parameter, CultureInfo culture) => throw new NotImplementedException();
    }
}
//MainWindow.xaml
<Window x:Class="SchoolApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:local="clr-namespace:SchoolApp"
        Title="Школа: Учителя и Классы" Height="480" Width="800" WindowStartupLocation="CenterScreen">
    <Window.Resources>
        <local:InverseBooleanConverter x:Key="InverseBool"/>
    </Window.Resources>
    <Grid Margin="10">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <Menu Grid.Row="0" Height="30">
            <MenuItem Header="Записи">
                <MenuItem Header="Добавить" Command="{Binding AddCommand}"/>
                <MenuItem Header="Редактировать" Command="{Binding EditCommand}"/>
                <MenuItem Header="Удалить" Command="{Binding DeleteCommand}"/>
            </MenuItem>
        </Menu>

        <StackPanel Grid.Row="0" Orientation="Horizontal" Margin="0,35,0,5">
            <TextBlock Text="Поиск:" VerticalAlignment="Center" Margin="5,0,5,0"/>
            <TextBox Width="200" Text="{Binding SearchText, UpdateSourceTrigger=PropertyChanged}"/>
            <TextBlock Text="Сортировка:" VerticalAlignment="Center" Margin="10,0,5,0"/>
            <ComboBox Width="180" SelectedValuePath="Content" SelectedValue="{Binding SelectedSort}">
                <ComboBoxItem Content="t.FullName">ФИО</ComboBoxItem>
                <ComboBoxItem Content="t.Subject">Предмет</ComboBoxItem>
                <ComboBoxItem Content="c.Number">Класс</ComboBoxItem>
                <ComboBoxItem Content="c.Floor">Этаж</ComboBoxItem>
            </ComboBox>
        </StackPanel>

        <DataGrid Grid.Row="1" ItemsSource="{Binding Items}" SelectedItem="{Binding SelectedItem}"
                  AutoGenerateColumns="False" IsReadOnly="True" Margin="0,5">
            <DataGrid.Columns>
                <DataGridTextColumn Header="ФИО" Binding="{Binding TeacherFullName}" Width="*"/>
                <DataGridTextColumn Header="Предмет" Binding="{Binding TeacherSubject}" Width="*"/>
                <DataGridTextColumn Header="Класс" Binding="{Binding ClassNumber}" Width="100"/>
                <DataGridTextColumn Header="Этаж" Binding="{Binding ClassFloor}" Width="80"/>
            </DataGrid.Columns>
            <DataGrid.ContextMenu>
                <ContextMenu>
                    <MenuItem Header="Редактировать" Command="{Binding EditCommand}"/>
                    <MenuItem Header="Удалить" Command="{Binding DeleteCommand}"/>
                </ContextMenu>
            </DataGrid.ContextMenu>
        </DataGrid>

        <StackPanel Grid.Row="2" Orientation="Horizontal" HorizontalAlignment="Center" Margin="0,5">
            <Button Content="◀" Command="{Binding PrevPageCommand}" Margin="5"/>
            <TextBlock Text="{Binding CurrentPage, StringFormat=Страница {0} из {1}}" Margin="10" VerticalAlignment="Center"/>
            <Button Content="▶" Command="{Binding NextPageCommand}" Margin="5"/>
        </StackPanel>
    </Grid>
</Window>
//MainWindow.xaml.cs
using SchoolApp.ViewModels;
using System.Windows;

namespace SchoolApp
{
    public partial class MainWindow : Window
    {
        public MainWindow()
        {
            InitializeComponent();
            DataContext = new MainViewModel();
        }
    }
}
//EditWindow.xaml
<Window x:Class="SchoolApp.EditWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Учитель" Height="280" Width="380" WindowStartupLocation="CenterOwner">
    <Grid Margin="15">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
        </Grid.RowDefinitions>

        <TextBlock Text="ФИО:" Grid.Row="0"/>
        <TextBox Grid.Row="0" Margin="0,20,0,0" Text="{Binding FullName}"/>

        <TextBlock Text="Предмет:" Grid.Row="1" Margin="0,10,0,5"/>
        <TextBox Grid.Row="1" Margin="0,25,0,0" Text="{Binding Subject}"/>

        <TextBlock Text="Класс:" Grid.Row="2" Margin="0,10,0,5"/>
        <ComboBox Grid.Row="2" Margin="0,25,0,0" ItemsSource="{Binding Classes}"
                  DisplayMemberPath="Number" SelectedItem="{Binding SelectedClass}"
                  IsEnabled="{Binding IsCreatingNewClass, Converter={StaticResource InverseBool}}"/>
        
        <StackPanel Grid.Row="3" Orientation="Horizontal" Margin="0,10">
            <CheckBox Content="Создать новый класс" IsChecked="{Binding IsCreatingNewClass}" VerticalAlignment="Center" Margin="0,0,10,0"/>
            <TextBox Width="60" Text="{Binding NewClassNumber}" IsEnabled="{Binding IsCreatingNewClass}" VerticalAlignment="Center"/>
            <TextBlock Text="Этаж:" VerticalAlignment="Center" Margin="5,0"/>
            <TextBox Width="40" Text="{Binding NewClassFloor}" IsEnabled="{Binding IsCreatingNewClass}" VerticalAlignment="Center"/>
        </StackPanel>

        <StackPanel Grid.Row="4" Orientation="Horizontal" HorizontalAlignment="Right" Margin="0,15,0,0">
            <Button Content="Сохранить" Command="{Binding SaveCommand}" Width="80" Margin="0,0,10,0" IsDefault="True"/>
            <Button Content="Отмена" Command="{Binding CancelCommand}" Width="80" IsCancel="True"/>
        </StackPanel>
    </Grid>
</Window>
//EditWindow.xaml.cs
using SchoolApp.ViewModels;
using System.Windows;

namespace SchoolApp
{
    public partial class EditWindow : Window
    {
        public EditWindow(int? teacherId = null)
        {
            InitializeComponent();
            DataContext = new EditViewModel(teacherId);
        }
    }
}ss
