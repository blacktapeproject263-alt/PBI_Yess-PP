# Инструкция по глобальной авторизации GitHub для всех проектов

Данная инструкция описывает, как на вашей рабочей станции настроена сквозная авторизация GitHub. Благодаря этой конфигурации **все текущие и будущие проекты** (Power BI, TMDL, скрипты, AI-ассистенты) автоматически проходят аутентификацию под вашей учетной записью **`blacktapeproject263-alt`** без повторного ввода паролей или токенов.

---

## 🛠️ Архитектура единой авторизации

Авторизация настроена на 3 независимых уровнях операционной системы:

```text
┌─────────────────────────────────────────────────────────────┐
│                 GitHub Personal Access Token                │
│            ghp_************************************         │
└──────────────┬───────────────────────────────┬──────────────┘
               │                               │
               ▼                               ▼
┌─────────────────────────────┐ ┌─────────────────────────────┐
│  Диспетчер учетных данных   │ │  Переменные окружения ОС    │
│  Windows (cmdkey / GCM)     │ │  (GH_TOKEN / GITHUB_TOKEN)  │
│  Авторизует Git CLI:        │ │  Авторизует:                │
│  - git push                 │ │  - GitHub CLI (gh)          │
│  - git pull                 │ │  - PowerShell скрипты       │
│  - git clone                │ │  - REST API запросы         │
└─────────────────────────────┘ └─────────────────────────────┘
               │                               │
               └──────────────┬────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│             Antigravity MCP (github-mcp-server)             │
│   Файл: C:\Users\a.syzdykov\.gemini\antigravity\mcp_config  │
│   Авторизует AI-ассистента на создание репозиториев,        │
│   управление файлами, ветками и PR через MCP-инструменты    │
└─────────────────────────────────────────────────────────────┘
```

---

## 1. Как это работает для Git (Любой проект на компьютере)

В диспетчере учетных данных Windows зарегистрирована постоянная запись:
* **Цель:** `git:https://github.com`
* **Пользователь:** `blacktapeproject263-alt`
* **Пароль (токен):** ваш персональный токен доступа.

### Подключение любого нового проекта за 3 шага:
В папке любого нового проекта (например, другого отчета Power BI) достаточно выполнить:

```powershell
# 1. Инициализация локального репозитория
git init -b main

# 2. Привязка к GitHub (замените New_Project_Name на имя вашего репо)
git remote add origin https://github.com/blacktapeproject263-alt/New_Project_Name.git

# 3. Фиксация и отправка
git add .
git commit -m "feat: initial commit"
git push -u origin main
```
> [!NOTE]
> Авторизация пройдет **автоматически в фоновом режиме**. Никаких окон с логином или запросов в терминале появляться не будет.

---

## 2. Как это работает для AI-ассистента и MCP

В файле конфигурации MCP (`~/.gemini/antigravity/mcp_config.json` и `~/.gemini/config/mcp_config.json`):

```json
"github-mcp-server": {
  "command": "C:\\Program Files\\nodejs\\npx.cmd",
  "args": [
    "-y",
    "@modelcontextprotocol/server-github"
  ],
  "env": {
    "GITHUB_PERSONAL_ACCESS_TOKEN": "<ВАШ_GITHUB_PERSONAL_ACCESS_TOKEN>"
  }
}
```

* AI-ассистент запускает официальный сервер `@modelcontextprotocol/server-github` напрямую через Node.js (без необходимости запускать Docker).
* Ассистент может по вашей команде автоматически:
  * Создавать новые репозитории: *«Создай репозиторий Name на GitHub»*.
  * Пушить файлы и изменения в ветки.
  * Создавать коммиты, Issue, Pull Request и релизы.

---

## 3. Системные переменные окружения

В профиле пользователя Windows установлены переменные:
* `GH_TOKEN`
* `GITHUB_TOKEN`

Это позволяет использовать GitHub API в скриптах PowerShell без ручной передачи заголовков:
```powershell
# Пример быстрой проверки профиля через PowerShell
$headers = @{ "Authorization" = "token $env:GH_TOKEN" }
Invoke-RestMethod -Uri "https://api.github.com/user" -Headers $headers
```

---

## 4. Как обновить токен в будущем (при истечении срока)

Если срок действия токена истечет или вы выпустите новый токен, обновите его одной командой в PowerShell:

```powershell
$newToken = "ВАШ_НОВЫЙ_ТОКЕН"

# 1. Обновление в Диспетчере Windows
cmdkey /generic:git:https://github.com /user:blacktapeproject263-alt /pass:$newToken

# 2. Обновление переменных окружения
[System.Environment]::SetEnvironmentVariable("GH_TOKEN", $newToken, "User")
[System.Environment]::SetEnvironmentVariable("GITHUB_TOKEN", $newToken, "User")

# 3. В файле mcp_config.json обновите значение GITHUB_PERSONAL_ACCESS_TOKEN
```
