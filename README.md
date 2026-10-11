# Control Registry Viewer

Компактная утилита для Windows, которая читает **CmControlVector** из ядра
(`ntoskrnl.exe`) и позволяет просматривать область реестра **Control**. Написана
на WinUI 3 (Windows App SDK) и устроена как WinObjEx64: слева список разделов,
дерево ключей и таблица значений.

Работает **без драйвера ядра** — всё читается из пользовательского режима, ничего
устанавливать и включать (debug mode) не нужно.

## Возможности

- **CmControlVector** — разбирает массив `CmControlVector` прямо из образа
  `ntoskrnl.exe` и показывает все записи (`KeyPath`, `ValueName`), сгруппированные
  по ключу. Каждая запись сверяется с реальным реестром и помечается:
  **value** (существует значение), **key** (ключ есть, значения нет) или
  **absent** (нет вовсе).
- **Создание отсутствующих записей** — у записей, которых нет в реестре,
  появляется кнопка **+ Add**. Открывается диалог с выбором типа
  (REG_SZ / EXPAND_SZ / DWORD / QWORD / BINARY / MULTI_SZ) и полем данных. Данные
  обязательны — без них значение не создаётся.
- **Registry browser** — просмотр любой ветки реестра. Есть закладки (Control,
  GraphicsDrivers, Session Manager, Memory Management, Lsa, Power, Services,
  PriorityControl, Multimedia\SystemProfile, GameConfigStore и др.) и поле ввода:
  введите путь `HKLM\…` или `HKCU\…` и нажмите Enter.
- **Open в regedit** — выделите ключ и нажмите **Open**, чтобы открыть системный
  редактор реестра ровно на этом ключе (как «Jump to» в WinObjEx64).
- **Loaded drivers** — список загруженных модулей ядра через
  `NtQuerySystemInformation`.
- **Installed software** — установленные программы из ветки Uninstall
  (HKLM 64 + 32 + HKCU).
- **User Shared Data** — отдельные поля `KUSER_SHARED_DATA`.
- **Boot / CI info** — сборка Windows, Secure Boot, Code Integrity / VBS,
  параметры дампа.

## Как читается CmControlVector

`CmControlVector` — это массив внутри `ntoskrnl.exe`, записи которого содержат
только статические указатели в образ, поэтому имена ключей и значений читаются
без доступа к памяти ядра:

1. `LoadLibraryEx(ntoskrnl.exe, DONT_RESOLVE_DLL_REFERENCES)` отображает образ в
   процесс.
2. В образе находится строка `L"ProtectionMode"` (та же опорная точка, что и у
   WinObjEx64), затем указатель на неё — одна из записей массива.
3. Массив проходится до начала и затем вперёд **до `KeyPath == NULL`**, как и в
   WinObjEx64. Размер записи определяется автоматически (он отличается в разных
   сборках Windows).
4. Каждая запись сверяется с реестром через `Microsoft.Win32.Registry`.

Живые значения из памяти ядра (колонка «Value (Memory)» и дамп в WinObjEx64)
здесь не читаются — для этого нужен драйвер и debug mode.

## Что эта программа НЕ делает

Часть функций WinObjEx64 существует только в памяти ядра и требует подписанного
драйвера (а для WinDbg-драйвера — ещё и debug mode): SSDT (`KiServiceTable`),
`W32pServiceTable`, системные callbacks, живые значения CmControlVector и дампы
структур объектов. Для них используйте WinObjEx64.

## Сборка

Нужны .NET 8 SDK и workload Windows App SDK
(`dotnet workload install windowsappsdk`).

Проект настроен как **self-contained** и **single file**: рантайм вшивается, на
выходе — один самораспаковывающийся exe, ставить отдельный Windows App Runtime не
нужно.

```
dotnet publish -c Release -r win-x64
```

Работает на x64 Windows 10/11. Запускается с правами администратора — запись в
`Control` и чтение части защищённых подключей (например `Lsa`) требуют прав
админа.

## Примечания

- Весь доступ к реестру идёт через 64-битное представление
  (`RegistryView.Registry64`); представление по умолчанию под 64-битным процессом
  даёт 32-битную проекцию (WOW6432), которая скрывает часть подключей.
- Снимите галку **Only existing** в разделе CmControlVector, чтобы видеть все
  записи массива, включая отсутствующие в реестре — так же, как показывает
  WinObjEx64.


