# WebAuthorization and Registration with Asp.Net Core 
## Начало работы
Необходимо создать проект ASP.NET Core Web App (Model-View-Controller).
<br> Подробней о создании проекта здесь: [METANIT.COM](https://metanit.com/sharp/aspnet5/3.1.php)

<br>Также нужно установить нужные для работы пакеты Nuget: 
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
Для того чтобы можно было взимодействовать с БД в папке Models необходимо создать контекст данных UsersContext  

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
В этом коде используется два похожих свойства: Password и ConfirmPassword для дополнительного подтверждения пароля при регистрации , а также для избежания занесения неверного пароля в БД.

<br> После в этой же папке добавим класс  LoginModel, который опишет модель для авторизации пользователя. 

```csharp
using System.ComponentModel.DataAnnotations;

namespace WebApplication3.ViewModels
{
    public class LoginModel
    {
        [Required(ErrorMessage = "Не указано Имя пользователя")]
        public string UserName { get; set; }

        [Required(ErrorMessage = "Не указан пароль")]
        [DataType(DataType.Password)]
        public string Password { get; set; }
    }
}
```
## Добавление главного контроллера 
<br> В папку Conrollers добавим AccountController и реализуем в нем следующий код: 
```csharp
using System.Collections.Generic;
using System.Threading.Tasks;
using Microsoft.EntityFrameworkCore;
using Microsoft.AspNetCore.Mvc;
using System.Security.Claims;
using WebApplication3.ViewModels; // пространство имен моделей RegisterModel и LoginModel
using WebApplication3.Models; // пространство имен UserContext и класса User
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Authentication.Cookies;
using System.Text;
using System.Security.Cryptography;// хеширование пароля

namespace WebApplication3.Controllers
{
    public class HashHelper //класс для хеширования пароля
    {
        public static string HashPasswordSHA1(string password) // метод  хешеирования пароля 
        {
            using (SHA1 sha1 = SHA1.Create())
            {
                byte[] bytes = Encoding.UTF8.GetBytes(password);
                byte[] hashBytes = sha1.ComputeHash(bytes);
                return Convert.ToBase64String(hashBytes);
            }
        }
    }

    public class AccountController : Controller
    {
        private UserContext db;
        public AccountController(UserContext context)
        {
            db = context;
        }

        private const string _allowedIPAddress = "127.0.0.1"; // замените на нужный IP-адрес

        [HttpGet]
        public IActionResult Login()
        {
            return View();
        }

        [HttpPost]
        [ValidateAntiForgeryToken]
        public async Task<IActionResult> Login(LoginModel model)
        {
            if (ModelState.IsValid)
            {// хеширование введеного пароля 
                var hashedPassword = HashHelper.HashPasswordSHA1(model.Password);

                User user = await db.Users.FirstOrDefaultAsync(u => u.UserName == model.UserName && u.Password == hashedPassword);

                if (user != null)
                {
                    await Authenticate(model.UserName); // аутентификация

                    return RedirectToAction("Index", "Home");
                }
                ModelState.AddModelError("", "Некорректные логин и(или) пароль");
            }
            return View(model);
        }
        [HttpGet]
        public IActionResult Register()
        {
            return View();
        }
        [HttpPost]
        [ValidateAntiForgeryToken]
        public async Task<IActionResult> Register(RegisterModel model)
        {

            //  var userIpAddress = HttpContext.Connection.RemoteIpAddress?.ToString();
            //костыль
            var userIpAddress = "127.0.0.1";
            if (userIpAddress == _allowedIPAddress)
            {

                if (ModelState.IsValid)
                {

                    User user = await db.Users.FirstOrDefaultAsync(u => u.UserName == model.UserName);

                    if (user == null)
                    {
                        // хешируем пароль
                        var hashedPassword = HashHelper.HashPasswordSHA1(model.Password);
                        // добавляем пользователя в бд
                        db.Users.Add(new User { UserName = model.UserName, Password = hashedPassword, Role = UserRole.User }); 
                        await db.SaveChangesAsync();

                        await Authenticate(model.UserName); // аутентификация

                        return RedirectToAction("Index", "Home");
                    }
                    else
                    {
                        ModelState.AddModelError("", "Некорректные логин и(или) пароль");
                    }
                       
                }
                return View(model);

            }
            ModelState.AddModelError("", "Registration is only allowed from a specific network");
            return View();
        }
        private async Task Authenticate(string userName)
        {
            // создаем один claim
            var claims = new List<Claim>
            {
                new Claim(ClaimsIdentity.DefaultNameClaimType, userName)
            };
            // создаем объект ClaimsIdentity
            ClaimsIdentity id = new ClaimsIdentity(claims, "ApplicationCookie", ClaimsIdentity.DefaultNameClaimType, 
                ClaimsIdentity.DefaultRoleClaimType);

            var authProperties = new AuthenticationProperties
            {
                // указываем время жизни куки
                ExpiresUtc = DateTimeOffset.UtcNow.AddMinutes(30), // куки будет действовать 30 минут
                IsPersistent = true // куки будут сохранены между сессиями браузера
            };

            // установка аутентификационных куки
            // await HttpContext.SignInAsync(CookieAuthenticationDefaults.AuthenticationScheme, new ClaimsPrincipal(id));
            await HttpContext.SignInAsync(CookieAuthenticationDefaults.AuthenticationScheme, new ClaimsPrincipal(id), authProperties);
        }

        [HttpPost]
        [ValidateAntiForgeryToken]
        public async Task<IActionResult> Logout()
        {
            await HttpContext.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme);
            return RedirectToAction("Login", "Account");
        }
    }
}
```
***Login:***
```csharp
HttpGet Login(): Отображает страницу входа.
HttpPost Login(LoginModel model)
```
 При получении данных формы входа, хеширует введенный пароль и проверяет его с базой данных. Если пользователь найден, выполняет аутентификацию и перенаправляет на главную страницу.

<br>***Register:***
```csharp
HttpGet Register(): Отображает страницу регистрации.
HttpPost Register(RegisterModel model)
```
При получении данных формы регистрации проверяет доступ с указанного IP-адреса. Если пользователь не существует, хеширует пароль и добавляет нового пользователя в базу данных, затем выполняет аутентификацию и перенаправляет на главную страницу.

***Authenticate:***
Создает и устанавливает аутентификационные куки с идентификатором пользователя.
<br>
***Logout:***
Выполняет выход пользователя путем удаления аутентификационных куки и перенаправления на страницу входа.

## Создание представлений
В папке Views создаем папку Account, в которой будет определено 3 файла: _Layout.cshtml, Login.cshtml, Register.cshtml. 
В этих файлах мы описываем интерфейс который видит пользователь при входе на сайт. 
<br> Пример файла Login.cshtml: 
```html
@model WebApplication3.ViewModels.LoginModel

<h2>Вход на сайт</h2>

<a asp-action="Register" asp-controller="Account">Регистрация</a>

<form asp-action="Login" asp-controller="Account" asp-anti-forgery="true">
    <div class="validation" asp-validation-summary="ModelOnly"></div>
    <div>
        <div class="form-group">
            <label asp-for="UserName">Введите Логин</label>
            <input type="text" asp-for="UserName" />
            <span asp-validation-for="UserName" />
        </div>
        <div class="form-group">
            <label asp-for="Password">Введите пароль</label>
            <input asp-for="Password" />
            <span asp-validation-for="Password" />
        </div>
        <div class="form-group">
            <input type="submit" value="Войти" class="btn btn-outline-dark" />
        </div>
    </div>
</form>
```
## Результат
После запуска проекта мы будем отправлены на  данную страничку:
<br> ![image](https://github.com/user-attachments/assets/6d469501-6c34-4fc6-815e-850d4ae8e4da)
<br> Так как мы не зарегистрированы, перейдем на страницу регистрации и заполним поля нажмем на кнопку регистрация:
<br> ![image](https://github.com/user-attachments/assets/15db734b-7188-4006-8001-66ed0e9fe60a)
<br> При успешной регистрации мы попадем на данную страницу: 
<br> ![image](https://github.com/user-attachments/assets/79aedf93-e192-4955-987f-020fc56753a3)
<br> А наши данные будут занесены в Бд, также мы получим куки с сроком действия: 
<br> ![image](https://github.com/user-attachments/assets/771da9cf-3431-4085-9027-859357b33ed1)
<br> ![image](https://github.com/user-attachments/assets/b41586f4-7dd5-4346-831e-88e56155a939)

## Конец 
После выполенения всех действий описанных выше, вы сможете создать и настроить свой сервис для регистрации и авторизации. 

