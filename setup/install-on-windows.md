# Установка на Windows

Как мы уже знаем, для разработки на Rust под Windows, кроме Rust тулчейна нам потребуется стандартная библиотека C и линкер. На данный момент, на выбор есть два варианта их установки:

* Установить Microsoft Visual C++, которая помимо всего прочего включает в себя и стандартную библиотеку C, и линкер.\
  Этот способ быстрее и удобнее.
* Установить MinGW, которая включает в себя порт GCC под Windows, который в свою очередь содержит стандартную библиотеку C и линкер.\
  Этот способ неменого менее дружественный для тех, кто не знаком с экосистемой C++ под Windows.

По умолчанию рекомендуется использовать вариант с Visual C++.

## Rustup

Для всех вариантов установки нам потребуется утилита rustup. Для Window установщик rustup поставляется в виде исполняемого файла `rustup-init.exe`, который можно скачать с официально сайта [https://rust-lang.org/tools/install/](https://rust-lang.org/tools/install/)

<figure><img src="../.gitbook/assets/install_sdk-0.png" alt=""><figcaption></figcaption></figure>

Или по прямой ссылке: [https://static.rust-lang.org/rustup/dist/x86\_64-pc-windows-gnu/rustup-init.exe](https://static.rust-lang.org/rustup/dist/x86_64-pc-windows-gnu/rustup-init.exe)

Пока что просто скачаем `rustup-init.exe`, но запускать его будет после установки C++ тулчейна.

## Установка с Microsoft Visual C++

Мы предполагаем, что у вас еще не установлена Visual Studio.

1\) Скачайте инсталятор Visual Studio с [https://visualstudio.microsoft.com/](https://visualstudio.microsoft.com/)\
Нам потребуется бесплатная версия Visual Studio Community.

2\) Запустите инсталятор, нажмите "Continue", после чего вы должны увидеть окно выбора компонентов для установки.

Выберете категорию "Desktop development with C++", и отметте только компоненты

* MSVC Build Tools for x64/x86
* Windows 11 SDK

<figure><img src="../.gitbook/assets/install_visual_studio_min_components.png" alt=""><figcaption></figcaption></figure>

После нажмите "Install", и дождитесь завершения установки компонентов. После этого можно закрывать инсталятор.

{% hint style="info" %}
Если вы не будете использовать библиотеки написанные на C/C++ (а в рамках этой книги мы не будем их использовать), то вам хватит только этих двух компонентов. Но в будущем вам могут понадобиться еще:

* C++ CMake tools for Windows — CMake утилита при помощи, которой собирается множество библиотек написанных на C++
* vcpkg package manager — утилита для установки библиотек на C/C++
{% endhint %}

3\) Теперь пришла очередь ранее скаченного `rustup-init.exe`. Запускаем инсталятор. Установщик по умолчанию предложит установить Rust тулчейн под target (целевую плтформу) x86\_64-pc-windows-msvc. Это как раз и есть сборка под Windows с использованием линкера и стандартной бибилотеки от Visual C++. В консоли это должено выглядеть так:

<figure><img src="../.gitbook/assets/install_sdk-4.png" alt=""><figcaption></figcaption></figure>

Выбираем 1-й вариант (просто установить тулчейн с настроками по-умолчанию).&#x20;

<figure><img src="../.gitbook/assets/install_sdk-5.png" alt=""><figcaption></figcaption></figure>

После установки компонентов Rust тулчейна, это окно консоли можно закрывать. Всё что нужно — установлено.

4\) Теперь надо проверить, что всё работает корректно.

Откройте консоль (PowerShell или cmd), и выполните следующие команды:

* `cargo new test_rust` — создать новый Rust проект. С утилитой Cargo мы познакомимся потом, а пока что нам достаточно знать, что этой командой мы создадим "болванку" Rust программы, которая просто печатает на консоль строку "Hello, world!"
* `cd test_rust` — перейти в свежесозданный каталог test\_rust
* `cargo run` — скомпилировать и запустить программу

В консоли это должно выглядеть так:

<figure><img src="../.gitbook/assets/verify_installation.png" alt=""><figcaption></figcaption></figure>

Всё готово.

***

## Установка с MinGW

[MinGW](https://www.mingw-w64.org/) — порт на Windows компилятора GCC и различных утилит. Он так же содержит свой линкер и стандартную библотеку C, поэтому может быть использован для сборки програм на Rust.

Существует [несколько способов](https://github.com/niXman/mingw-builds-binaries/releases) установки MinGW. Мы рассмотрим два из них:

* [MinGW-W64-builds](https://www.mingw-w64.org/downloads/#mingw-w64-builds) — просто архив с утилитами и библиотеками
* [MSYS2](https://www.msys2.org/) — среда для создания Linux-подобного окружения для разработки под Windows. Предлагает пакетный менеджер для простой установки утилит и библиотек, через который можно установить MinGW.

***

### MinGW w64

Оригинальная сборка MinGW-w64 поставляется просто как архив, который можно скачать с официальной GitHub страницы: [https://github.com/niXman/mingw-builds-binaries/releases](https://github.com/niXman/mingw-builds-binaries/releases).

На странице скачивания, вы можетей найти несколько вариантов сборки, чьи имена составлены по схеме:

`архитектура-версия-relese-АПИ_многопоточности-эксепшены-Си_рантайм-ревизия.7z`

Например:

* x86\_64-15.2.0-release-mcf-seh-ucrt-rt\_v13-rev0.7z
* x86\_64-15.2.0-release-posix-seh-msvcrt-rt\_v13-rev0.7z
* x86\_64-15.2.0-release-posix-seh-ucrt-rt\_v13-rev0.7z
* x86\_64-15.2.0-release-win32-seh-msvcrt-rt\_v13-rev0.7z
* x86\_64-15.2.0-release-win32-seh-ucrt-rt\_v13-rev0.7z&#x20;

Вам подойдёт любой вариант, кроме `mcf`, который не работал с Rust на момент написания этого текста (Rust 1.92). Лично автор предпочитает `win32-seh-ucrt-rt` вариант.

1\) Скачайте 7z архив MinGW, и распакуйте в какую-то папку, например `C:\dev\mingw64`.

<figure><img src="../.gitbook/assets/mingw_folder.png" alt="" width="563"><figcaption></figcaption></figure>

2\) Добавте каталог `mingw64/bin` в системную переменную `Path`.

<figure><img src="../.gitbook/assets/mingw_path.png" alt=""><figcaption></figcaption></figure>

3\) Теперь запустите `rustup-init.exe`.

На первый вопрос, который предлагает автоматическую установку компонентов, выберите пункт (3) — ничего не устанавливать.

<figure><img src="../.gitbook/assets/mingw_rustup_1.png" alt=""><figcaption></figcaption></figure>

Далее Rustup покажет конфигурацию, которую он предложит установить. По умолчанию это будет `x86_64-pc-windows-msvc` (тулчей и целевая платформа для Visual C++).

Выберети вариант (2) — кастомизировать конфигурацию.

<figure><img src="../.gitbook/assets/mingw_rustup_2.png" alt=""><figcaption></figcaption></figure>

Rustup попросит указать имя Rust тулчейна, который вы хотите установить. Вместо предлагаемого по умолчанию `x86_64-pc-windows-msvc`, укажите `x86_64-pc-windows-gnu`.

<figure><img src="../.gitbook/assets/mingw_rustup_3.png" alt=""><figcaption></figcaption></figure>

Для остальных параметров подходят значения по умолчанию.

<figure><img src="../.gitbook/assets/mingw_rustup_4.png" alt=""><figcaption></figcaption></figure>

Установка завершена. Теперь протестируем, что мы можем собрать программу Rust.

Откройте новую консоль, и создайте и запустите Hello World программу, так же, как и в опасании установки вместе с Visual C++.

* `cargo new test_rust`
* `cd test_rust`
* `cargo run`

Результат должен выглядеть так:

<figure><img src="../.gitbook/assets/mingw_rustup_5.png" alt=""><figcaption></figcaption></figure>

***

### MSYS2

Если в предыдщем сценарии с MinGW w64, мы вручную качали дистрибутив MinGW и самостоятельно добавляли его в системные пути, то при использовании MSYS2 всё будет куда более автоматизировано: [MSYS2](https://www.msys2.org/) предоставляет пакетный менеджер `pacman`, который умеет скачивать и устанавливать программы и библиотеки из удалённого репозитория, и с его помощью мы установим MinGW.

1\) Сначала скачайте инсталятор MSYS2: [https://www.msys2.org/#installation](https://www.msys2.org/#installation)

<figure><img src="../.gitbook/assets/msys2_install_1.png" alt="" width="563"><figcaption></figcaption></figure>

Запустите инсталятор, и выберите путь по которому будет хранится как сам MSYS2, так и устанавливаемые им программы и бибилотеки:

<figure><img src="../.gitbook/assets/msys2_install_2.png" alt="" width="480"><figcaption></figcaption></figure>

2\) После завршения установки, откроем msys2 консоль (должна появится в пеню "Пуск") и установим пакеты `mingw` и `base-develop` при помощи команд:

`pacman -S mingw-w64-x86_64-toolchain`

<figure><img src="../.gitbook/assets/msys2_install_3.png" alt=""><figcaption></figcaption></figure>

`pacman -S base-devel`

<figure><img src="../.gitbook/assets/msys2_install_4.png" alt=""><figcaption></figcaption></figure>

Вот и всё: MinGW установлен и добавлен в системные пути.

Теперь осталось установить Rust тулчейн. Делается это точно таким же образом, как и для ручной установки MinGW w64.

4\) Запустите `rustup-init.exe`.

Далее вместо `x86_64-pc-windows-msvc` выберите `x86_64-pc-windows-gnu` и завершить установку.

<figure><img src="../.gitbook/assets/msys2_install_5.png" alt=""><figcaption></figcaption></figure>

***

Установка Rust завершена. Теперь можно приступать к изучению самого языка.
