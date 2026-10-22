# Методическое пособие: Модульное тестирование веб-приложений на чистом PHP (LAMP + MVC)

> **Назначение**: Практическое руководство по проведению модульного тестирования MVC-приложений на чистом PHP (стек LAMP) с использованием фреймворка PHPUnit, оценке покрытия кода (Code Coverage), адаптации тестов под любую пользовательскую структуру проекта и запуск без доступа к SSH/терминалу сервера.

---

## 📋 Содержание
1. [Введение и архитектурные принципы](#1-введение-и-архитектурные-принципы)
   - [1.1. Что такое модульный тест (Unit Test)?](#11-что-такое-модульный-тест-unit-test)
   - [1.2. Архитектура LAMP + MVC и слои приложения](#12-архитектура-lamp--mvc-и-слои-приложения)
   - [1.3. Карта соответствия: Слои MVC, типы данных и задачи тестирования](#13-карта-соответствия-слои-mvc-типы-данных-и-задачи-тестирования)
2. [Подготовка окружения и адаптация структуры проекта](#2-подготовка-окружения-и-адаптация-структуры-проекта)
   - [2.1. Пошаговый алгоритм адаптации тестов под вашу структуру каталогов](#21-пошаговый-алгоритм-адаптации-тестов-под-вашу-структуру-каталогов)
   - [2.2. Конфигурация `composer.json` для произвольных структур](#22-конфигурация-composerjson-для-произвольных-структур)
   - [2.3. Конфигурация `phpunit.xml`](#23-конфигурация-phpunitxml)
3. [Лабораторная работа №1: Тестирование сущностей и моделей (Model/Entity)](#3-лабораторная-работа-1-тестирование-сущностей-и-моделей-modelentity)
4. [Лабораторная работа №2: Тестирование ошибок и исключений (Unhappy Paths)](#4-лабораторная-работа-2-тестирование-ошибок-и-исключений-unhappy-paths)
5. [Лабораторная работа №3: Дата-провайдеры и параметризованные тесты](#5-лабораторная-работа-3-дата-провайдеры-и-параметризованные-тесты)
6. [Лабораторная работа №4: Изоляция базы данных и Мок-объекты (Mocking Repositories)](#6-лабораторная-работа-4-изоляция-базы-данных-и-мок-объекты-mocking-repositories)
7. [Лабораторная работа №5: Тестирование контроллеров и подмена суперглобалов](#7-лабораторная-работа-5-тестирование-контроллеров-и-подмена-суперглобалов)
8. [Лабораторная работа №6: Веб-раннер тестов на хостинге без доступа к SSH](#8-лабораторная-работа-6-веб-раннер-тестов-на-хостинге-без-доступа-к-ssh)
9. [Автоматический контроль покрытия кода (Code Coverage)](#9-автоматический-контроль-покрытия-кода-code-coverage)
10. [Инструкция: Выполнение composer dump-autoload без SSH (4 метода)](#10-инструкция-выполнение-composer-dump-autoload-без-доступа-к-ssh-4-метода)
11. [Шпаргалка: Подмена суперглобальных массивов ($_GET, $_POST, $_SESSION)](#11-шпаргалка-подмена-суперглобальных-массивов-_get-_post-_session)
12. [Чек-лист готовности и лучшие практики](#12-чек-лист-готовности-и-лучшие-практики)

---

## 1. Введение и архитектурные принципы

### 1.1. Что такое модульный тест (Unit Test)?
Модульный тест — это автоматизированная проверка **одного класса или метода в полной изоляции** от внешней среды (базы данных MySQL, сетевых вызовов, файловой системы и HTTP-сессий) [1, 2].

### 1.2. Архитектура LAMP + MVC и слои приложения
Для достижения высокой тестируемости приложения его код разделяется на четкие слои с применением принципа **внедрения зависимостей (Dependency Injection)** [4, 6]:

```text
       [ HTTP-запрос / Браузер ]
                  │
                  ▼
         ┌─────────────────┐
         │   Controller    │ ◄── Тестируется с помощью Моков
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │ Business/Service│ ◄── Содержит use-cases приложения
         └────────┬────────┘
                  │
        ┌─────────┴─────────┐
        ▼                   ▼
┌──────────────┐    ┌──────────────┐
│ Model/Entity │    │  Repository  │ ◄── Подменяется Mock-объектом
└──────────────┘    └──────────────┘     (без реальных SQL-запросов)
```

### 1.3. Карта соответствия: Слои MVC, типы данных и задачи тестирования

Поскольку у каждого студента веб-приложение может иметь уникальное именование каталогов (`app/`, `src/`, `core/`, `classes/`) и слоев, используйте эту матрицу для сопоставления вашего кода с типом теста и используемыми типами данных:

| Слой вашего MVC | Примеры ваших файлов | Входные / Выходные типы данных | Задача тестирования | Используемый инструмент PHPUnit |
| :--- | :--- | :--- | :--- | :--- |
| **Model / Entity / Value Object** | `User.php`, `Product.php`, `OrderEntity.php` | **Вход**: скаляры (`string`, `int`, `float`, `bool`).<br>**Выход**: состояние объекта, геттеры, исключения. | Проверка корректности инвариантов сущности, валидации данных в конструкторе или сеттерах. | `assertSame()`, `assertTrue()`, `expectException()` |
| **Validator / Helper / Checker** | `EmailValidator.php`, `FormChecker.php`, `StringHelper.php` | **Вход**: `string`, `array` (массив формы).<br>**Выход**: `bool`, массив ошибок. | Проверка множества наборов данных (граничные значения, корректные/некорректные строки). | `#[DataProvider]`, `assertFalse()` |
| **Repository / DAO / DB-Mapper** | `UserRepository.php`, `OrderDb.php`, `ProductMapper.php` | **Вход**: ID (`int`), массивы условий.<br>**Выход**: объекты моделей, массивы строк из БД. | В юнит-тестах этот слой **подменяется Моком (`Mock`)**, чтобы сервис тестировался без подключения к MySQL. | `$this->createMock()`, `willReturn()` |
| **Service / Use-Case / Domain** | `UserService.php`, `OrderProcessor.php`, `AuthManager.php` | **Вход**: DTO, сущности, ID.<br>**Выход**: результаты бизнес-операций, статусы. | Проверка бизнес-сценариев использования (use-cases) при взаимодействии с подмененным репозиторием. | `expects($this->once())`, `with()` |
| **Controller / Action / Handler** | `UserController.php`, `LoginAction.php`, `CartController.php` | **Вход**: `Request` (`$_GET`, `$_POST`, `$_SESSION`).<br>**Выход**: HTML-строка, JSON, редирект. | Проверка приема входных параметров HTTP, передача их в Сервис и вызов Представления (View). | Подмена `Request` и `Service` через Моки |

---

## 2. Подготовка окружения и адаптация структуры проекта

### 2.1. Пошаговый алгоритм адаптации тестов под вашу структуру каталогов

Если ваша файловая структура отличается от стандартного примера (`src/` + `tests/`), выполните следующие шаги для адаптации каждого теста:

1. **Идентификация класса**: Найдите целевой класс в вашем проекте и определите его полное пространство имен (`namespace`).
2. **Зеркальное отражение пути**: В каталоге `tests/` создайте подкаталоги, полностью повторяющие путь к вашему классу.
   * *Пример*: Если ваш класс лежит в `app/Core/Domain/User.php`, то файл теста должен лежать в `tests/Core/Domain/UserTest.php`.
3. **Определение входных данных**: Посмотрите на аргументы проверяемого метода. Подготовьте тестовые значения соответствующих типов (`int`, `string`, `array` или Моки для классов-зависимостей).
4. **Определение результата**: Выясните, что возвращает метод (скаляр, объект, `void`) или какое исключение выбрасывает (`Exception`). Выберите соответствующий метод утверждения (`assert*`).

### 2.2. Конфигурация `composer.json` для произвольных структур

Для того чтобы PHPUnit и ваш проект корректно находили файлы независимо от структуры папок, настройте секцию `autoload` в `composer.json`:

```json
{
    "autoload": {
        "psr-4": {
            "App\\": "src/",
            "Core\\": "app/Core/",
            "Classes\\": "classes/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "Tests\\": "tests/"
        }
    },
    "require-dev": {
        "phpunit/phpunit": "^10.5 || ^11.0"
    }
}
```
*Примечание*: Выполните команду `composer dump-autoload` после любого изменения структуры пространств имен.

### 2.3. Конфигурация `phpunit.xml`
Укажите PHPUnit, где искать ваши тесты и исходный код приложения:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit bootstrap="vendor/autoload.php"
         colors="true"
         cacheDirectory=".phpunit.cache">
    <testsuites>
        <testsuite name="Unit">
            <directory>tests</directory>
        </testsuite>
    </testsuites>
    <source>
        <include>
            <!-- Добавьте директории с вашим исходным кодом (src, app, classes) -->
            <directory suffix=".php">src</directory>
            <directory suffix=".php">app</directory>
        </include>
    </source>
</phpunit>
```

---

## 3. Лабораторная работа №1: Тестирование сущностей и моделей (Model/Entity)

**Цель**: Освоить структуру теста **AAA (Arrange, Act, Assert)** при проверке состояния сущностей [1, 5].

🔍 **Руководство по адаптации под вашу структуру проекта:**
* **Целевой слой MVC**: `Model`, `Entity`, `Value Object` (например, `User.php`, `Product.php`, `Order.php`).
* **Типы данных**: Скаляры (`string`, `int`, `float`, `bool`) — передаются в конструктор или сеттеры.
* **Задача теста**: Убедиться, что объект правильно сохраняет свое состояние и возвращает его через геттеры.
* **Зависимости**: Нет (чистый объект в памяти).

### Пример кода тестируемого класса (`src/Models/User.php` или аналогичного в вашем проекте)
```php
<?php
namespace App\Models; // 👈 Замените namespace на ваш (например, App\Entity или Core\Models)

class User
{
    private string $name;
    private string $email;

    public function __construct(string $name, string $email)
    {
        $this->name = $name;
        $this->email = $email;
    }

    public function getName(): string
    {
        return $this->name;
    }

    public function getEmail(): string
    {
        return $this->email;
    }
}
```

### Тестовый класс (`tests/Unit/Models/UserTest.php`)
```php
<?php
namespace Tests\Unit\Models; // 👈 Замените namespace согласно вашей папке в tests/

use App\Models\User; // 👈 Импортируйте ВАШ класс сущности
use PHPUnit\Framework\TestCase;

final class UserTest extends TestCase
{
    public function testUserCanBeCreatedWithValidData(): void
    {
        // 1. Arrange (Подготовка) — заводим исходные данные соответствующих типов
        $name = 'Алексей';
        $email = 'alex@example.com';

        // 2. Act (Действие) — создаем экземпляр вашей сущности
        $user = new User($name, $email);

        // 3. Assert (Проверка) — сверяем результаты работы геттеров
        $this->assertSame($name, $user->getName());
        $this->assertSame($email, $user->getEmail());
    }
}
```

---

## 4. Лабораторная работа №2: Тестирование ошибок и исключений (Unhappy Paths)

**Цель**: Научиться тестировать реакцию кода на невалидные входные данные с использованием структуры **Arrange, Expect, Act** [1].

🔍 **Руководство по адаптации под вашу структуру проекта:**
* **Целевой слой MVC**: `Model`, `Entity`, `Service`, `Domain Class`.
* **Типы данных**: Некорректные скаляры (невалидный email, отрицательная цена, пустая строка) -> Ожидаемый тип данных: исключение (`\InvalidArgumentException`, `\DomainException` или ваше пользовательское исключение).
* **Задача теста**: Проверить, что класс защищает свое состояние от некорректных данных и выбрасывает исключение.
* **Зависимости**: Нет.

### Модификация класса `User` (`src/Models/User.php`)
```php
public function __construct(string $name, string $email)
{
    if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
        throw new \InvalidArgumentException("Некорректный email адрес");
    }
    $this->name = $name;
    $this->email = $email;
}
```

### Тест проверки исключения (`tests/Unit/Models/UserTest.php`)
```php
public function testCannotBeCreatedFromInvalidEmail(): void
{
    // 1. Expect (Ожидание) — указываем класс исключения и (опционально) сообщение ДО вызова кода
    $this->expectException(\InvalidArgumentException::class);
    $this->expectExceptionMessage('Некорректный email адрес');

    // 2. Act (Действие) — провоцируем выброс исключения передачи невалидных данных
    new User('Алексей', 'invalid-email-string');
}
```

---

## 5. Лабораторная работа №3: Дата-провайдеры и параметризованные тесты

**Цель**: Избавиться от дублирования тестового кода за счет дата-провайдеров (`#[DataProvider]`) [1].

🔍 **Руководство по адаптации под вашу структуру проекта:**
* **Целевой слой MVC**: `Validator`, `Checker`, `FormRequest`, `Helper`.
* **Типы данных**:
  * Вход: `string`, `int`, `array` (наборы граничных данных).
  * Выход: `bool` (`true`/`false`) или массив ошибок.
* **Задача теста**: Проверить алгоритм на десятках комбинаций данных (пустые строки, спецсимволы, граничные значения) в одном тестовом методе.

### Код валидатора (`src/Checkers/UserChecker.php`)
```php
<?php
namespace App\Checkers; // 👈 Укажите namespace вашего валидатора

class UserChecker
{
    public function isValidEmail(string $email): bool
    {
        return filter_var($email, FILTER_VALIDATE_EMAIL) !== false;
    }
}
```

### Тест с провайдером данных (`tests/Unit/Checkers/UserCheckerTest.php`)
```php
<?php
namespace Tests\Unit\Checkers;

use App\Checkers\UserChecker; // 👈 Импортируйте ваш класс
use PHPUnit\Framework\Attributes\DataProvider;
use PHPUnit\Framework\TestCase;

final class UserCheckerTest extends TestCase
{
    // Статический метод-провайдер возвращает массив наборов: ['описание' => [аргумент1, ожидания]]
    public static function emailProvider(): array
    {
        return [
            'валидный email'       => ['user@example.com', true],
            'email без домена'     => ['user@', false],
            'пустая строка'        => ['', false],
            'домен с поддоменами' => ['dev@sub.domain.ru', true],
        ];
    }

    #[DataProvider('emailProvider')]
    public function testEmailValidation(string $email, bool $expected): void
    {
        $checker = new UserChecker();
        $this->assertSame($expected, $checker->isValidEmail($email));
    }
}
```

---

## 6. Лабораторная работа №4: Изоляция базы данных и Мок-объекты (Mocking Repositories)

**Цель**: Научиться тестировать бизнес-сервисы без подключения к реальной базе данных MySQL [6, 8].

🔍 **Руководство по адаптации под вашу структуру проекта:**
* **Целевой слой MVC**: `Service`, `Use-Case`, `Manager` (принимает `Repository` или `DAO` через конструктор).
* **Типы данных**:
  * Зависимость: Класс репозитория/базы данных (`UserRepository`, `ProductDb`, `PDOHandler`).
  * Возвращаемое значение репозитория: Объекты сущностей или массив строк из таблицы MySQL.
* **Задача теста**: Изолировать сервис от реального сервера MySQL с помощью Мок-объекта (`Mock`), задав заранее заготовленный ответ базы данных.

### Бизнес-сервис (`src/Services/UserService.php`)
```php
<?php
namespace App\Services;

use App\Models\User;
use App\Repositories\UserRepository; // 👈 Замените на класс вашего слоя работы с БД

class UserService
{
    private UserRepository $repository;

    public function __construct(UserRepository $repository)
    {
        $this->repository = $repository;
    }

    public function getUserNameById(int $id): string
    {
        $user = $this->repository->findById($id);
        if (!$user) {
            throw new \RuntimeException("Пользователь не найден");
        }
        return $user->getName();
    }
}
```

### Тест с использованием Мока (`tests/Unit/Services/UserServiceTest.php`)
```php
<?php
namespace Tests\Unit\Services;

use App\Models\User;
use App\Repositories\UserRepository; // 👈 Класс/интерфейс вашего репозитория для мокинга
use App\Services\UserService;
use PHPUnit\Framework\TestCase;

final class UserServiceTest extends TestCase
{
    public function testGetUserNameByIdReturnsCorrectName(): void
    {
        // 1. Создаем мок класса репозитория (вместо реального вызова MySQL)
        $mockRepo = $this->createMock(UserRepository::class);

        // 2. Настраиваем ожидание: метод 'findById' вызовется 1 раз с ID=10 и вернет готовую модель
        $mockRepo->expects($this->once())
            ->method('findById')
            ->with(10)
            ->willReturn(new User('Мария', 'maria@test.ru'));

        // 3. Внедряем мок в ваш сервис через конструктор
        $service = new UserService($mockRepo);

        // 4. Вызываем тестируемый метод сервиса
        $name = $service->getUserNameById(10);
        $this->assertSame('Мария', $name);
    }
}
```

---

## 7. Лабораторная работа №5: Тестирование контроллеров и подмена суперглобалов

**Цель**: Научиться тестировать методы контроллеров, зависящих от `$_GET`, `$_POST`, `$_SESSION` и `$_SERVER`.

🔍 **Руководство по адаптации под вашу структуру проекта:**
* **Целевой слой MVC**: `Controller`, `Action`, `Handler`.
* **Типы данных**:
  * Вход: HTTP-параметры (`Request`, `$_POST`, `$_GET`).
  * Выход: Сформированная строка (HTML/JSON) или ответ сервиса.
* **Задача теста**: Проверить цепочку: получение HTTP-параметров -> вызов бизнес-сервиса -> возврат результата контроллером.

### Подход с объектом-оберткой `Request` (`src/Http/Request.php`)
```php
<?php
namespace App\Http;

class Request
{
    public function getPost(string $key, mixed $default = null): mixed
    {
        return $_POST[$key] ?? $default;
    }
}
```

### Контроллер (`src/Controllers/UserController.php`)
```php
<?php
namespace App\Controllers;

use App\Http\Request;
use App\Services\UserService;

class UserController
{
    private UserService $service;

    public function __construct(UserService $service)
    {
        $this->service = $service;
    }

    public function register(Request $request): string
    {
        $name = $request->getPost('name');
        if (empty($name)) {
            return "Ошибка: имя обязательно";
        }
        return "Успешно: " . htmlspecialchars($name);
    }
}
```

### Тест контроллера (`tests/Unit/Controllers/UserControllerTest.php`)
```php
<?php
namespace Tests\Unit\Controllers;

use App\Controllers\UserController;
use App\Http\Request;
use App\Services\UserService;
use PHPUnit\Framework\TestCase;

final class UserControllerTest extends TestCase
{
    public function testRegisterActionSuccess(): void
    {
        // Подменяем и сервис, и HTTP-запрос моками
        $mockService = $this->createMock(UserService::class);
        $mockRequest = $this->createMock(Request::class);

        $mockRequest->method('getPost')
            ->with('name')
            ->willReturn('Дмитрий');

        $controller = new UserController($mockService);
        $response = $controller->register($mockRequest);

        $this->assertStringContainsString('Успешно: Дмитрий', $response);
    }
}
```

---

## 8. Лабораторная работа №6: Веб-раннер тестов на хостинге без доступа к SSH

**Цель**: Настроить безопасный запуск тестов PHPUnit через веб-интерфейс [5].

🔍 **Руководство по адаптации под вашу структуру проекта:**
* Укажите путь к вашей директории с тестами в методе `$runner->run(['phpunit', '--colors=never', 'путь/к/вашим/тестам'])`.

```php
<?php
// Веб-скрипт запуска тестов
require_once __DIR__ . '/vendor/autoload.php';

if (isset($_GET['run'])) {
    $runner = new PHPUnit\TextUI\Application();
    $runner->run(['phpunit', '--colors=never', 'tests']);
}
```

---

## 9. Автоматический контроль покрытия кода (Code Coverage)

### 9.1. Что такое покрытие кода и зачем оно нужно?
**Покрытие кода (Code Coverage)** — это метрика, показывающая, какой процент строк, условий и ветвей исходного кода программы выполняется при прохождении всех автотестов [1, 2].

* **Line Coverage (Покрытие строк)**: Процент исполняемых строк кода, затронутых тестами.
* **Branch Coverage (Покрытие ветвлений)**: Проверка всех возможных развилок `if/else`, `switch` и `try/catch`.
* **Path Coverage (Покрытие путей)**: Проверка всех комбинаций путей выполнения программы.

> 💡 **Правило золотой середины**: Рекомендуемый целевой уровень покрытия для коммерческих веб-приложений составляет **80%–85%**. Попытки добиться 100% покрытия часто приводят к написанию бесполезных «формальных» тестов.

### 9.2. Драйверы покрытия в экосистеме PHP
Для сбора метрик покрытия требуется специальный драйвер в среде выполнения PHP [2, 5]:

1. **Xdebug** — классический и наиболее распространенный отладчик.
   * *Плюсы*: Генерирует детальные отчеты, встроен во многие сборки локальных серверов (XAMPP, Open Server, Docker).
   * *Минусы*: Достаточно медленный.
   * *Включение*: В `php.ini` укажите `xdebug.mode=coverage`.
2. **PCOV** — легкий и сверхбыстрый драйвер, разработанный специально для сбора Code Coverage.
   * *Плюсы*: В 5–10 раз быстрее Xdebug, минимальное потребление памяти.
   * *Рекомендация*: Идеальный выбор для CI/CD и постоянной разработки.
3. **PHPDBG** — встроенный модуль отладки PHP (запускается через CLI: `phpdbg -qrr vendor/bin/phpunit`).

### 9.3. Настройка `phpunit.xml` для учета покрытия
Укажите каталоги ваших исходников (`src`, `app`, `classes`), исключив сторонние папки и представления [1]:

```xml
<source>
    <include>
        <!-- Добавьте пути к исходному коду вашей структуры -->
        <directory suffix=".php">src</directory>
        <directory suffix=".php">app</directory>
    </include>
    <exclude>
        <directory>src/Views</directory>
        <directory>app/Views</directory>
    </exclude>
</source>
```

### 9.4. Генерация и анализ HTML-отчета
Для генерации визуального интерактивного HTML-отчета выполните команду [1]:

```bash
XDEBUG_MODE=coverage vendor/bin/phpunit --coverage-html build/coverage-html
```

В директории `build/coverage-html/` появится веб-сайт отчета (`index.html`):
* 🟢 **Зеленый цвет**: Строки кода полностью выполнены тестами.
* 🔴 **Красный цвет**: Непокрытые строки (код ни разу не вызывался в тестах).
* 🟡 **Желтый цвет**: Частично покрытые условия (например, в `if ($a && $b)` была проверена только первая часть).

### 9.5. Установка порога контроля покрытия в CI/CD
Для автоматической проверки порога покрытия в терминале или CI/CD можно использовать флаг `--fail-on-low-coverage` вместе со встроенной опцией:

```bash
# Тест завершится ошибкой (exit code > 0), если покрытие кода ниже 80%
vendor/bin/phpunit --coverage-text --fail-on-low-coverage
```

---

## 10. Инструкция по обновлению карты автозагрузки Composer без SSH (4 метода)

При добавлении новых классов или изменении пространств имен (namespace) необходимо обновлять карту автозагрузки Composer.

---

### 10.1. Способ 1: Локальная генерация и выгрузка по FTP/SFTP (Рекомендуемый)

1. Выполните команду генерации локально: `composer dump-autoload -o`
2. Загрузите обновленную папку `vendor/composer/` на хостинг.

---

### 10.2. Способ 2: Запуск через PHP-скрипт

1. Используйте PHP-скрипт для вызова выполнения команды генерации.
2. После работы обязательно удалите временные скрипты с хостинга.

---

### 10.3. Способ 3: Использование Web-терминала панели хостинга

1. Откройте Web-Shell или Терминал в панели управления хостингом.
2. Выполните команду `composer dump-autoload -o`.

---

### 10.4. Способ 4: Кастомный автозагрузчик

1. Подключите резервную функцию автозагрузки при отсутствии Composer CLI.

---

### 10.5. Сравнительная таблица методов

| Метод | Уровень безопасности | Рекомендация |
| :--- | :--- | :--- |
| **Локально + FTP** | Высокая | Основной рекомендуемый метод |
| **PHP-скрипт** | Средняя | При частых правках |
| **Web-Shell панели** | Высокая | При наличии терминала в панели |
| **Кастомный автозагрузчик** | Высокая | Крайний случай |

---



## 11. Шпаргалка: Подмена суперглобальных массивов ($_GET, $_POST, $_SESSION)

| Метод подмены | Пример реализации | Плюсы | Минусы |
| :--- | :--- | :--- | :--- |
| **Прямая перезапись и `tearDown()`** | `$_POST = ['key' => 'val'];` | Работает без модификации кода приложения | Требует ручного отката состояния в каждом тесте |
| **Атрибут `#[BackupGlobals(true)]`** | `#[BackupGlobals(true)] class Test extends TestCase` | Автоматически изолирует изменения массивов | Небольшое снижение скорости выполнения тестов |
| **Обертка `Request` + DI** | `function index(Request $req)` | Безупречная архитектура, чистые моки | Требует рефакторинга вызовов суперглобалов в коде |

---

## 12. Чек-лист готовности и лучшие практики

- [x] Вы сопоставили структуру своего проекта с матрицей слоев MVC в разделе 1.3.
- [x] Каждый тест независим от других и может быть запущен изолированно [2, 4].
- [x] Все внешние зависимости (PDO, внешние API, файлы) подменяются Mock-объектами или Stubs [6, 8].
- [x] Тесты не делают реальных запросов к базе данных MySQL [5, 6].
- [x] Названия тестовых методов четко описывают проверяемый сценарий (`testUserCannotBeCreatedWithInvalidEmail`) [1].
- [x] Включен автоматический контроль покрытия кода (покрытие ключевой бизнес-логики выше 80%) [1, 5].
- [x] Веб-раннер `run-tests.php` защищен паролем и удаляется с боевого хостинга сразу после проверки.
