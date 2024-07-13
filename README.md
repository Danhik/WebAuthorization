# WebAuthorization and Registration with Asp.Net Core 
## Начало работы
Необходимо создать проект ASP.NET Core Web App (Model-View-Controller).
<br> Подробней о создании проекта здесь: [METANIT.COM](https://metanit.com/sharp/aspnet5/3.1.php)

<br>Также необходимо установить нужные для работы пакеты Nuget: 
***<br> Microsoft.EntityFramework
<br> Microsoft.EntityFrameworkCore.SqlServer
<br> Microsoft.EntityFrameworkCore.Tools***
## Опредление моделей классов
В папке Models  необходимо создать класс User: 
<br>
```csharp
{
    public enum UserRole
    {
        Admin,
        User
    }

    public class User
    {
        public int Id { get; set; }
        public string UserName { get; set; }
        public string Password { get; set; }
        public UserRole Role { get; set; }
    }
}
```
Для того чтобы можно было взимодействовать с БД
 в папке Models необходимо создать контекст данных UsersContext  

```csharp
using Microsoft.EntityFrameworkCore;
using System.Collections.Generic;

namespace WebApplication3.Models
{
    public class UserContext : DbContext
    {
        public DbSet<User> Users { get; set; }
        public UserContext(DbContextOptions<UserContext> options)
            : base(options)
        {
            Database.EnsureCreated();
        }
    }
}
```
```csharp Database.EnsureCreated(); ```
Эта строчка  используется для автоматического создания таблиц в вашей БД по заданным полям из класса User. 

## Базовые настройки для работы 
<br> В файле конфигурации appsettings.json необходимо определить строку для подключения к БД, которая указывается вместо "СТРОКА ПОДКЛЮЧЕНИЯ".
<br> Ее можно получить нажав на свойства Бд, и скопировать из ![image](https://github.com/user-attachments/assets/4db4015f-f51a-43e1-a89e-35a77f8c44f8)



```json
{
  "ConnectionStrings": {
    "DefaultConnection": "СТРОКА ПОДКЛЮЧЕНИЯ"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "AllowedHosts": "*"
}
```

<br> После этого необходимо отредактировать класс Program.cs для работы с авторизацией и аунтификацией, например на такой: 

```csharp
using WebApplication3.Models; // Подключение пространства имен контекста данных UserContext
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authentication.Cookies; // Подключение пространства имен для аутентификации с использованием куки
using Microsoft.AspNetCore.Builder; // Подключение пространства имен для создания и настройки веб-приложения
using Microsoft.EntityFrameworkCore; // Подключение Entity Framework Core для работы с базой данных
using Microsoft.Extensions.Configuration; // Подключение пространства имен для работы с конфигурацией приложения
using Microsoft.Extensions.DependencyInjection; // Подключение пространства имен для внедрения зависимостей
using Microsoft.Extensions.Hosting; // Подключение пространства имен для управления жизненным циклом приложения

namespace WebApplication3
{
    public class Program
    {
        public static void Main(string[] args)
        {
            var builder = WebApplication.CreateBuilder(args); // Создание билдера для конфигурации приложения

            // Добавление сервисов в контейнер
            builder.Services.AddControllersWithViews(); // Добавление поддержки MVC контроллеров с представлениями

            // Настройка строки подключения и контекста данных
            string connection = builder.Configuration.GetConnectionString("DefaultConnection"); // Получение строки подключения из конфигурации
            builder.Services.AddDbContext<UserContext>(options => options.UseSqlServer(connection)); // Настройка контекста данных для использования SQL Server

            // Настройка аутентификации с использованием куки
            builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
                .AddCookie(options =>
                {
                    options.LoginPath = new Microsoft.AspNetCore.Http.PathString("/Account/Login"); // Указание пути для страницы входа
                    options.LogoutPath = new Microsoft.AspNetCore.Http.PathString("/Account/Logout"); // Указание пути для страницы выхода
                });

            var app = builder.Build(); 

            // Конфигурация конвейера HTTP-запросов
            if (app.Environment.IsDevelopment())
            {
                app.UseDeveloperExceptionPage(); 
            }
            else
            {
                app.UseExceptionHandler("/Home/Error"); 
                app.UseHsts(); 
            }

            app.UseHttpsRedirection(); 
            app.UseStaticFiles(); 

            app.UseRouting(); 

            app.UseAuthentication(); // Включение аутентификации
            app.UseAuthorization(); // Включение авторизации

            app.MapControllerRoute(
                name: "default", // Настройка маршрута по умолчанию для MVC
                pattern: "{controller=Home}/{action=Index}/{id?}");

            app.Run(); 
        }
    }
}
```
## Создание структуры авторизации и регистрации пользователя
Необходимо добавить папку ViewModels в сам проект, после этого в этой папке определить класс RegisterModel, который будет представлять модель для регистьрации пользователя. 
```csharp
using System.ComponentModel.DataAnnotations;

namespace WebApplication3.ViewModels
{
    public class RegisterModel
    {
        [Required(ErrorMessage = "Не указано Имя пользователя")]
        public string UserName { get; set; }



        [Required(ErrorMessage = "Не указан пароль")]
        [DataType(DataType.Password)]
        public string Password { get; set; }




        [DataType(DataType.Password)]
        [Compare("Password", ErrorMessage = "Пароль введен неверно")]
        public string ConfirmPassword { get; set; }
    }
}
```
В этом коде используется два похожих поля: Password и ConfirmPassword для дополнительного подтверждения пароля при регистрации , а также для избежания занесения неверного пароля в БД.
