# Скрипт автоматизации контента для сообществ ВКонтакте

Этот Python-скрипт автоматизирует создание, управление и публикацию контента для сообществ ВКонтакте. Он интегрируется с **Google Sheets** для планирования контента, **OpenAI** для генерации текстов, **VK API** для публикации постов и **2Captcha** для обработки капчи. Скрипт поддерживает создание сообществ, генерацию месячных и дневных контент-планов, а также планирование постов с учетом тем, тональности и стиля.

---

## Возможности скрипта

- **Создание сообщества**:
  - Создание нового сообщества ВКонтакте с автоматически сгенерированным названием, описанием и закрепленным постом на основе пользовательского ввода.
- **Планирование контента**:
  - Генерация контент-плана на 12 месяцев с месячными темами.
  - Создание 30-дневного дневного контент-плана для каждого месяца с оптимальным временем публикации.
- **Генерация и планирование постов**:
  - Автоматическая генерация постов с использованием модели **GPT-4o-mini** от OpenAI, адаптированных под тональность и стиль сообщества.
  - Планирование постов в ВКонтакте с обработкой ошибок (например, конфликт времени публикации, капча).
- **Интеграция с Google Sheets**:
  - Хранение данных о сообществах, контент-планах и постах в структурированной Google Таблице.
- **Обработка ошибок**:
  - Надежная обработка ошибок OpenAI, VK API и капчи с повторными попытками и задержками.
  - Логирование для отладки и мониторинга.
- **Расчет стоимости токенов**:
  - Отслеживание использования токенов OpenAI и расчет стоимости в микродолларах.
- **Отслеживание прогресса**:
  - Возобновление генерации контента с последнего запланированного поста для обеспечения непрерывности.

---

## Взаимосвязь контент-планов

Скрипт использует иерархическую структуру контент-планов, хранящихся в Google Sheets:

1. **Месячный контент-план** (`monthly_content_plan`):
   - Содержит 12 тем, каждая из которых охватывает один месяц.
   - Каждая тема связана с сообществом (`Community_ID`) и определяет общую направленность контента на месяц.
   - Статус темы обновляется на `Done`, когда все дневные темы для этого месяца обработаны.
   - Пример: Тема «Фитнес-мотивация» для сообщества о здоровье.

2. **Дневной контент-план** (`daily_content_plan`):
   - Для каждой месячной темы генерируется 30 дневных тем с указанием оптимального времени публикации (`Publish_Time`).
   - Дневные темы связаны с месячной темой и сообществом.
   - Статус дневной темы обновляется на `Done` после успешной генерации и планирования поста.
   - Пример: Дневная тема «Утренние упражнения» с временем публикации `[08:00]`.

3. **Посты** (`posts`):
   - Каждый пост связан с дневной и месячной темой, а также с сообществом.
   - Содержит текст поста, дату и время публикации, ID поста в ВКонтакте и информацию о токенах.
   - Пример: Пост с текстом «Начни день с зарядки! [club123|Наше сообщество]» и временем публикации.

**Как это работает**:
- Скрипт проверяет наличие месячного плана. Если его нет, генерирует 12 тем.
- Для каждой месячной темы создается дневной план с 30 темами и временем публикации.
- Для каждой дневной темы генерируется и планируется пост, который публикуется в указанное время.
- Прогресс отслеживается через Google Sheets, что позволяет возобновить работу с последнего запланированного поста, избегая дублирования.

---

## Требования

### Системные требования
- **Python**: Версия 3.8 или выше.
- **pip**: Менеджер пакетов Python для установки зависимостей.

### Зависимости
Установите необходимые библиотеки с помощью команды:

pip install gspread oauth2client openai vk_api requests 

## Настройка окружения

### 1. Настройка Google Sheets и сервисного аккаунта

#### 1.1. Создание Google аккаунта
Если у вас нет Google аккаунта, зарегистрируйтесь:

- Перейдите на [accounts.google.com](https://accounts.google.com).
- Нажмите «Создать аккаунт» и следуйте инструкциям, указав email, пароль и личные данные.
- Подтвердите аккаунт через email или телефон.
- Этот аккаунт будет использоваться для доступа к Google Диску и Google Cloud.

#### 1.2. Создание Google Таблицы
- Войдите в Google Диск: [drive.google.com](https://drive.google.com).
- Нажмите «Создать» → «Google Таблицы».
- Назовите таблицу `VK_Community_Content_Plan`.
- Скрипт автоматически создаст листы: `communities`, `monthly_content_plan`, `daily_content_plan`, `posts`.
- Убедитесь, что таблица доступна для редактирования сервисным аккаунтом (см. ниже).

#### 1.3. Создание сервисного аккаунта
- Перейдите в [Google Cloud Console](https://console.cloud.google.com).
- Создайте новый проект:
  - Нажмите на выпадающее меню проектов в верхнем левом углу и выберите «New Project».
  - Укажите название (например, `VKContentAutomation`) и создайте проект.
- Включите API:
  - Перейдите в «APIs & Services» → «Library».
  - Найдите и включите **Google Sheets API** и **Google Drive API**.
- Создайте сервисный аккаунт:
  - Перейдите в «Credentials» → «Create Credentials» → «Service Account».
  - Укажите имя (например, `vk-content-service`) и описание.
  - Пропустите шаги с ролями и пользователями (они не требуются).
  - Нажмите «Done» и сохраните email сервисного аккаунта (например, `vk-content-service@your-project.iam.gserviceaccount.com`).
- Создайте ключ JSON:
  - В разделе «Credentials» найдите созданный сервисный аккаунт и нажмите на него.
  - Перейдите во вкладку «Keys» → «Add Key» → «Create new key» → выберите «JSON».
  - Скачайте файл `credentials.json`.

#### 1.4. Размещение `credentials.json`
- Переместите скачанный файл `credentials.json` в корневую папку проекта, где находится скрипт `creator_new_stable.py`.
- Убедитесь, что файл имеет правильное имя (`credentials.json`) и находится в той же директории, что и скрипт.

### 1.5. Предоставление доступа к таблице
- Откройте Google Таблицу `VK_Community_Content_Plan` в Google Диске.
- Нажмите «Поделиться» (кнопка в правом верхнем углу).
- В поле «Добавить людей или группы» введите email сервисного аккаунта (например, `vk-content-service@your-project.iam.gserviceaccount.com`).
- Убедитесь, что выбрана роль «Редактор», и отправьте приглашение.
- Сервисный аккаунт автоматически получит доступ, уведомление не требуется.

### 2. Получение токена ВКонтакте
- **Получение токена через Kate Mobile**:
  - Используйте следующую ссылку для авторизации: [Получить токен ВКонтакте](https://id.vk.com/auth?return_auth_hash=11e9a6ff516853c86f&redirect_uri=https%3A%2F%2Foauth.vk.com%2Fblank.html&redirect_uri_hash=a0f0b14f8eedcafbe2&force_hash=1&app_id=2685278&response_type=token&code_challenge=&code_challenge_method=&scope=1040183263&state=)
  - **Важно**: Открывайте ссылку в браузере на компьютере (например, Chrome, Firefox). **Не используйте мобильное приложение ВКонтакте**, так как оно может не вернуть токен корректно.
  - Войдите в свой аккаунт ВКонтакте, разрешите доступ для приложения Kate Mobile.
  - После авторизации вы будете перенаправлены на страницу с URL вида:
    ```
    https://oauth.vk.com/blank.html#access_token=ВАШ_ТОКЕН&expires_in=0&user_id=ВАШ_ID
    ```
  - Скопируйте значение `access_token` (например, `vk1.a.To_Y114...`).
- **Добавление токена в скрипт**:
  - Откройте файл `creator_new_stable.py` в текстовом редакторе.
  - Найдите строку:
    ```python
    vk_token = "vk1.a.To_Y114..."
    ```
  - Замените значение на скопированный токен.
  - **Замените значение на скопированный токен**:
    - Значение `access_token`, которое вы скопировали, нужно вставить в переменную `vk_token`.
    - Убедитесь, что токен вставлен корректно, без лишних пробелов или символов.

### 3. Получение ключа OpenAI
- **Регистрация в OpenAI**:
  - Перейдите на [OpenAI Platform](https://platform.openai.com).
  - Нажмите «Sign Up» и создайте аккаунт, указав email, пароль и подтвердив номер телефона.
  - Если у вас уже есть аккаунт, войдите в систему.
- **Создание API-ключа**:
  - После входа перейдите в раздел «API Keys» (обычно доступен через боковое меню или [dashboard](https://platform.openai.com/account/api-keys)).
  - Нажмите «Create new secret key».
  - Скопируйте сгенерированный ключ (например, `sk-7oRdMZm...`).
  - **Важно**: Сохраните ключ в безопасном месте, так как он не будет отображаться повторно.
- **Добавление ключа в скрипт**:
  - Откройте файл `creator_new_stable.py`.
  - Найдите строку:
    ```python
    api_key = "sk-7oRdMZm..."
    ```
  - Замените значение на ваш API-ключ OpenAI.
- **Проверка лимитов**:
  - Убедитесь, что у вас достаточно кредитов на аккаунте OpenAI для использования модели `gpt-4o-mini`.
  - При необходимости пополните баланс в разделе «Billing».

### 4. Настройка 2Captcha
- **Регистрация в 2Captcha**:
  - Перейдите на [2Captcha](https://2captcha.com) и зарегистрируйтесь.
  - Подтвердите email и войдите в аккаунт.
- **Получение API-ключа**:
  - В личном кабинете найдите раздел «API Key» или «Account Settings».
  - Скопируйте ключ (например, `a32f5477...`).
- **Добавление ключа в скрипт**:
  - Откройте файл `creator_new_stable.py`.
  - Найдите строку:
    ```python
    CAPTCHA_API_KEY = "a32f5477..."
    ```
  - Замените значение на ваш ключ 2Captcha.
- **Пополнение баланса**:
  - Убедитесь, что на вашем аккаунте 2Captcha достаточно средств для обработки капчи (стоимость обычно ~$0.001 за капчу).

## Установка и запуск
- **Установите Python**:
  - Скачайте и установите Python 3.8+ с [python.org](https://www.python.org/downloads/).
  - Убедитесь, что `pip` установлен (обычно идет с Python).
  - Проверьте версии:
    ```bash
    python --version
    pip --version
    ```
- **Установите зависимости**:
  ```bash
  pip install gspread oauth2client openai vk_api requests
  ```
- **Скачайте скрипт**:
  - Сохраните файл `creator_new_stable.py` в рабочую папку.
- **Разместите `credentials.json`**:
  - Поместите файл `credentials.json` в ту же папку, где находится скрипт.
- **Запустите скрипт**:
  ```bash
  python creator_new_stable.py
  ```
  - Скрипт запросит выбор или создание сообщества и начнет генерацию контент-планов и постов.

## Работа с промптами
Скрипт содержит готовые промпты для OpenAI, которые определяют структуру и стиль генерируемого контента. Они находятся в функциях:
- `create_community` (для создания сообщества).
- `generate_monthly_content_plan` (для месячных тем).
- `generate_daily_content_plan` (для дневных тем).
- `generate_post` (для генерации постов).

### Что можно редактировать
- **Текст внутри `content`**:
  - Вы можете изменять инструкции, чтобы настроить тон, стиль или дополнительные требования. Например, в `generate_post`:
    ```python
    "Пожелания к тексту:\n"
    "- Форматирование без markdown **, HTML-тегов.\n"
    "- Минимум SEO-фраз, максимум пользы для людей.\n"
    "- Идеальная грамматика, пунктуация и логическая структура.\n"
    ```
    Можно добавить: «Используйте юмор» или «Добавьте больше эмодзи».
- **Дополнительные инструкции**:
  - Поле `additional_instructions` в промптах позволяет уточнить стиль или контекст. Например:
    ```python
    f"Дополнительные Инструкции: {additional_instructions}\n"
    ```
    Пользователь может ввести «Пишите в дружелюбном тоне» при создания сообщества.

### Критические части, которые нельзя менять
Эти элементы обеспечивают корректную работу скрипта и парсинг ответов OpenAI:
1. **Разделитель `|`**:
   - Используется для разделения тем или частей ответа. Пример:
     ```python
     "Раздели темы через |"
     ```
     Удаление или замена `|` нарушит разбиение ответа на части.
2. **Формат времени `[HH:MM]`**:
   - Необходим для извлечения времени публикации в дневных темах. Пример:
     ```python
     "[HH:MM] Тема 1 | [HH:MM] Тема 2"
     ```
     Изменение формата (например, на `(HH:MM)`) сломает регулярное выражение в `generate_daily_content_plan`.
3. **Упоминания ВКонтакте** (`[id...|...]` и `[club...|...]`):
   - Обеспечивают правильное форматирование ссылок. Пример:
     ```python
     "[id197539390|контекстное упоминание авторства]"
     "[club{group_id}|упоминание сообщества в контексте]"
     ```
     Удаление или изменение формата нарушит ссылки в постах.
4. **Переменные в фигурных скобках**:
   - Поля вроде `{title}`, `{description}`, `{publish_dt}` заменяются динамически. Пример:
     ```python
     f"Сообщество ВК: [{{'Название': '{title}', 'Описание': '{description}', 'Тональность и Стиль': '{tone_and_style}'}}]\n"
     ```
     Удаление скобок или изменение имен переменных приведет к ошибкам.
5. **Ограничения на количество тем**:
   - Промпты жестко задают количество тем (12 для месячного плана, 30 для дневного). Пример:
     ```python
     "12 тем значит 12, ни больше, ни меньше"
     ```
     Изменение этого числа нарушит логику работы с таблицами.

**Рекомендация**:
- Сохраните оригинальный промпт перед редактированием.
- Тестируйте изменения на небольшом количестве данных, чтобы убедиться, что парсинг работает корректно.
- Не удаляйте ключевые слова вроде `Раздели через |`, `[HH:MM]`, или упоминания ВКонтакте.

## Примечания
- **Лимит постов**: Скрипт останавливается после планирования 360 постов, чтобы избежать перегрузки.
- **Логирование**: Действия записываются в лог (уровень `INFO` и выше). Логи помогают отслеживать ошибки и прогресс.
- **Обработка ошибок**:
  - При ошибке VK API 214 (занятое время) скрипт сдвигает время на 10 минут.
  - При ошибках OpenAI или 2Captcha используются повторные попытки с задержками (до 60 минут).
- **Токены и стоимость**: Скрипт рассчитывает стоимость токенов OpenAI в микродолларах и сохраняет в таблице.

## Пример работы
1. Запустите скрипт:
   ```bash
   python creator_new_stable.py
   ```
2. Выберите или создайте сообщество:
   - Введите тему (например, «Фитнес и здоровье»).
   - Укажите инструкции (например, «Мотивационный тон»).
3. Скрипт:
   - Создаст сообщество ВКонтакте.
   - Сгенерирует месячный план (12 тем).
   - Создаст дневной план (30 тем для каждой месячной темы).
   - Запланирует посты с учетом времени публикации.
4. Проверьте:
   - Google Таблицу `VK_Community_Content_Plan` для просмотра планов и постов.
   - Сообщество ВКонтакте для подтверждения публикации.

## Устранение неполадок
- **Ошибка «Файл credentials.json не найден»**:
  - Проверьте, что `credentials.json` находится в той же папке, что и скрипт.
  - Убедитесь, что файл назван корректно (без лишних пробелов или расширений).
- **Ошибка VK API (invalid access_token)**:
  - Проверьте токен, убедитесь, что он получен через указанную ссылку.
  - Попробуйте получить новый токен.
- **Ошибка OpenAI**:
  - Проверьте API-ключ и наличие кредитов в аккаунте OpenAI.
  - Убедитесь, что модель `gpt-4o-mini` доступна.
- **Капча не решается**:
  - Проверьте баланс на 2Captcha и правильность API-ключа.
  - Убедитесь, что у вас достаточно средств (~$0.001 за капчу).
