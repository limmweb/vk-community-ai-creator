import gspread
from oauth2client.service_account import ServiceAccountCredentials
from datetime import datetime, timedelta
import time
import random
import openai
import vk_api
from vk_api.exceptions import ApiError, Captcha
import requests
import re
import logging
import os

# Настройка логирования
logging.basicConfig(level=logging.INFO, format='[%(levelname)s] - %(asctime)s - %(message)s')
logger = logging.getLogger(__name__)

# Google Sheets
scope = ["https://spreadsheets.google.com/feeds", "https://www.googleapis.com/auth/drive"]
if not os.path.exists("credentials.json"):
    raise FileNotFoundError(f"Файл 'credentials.json' не найден в текущей директории: {os.getcwd()}")
creds = ServiceAccountCredentials.from_json_keyfile_name("credentials.json", scope)
client = gspread.authorize(creds)

# Подключение к OpenAI и VK
api_key = "" Введите ваш ключ OpenAI
vk_token = "" Введите ваш ключ ВК
vk_session = vk_api.VkApi(token=vk_token)
vk = vk_session.get_api()
openai_client = openai.OpenAI(api_key=api_key)

# 2Captcha API ключ
CAPTCHA_API_KEY = "" Ваш 2captcha key

# Инициализация таблиц
def init_sheets():
    spreadsheet = client.open("Введите сюда название Google Таблицы для работы")
    sheets = {
        "communities": ["Community_ID", "Title", "Description", "Pinned_Post", "VK_Group_ID", "Creator_Name", "Creator_ID", "Created_At", "Tone_And_Style", "Additional_Instructions"],
        "monthly_content_plan": ["Community_ID", "Month_Theme", "Status", "Created_At", "Tokens_In", "Tokens_Out", "Tokens_Total", "Tokens_Per_Theme", "Tone_And_Style", "Additional_Instructions", "Tokens_Cost"],
        "daily_content_plan": ["Community_ID", "Month_Theme", "Day_Theme", "Status", "Created_At", "Publish_Time", "Tokens_In", "Tokens_Out", "Tokens_Total", "Tokens_Per_Theme", "Tone_And_Style", "Additional_Instructions", "Tokens_Cost"],
        "posts": ["Community_ID", "Month_Theme", "Day_Theme", "Content", "Publish_Date", "Publish_Time", "VK_Post_ID", "Status", "Created_At", "Tokens_In", "Tokens_Out", "Tokens_Total", "Tone_And_Style", "Additional_Instructions", "Tokens_Cost"]
    }
    
    for sheet_name, headers in sheets.items():
        try:
            worksheet = spreadsheet.worksheet(sheet_name)
            existing_headers = worksheet.row_values(1)
            current_cols = worksheet.col_count
            required_cols = len(headers)

            logger.info(f"Проверка таблицы '{sheet_name}': текущие столбцы = {current_cols}, требуемые = {required_cols}")

            if current_cols < required_cols:
                worksheet.add_cols(required_cols - current_cols)
                worksheet = spreadsheet.worksheet(sheet_name)
                logger.info(f"Расширено количество столбцов в '{sheet_name}' с {current_cols} до {required_cols}")

            for header in headers:
                if header not in existing_headers:
                    col_idx = len(existing_headers) + 1
                    if col_idx > worksheet.col_count:
                        worksheet.add_cols(1)
                        logger.info(f"Добавлен дополнительный столбец в '{sheet_name}' для '{header}'")
                    worksheet.update_cell(1, col_idx, header)
                    existing_headers.append(header)
                    logger.info(f"Добавлена колонка '{header}' в таблицу '{sheet_name}'")
            logger.info(f"Таблица '{sheet_name}' готова к использованию")
        except gspread.exceptions.WorksheetNotFound:
            worksheet = spreadsheet.add_worksheet(title=sheet_name, rows=1000, cols=len(headers))
            worksheet.append_row(headers)
            logger.info(f"Создана таблица '{sheet_name}' с заголовками: {headers}")
    return spreadsheet

# Добавление строки с логированием
def append_row(sheet, row):
    sheet.append_row(row)
    logger.info(f"Добавлена строка в '{sheet.title}': {row}")

# Обновление статуса с логированием
def update_status(sheet, condition, status):
    records = sheet.get_all_records()
    for i, row in enumerate(records, start=2):
        if all(row[key] == value for key, value in condition.items()):
            sheet.update_cell(i, sheet.find("Status").col, status)
            logger.info(f"Обновлен статус в '{sheet.title}' для строки {i} на '{status}' с условием {condition}")

# Расчет стоимости токенов (в микродолларах)
def calculate_token_cost(tokens_in, tokens_out):
    cost = (tokens_in / 1_000_000 * 0.15 + tokens_out / 1_000_000 * 0.6) * 1_000_000  # В микродолларах (целое число)
    return int(cost)

# Обработка ошибок OpenAI
def openai_request(messages, max_tokens, retries=3):
    logger.info(f"Запрос к OpenAI: {messages}")
    for attempt in range(retries):
        try:
            response = openai_client.chat.completions.create(
                model="gpt-4o-mini",
                messages=messages,
                max_tokens=max_tokens,
            )
            return response
        except Exception as e:
            logger.error(f"Ошибка OpenAI: {e}. Попытка {attempt + 1}/{retries}")
            if attempt < retries - 1:
                time.sleep(600)
            else:
                logger.info("Ожидание 60 минут после неудачных попыток OpenAI")
                time.sleep(3600)
                return openai_request(messages, max_tokens, retries)

# Обработка ошибок VK с 2Captcha
def solve_captcha(captcha_url):
    for attempt in range(3):
        try:
            response = requests.post(
                "http://2captcha.com/in.php",
                data={"key": CAPTCHA_API_KEY, "method": "userrecaptcha", "googlekey": captcha_url.split("key=")[1].split("&")[0], "pageurl": captcha_url}
            )
            captcha_id = response.text.split("|")[1]
            time.sleep(10)
            for _ in range(10):
                result = requests.get(f"http://2captcha.com/res.php?key={CAPTCHA_API_KEY}&action=get&id={captcha_id}")
                if "CAPCHA_NOT_READY" not in result.text:
                    return result.text.split("|")[1]
                time.sleep(5)
        except Exception as e:
            logger.error(f"Ошибка 2Captcha: {e}. Попытка {attempt + 1}/3")
            if attempt == 2:
                logger.info("Ожидание 1 часа после неудачных попыток 2Captcha")
                time.sleep(3600)
    return None

def schedule_vk_post(vk, group_id, message, publish_date=None):
    max_attempts = 2  # Ограничим количество попыток
    attempt = 0
    
    while attempt < max_attempts:
        try:
            params = {"owner_id": -int(group_id), "message": message}
            if publish_date:
                now = int(time.time())
                max_future = now + 31536000  # 1 год в секундах
                if now < publish_date <= max_future:
                    params["publish_date"] = publish_date
                else:
                    logger.warning(f"Недопустимая дата публикации: {publish_date}. Публикуем немедленно.")
            response = vk.wall.post(**params)
            logger.info(f"Пост запланирован: {response}")
            return response
        
        except Captcha as e:
            logger.info(f"Требуется капча: {e.get_url()}")
            captcha_key = solve_captcha(e.get_url())
            if captcha_key:
                params["captcha_sid"] = e.sid
                params["captcha_key"] = captcha_key
            else:
                logger.info("Ожидание 1 часа после неудачи с капчей")
                time.sleep(3600)
                attempt += 1
        
        except ApiError as e:
            if e.code == 214:  # Ошибка 214: пост уже запланирован на это время
                logger.warning(f"Ошибка VK API: {e}. Время занято, сдвигаем на 10 минут.")
                if publish_date:
                    publish_date += 600  # Сдвиг на 10 минут (600 секунд)
                attempt += 1
            elif "invalid access_token" in str(e) or "user is blocked" in str(e):
                logger.error(f"Критическая ошибка VK: {e}. Остановка выполнения")
                exit()
            else:
                logger.error(f"Ошибка VK API: {e}. Ожидание 5 минут")
                time.sleep(300)
                attempt += 1
    
    logger.error(f"Не удалось запланировать пост после {max_attempts} попыток.")
    return None

# Очистка от Markdown (исправленная версия)
def clean_markdown(text):
    # Шаг 1: Защищаем упоминания VK
    vk_mentions = {}
    counter = 0
    
    # Находим все упоминания [id\d+|...] и [club\d+|...]
    def protect_mentions(match):
        nonlocal counter
        placeholder = f"__VKMENTION{counter}__"
        vk_mentions[placeholder] = match.group(0)
        counter += 1
        return placeholder
    
    text = re.sub(r'\[(id|club)\d+\|[^\[\]]*\]', protect_mentions, text.strip().replace("'", ""))
    
    # Шаг 2: Удаляем все оставшиеся [ и ] и двойные звёздочки **
    text = re.sub(r'[\[\]]', '', text)  # Удаляем квадратные скобки
    text = re.sub(r'\*\*', '', text)    # Удаляем двойные звёздочки
    
    # Шаг 3: Восстанавливаем упоминания VK
    for placeholder, mention in vk_mentions.items():
        text = text.replace(placeholder, mention)
    
    return text

# Получение данных сообщества
def get_community_info(community_id):
    sheet = spreadsheet.worksheet("communities")
    community = [row for row in sheet.get_all_records() if row["Community_ID"] == community_id]
    if community:
        return community[0]["Title"], community[0]["Description"], community[0]["Tone_And_Style"], community[0]["Additional_Instructions"], community[0]["Community_ID"]
    return None, None, None, None, None

# Выбор или создания сообщества
def select_or_create_community():
    sheet = spreadsheet.worksheet("communities")
    communities = sheet.get_all_records()
    if communities:
        print("Выберите уже созданное сообщество для продолжения или создайте новое:")
        for idx, comm in enumerate(communities, 1):
            print(f"{idx}: Название: {comm['Title']}")
        print("0: Создайте новое сообщество")
        while True:
            try:
                choice = int(input("Введите номер вашего выбора (0 для нового сообщества): "))
                if choice == 0:
                    return create_community()
                elif 1 <= choice <= len(communities):
                    return communities[choice-1]["Community_ID"]
                else:
                    print(f"Пожалуйста, выберите число от 0 до {len(communities)}.")
            except ValueError:
                print("Ошибка: введите целое число.")
    else:
        return create_community()

# Создание сообщества с данными пользователя
def create_community():
    sheet = spreadsheet.worksheet("communities")
    posts_sheet = spreadsheet.worksheet("posts")
    user_info = vk.users.get()[0]
    creator_name = f"{user_info['first_name']} {user_info['last_name']}"
    creator_id = user_info['id']
    
    theme = input("Ваше сообщество будет об: ")
    additional_instructions = input("Напишите дополнительные инструкции (оставьте пустым, если нет): ")
    
    messages = [
        {
            "role": "user",
            "content": (
                f"Текущая дата: {datetime.now().strftime('%A, %d %B, %Y')}\n"
                f"Сообщество ВК: [{{'Название': '{theme}', 'Описание': '', 'Тональность и Стиль': '', 'ID': ''}}]\n"
                f"Дополнительные Инструкции: {additional_instructions}\n"
                f"Задача: Создай Название (до 50 символов), Описание (от 100 до 500 символов), Текст закрепленного поста (от 100 до 500 символов) и Тональность и Стиль для ВК сообщества о '{theme}'. "
                "Закрепленный поста (от 100 до 500 символов) и Тональность и Стиль для ВК сообщества о '{theme}'. "
                "Не используй кавычки, не пиши слова Описание и Название буквально. Раздели через |. "
                "Учитывай информацию 'Сообщество ВК' и 'Дополнительные Инструкции', действуй предусмотрительно для будущих дат, опираясь на текущий момент."
            )
        }
    ]
    response = openai_request(messages, max_tokens=10000)
    title, description, pinned_post, tone_and_style = map(str.strip, response.choices[0].message.content.split("|"))
    vk_response = vk.groups.create(title=title, description=description, type='public', subtype=3)
    community_id = vk_response["id"]
    
    logger.info(f"[Добавление описания в группу] Описание группы {community_id} обновлено")
    vk.groups.edit(group_id=community_id, description=description)
    
    tokens_in, tokens_out, tokens_total = response.usage.prompt_tokens, response.usage.completion_tokens, response.usage.prompt_tokens + response.usage.completion_tokens
    tokens_cost = calculate_token_cost(tokens_in, tokens_out)
    append_row(sheet, [community_id, title, description, pinned_post, community_id, creator_name, creator_id, datetime.now().isoformat(), tone_and_style, additional_instructions])
    
    publish_dt = datetime.now() + timedelta(seconds=60)
    pin_response = schedule_vk_post(vk, community_id, pinned_post, int(publish_dt.timestamp()))
    if pin_response:
        try:
            logger.info(f"[Закрепление поста] Закрепленный пост опубликован и закреплен: {pin_response}")
            vk.wall.pin(owner_id=-community_id, post_id=pin_response["post_id"])
            append_row(posts_sheet, [community_id, "", "Закрепленный пост", pinned_post, publish_dt.isoformat(), publish_dt.strftime("%H:%M"), 
                                     pin_response["post_id"], "Done", datetime.now().isoformat(), tokens_in, tokens_out, tokens_total, tone_and_style, additional_instructions, tokens_cost])
        except ApiError as e:
            logger.error(f"[Закрепление поста] Не удалось закрепить пост: {e}")
    else:
        logger.error(f"[Закрепление поста] Не удалось опубликовать закрепленный пост")
    
    return community_id

# Генерация месячного плана
def generate_monthly_content_plan(community_id):
    sheet = spreadsheet.worksheet("monthly_content_plan")
    title, description, tone_and_style, additional_instructions, group_id = get_community_info(community_id)
    messages = [
        {
            "role": "user",
            "content": (
                f"Текущая дата: {datetime.now().strftime('%A, %d %B, %Y')}\n"
                f"Сообщество ВК: [{{'Название': '{title}', 'Описание': '{description}', 'Тональность и Стиль': '{tone_and_style}', 'ID': 'id{group_id}'}}]\n"
                f"Дополнительные Инструкции: {additional_instructions}\n"
                f"Задача: Сгенерируй 12 месячных тем для контент-плана ВК сообщества на основе темы '{title}'. "
                "Каждая тема рассчитана на 1 месяц и предназначена для ежедневной публикации (+1 день каждая тема) на 30 дней вперед. "
                "12 тем значит 12, ни больше, ни меньше"
                "Раздели темы через |. Учитывай информацию 'Сообщество ВК' и 'Дополнительные Инструкции', "
                "действуй предусмотрительно для будущих дат, опираясь на текущий момент. "
                "В ответ предоставь сразу готовый план без комментариев в начале и конце."
            )
        }
    ]
    
    logger.info(f"[Создание контент-плана (уровень КП)] Генерация месячного контент-плана для community_id={community_id}")
    response = openai_request(messages, max_tokens=10000)
    themes = [theme.strip() for theme in response.choices[0].message.content.split("|") if theme.strip()]
    tokens_in, tokens_out, tokens_total = response.usage.prompt_tokens, response.usage.completion_tokens, response.usage.prompt_tokens + response.usage.completion_tokens
    tokens_per_theme = round(tokens_total / len(themes))
    tokens_cost = calculate_token_cost(tokens_in, tokens_out)
    
    for theme in themes:
        append_row(sheet, [community_id, clean_markdown(theme), "Pending", datetime.now().isoformat(), tokens_in, tokens_out, tokens_total, tokens_per_theme, tone_and_style, additional_instructions, tokens_cost])
    logger.info(f"[Создание контент-плана (уровень КП)] Сгенерированы месячные темы для '{title}': {response.choices[0].message.content}")
    return themes

# Генерация дневного плана
def generate_daily_content_plan(community_id, month_theme):
    sheet = spreadsheet.worksheet("daily_content_plan")
    title, description, tone_and_style, additional_instructions, group_id = get_community_info(community_id)
    month_theme = month_theme.strip()
    messages = [
        {
            "role": "user",
            "content": (
                f"Текущая дата: {datetime.now().strftime('%A, %d %B, %Y')}\n"
                f"Сообщество ВК: [{{'Название': '{title}', 'Описание': '{description}', 'Тональность и Стиль': '{tone_and_style}', 'ID': 'id{group_id}'}}]\n"
                f"Дополнительные Инструкции: {additional_instructions}\n"
                f"Задача: Сгенерируй 30 дневных тем для контент-плана ВК сообщества на основе темы '{month_theme}'. "
                "Для каждой темы укажи в квадратных скобках [час:минуты] оптимальное время публикации в формате [HH:MM], "
                "определяемое на основе самой темы, дня недели, месяца, времени года и связности с другими темами в плане. "
                "Формат ответа должен быть: [HH:MM] Тема 1 | [HH:MM] Тема 2 | ... | [HH:MM] Тема 30. "
                "Учитывай информацию 'Сообщество ВК' и 'Дополнительные Инструкции', действуй предусмотрительно для будущих дат, "
                "опираясь на текущий момент. В ответ предоставь сразу готовый план без комментариев в начале и конце."
            )
        }
    ]
    
    logger.info(f"[Создание контент-плана (уровень КП)] Генерация дневного контент-плана для community_id={community_id}, month_theme='{month_theme}'")
    response = openai_request(messages, max_tokens=10000)
    raw_themes = [theme.strip() for theme in response.choices[0].message.content.split("|") if theme.strip()]
    tokens_in, tokens_out, tokens_total = response.usage.prompt_tokens, response.usage.completion_tokens, response.usage.prompt_tokens + response.usage.completion_tokens
    tokens_per_theme = round(tokens_total / len(raw_themes))
    tokens_cost = calculate_token_cost(tokens_in, tokens_out)
    
    themes = []
    for theme in raw_themes:
        match = re.search(r'\[(\d{2}:\d{2})\]', theme)
        publish_time = match.group(1) if match else "15:00"
        theme_text = clean_markdown(theme.replace(f"[{publish_time}]", ""))
        append_row(sheet, [community_id, month_theme, theme_text, "Pending", datetime.now().isoformat(), publish_time, tokens_in, tokens_out, tokens_total, tokens_per_theme, tone_and_style, additional_instructions, tokens_cost])
        themes.append((theme_text, publish_time))
    logger.info(f"[Создание контент-плана (уровень КП)] Сгенерированы дневные темы для '{month_theme}': {response.choices[0].message.content}")
    return themes

# Генерация поста
def generate_post(community_id, month_theme, day_theme, publish_time, day_offset, is_first=False):
    sheet = spreadsheet.worksheet("posts")
    title, description, tone_and_style, additional_instructions, group_id = get_community_info(community_id)

    # Определяем дату и время публикации
    if is_first:
        publish_dt = datetime.now() + timedelta(seconds=60 if day_theme == "Закрепленный пост" else 300)
        publish_time = publish_dt.strftime("%H:%M")
    else:
        base_date = datetime.now() + timedelta(days=day_offset)
        if not publish_time:
            publish_time = "15:00"
        try:
            hours, minutes = map(int, publish_time.split(":"))
            publish_dt = base_date.replace(hour=hours, minute=minutes, second=0, microsecond=0)
        except (ValueError, TypeError):
            publish_dt = base_date.replace(hour=15, minute=0, second=0, microsecond=0)
            publish_time = "15:00"

    # Генерация улучшенного контента с новым промптом
    messages = [
        {
            "role": "user",
            "content": (
                f"Дата публикации: {publish_dt.strftime('%A, %d %B, %Y')}\n"
                f"Информация о сообществе ВК: [{{'Название': '{title}', 'Описание': '{description}', 'Тональность и Стиль': '{tone_and_style}', 'ID': 'club{group_id}'}}]\n"
                f"Тема дня: '{day_theme}'\n"
                f"Дополнительные инструкции: {additional_instructions}\n\n"

                "Задача: Создай пост для указанного ВК-сообщества на заданную тему дня.\n"
                "Пожелания к тексту:\n"
                "- Форматирование без markdown **, HTML-тегов.\n"
                "- Минимум SEO-фраз, максимум пользы для людей.\n"
                "- Идеальная грамматика, пунктуация и логическая структура.\n"
                "- Уникальность контента, без плагиата.\n"
                "- Приветствуются эпиодические точные вставки на других языках для демонстрации экспертности.\n\n"

                "Обязательные элементы текста:\n"
                "- Нативный призыв поставить лайк или подписаться (например: 'если ты тоже любишь маму, поставь лайк').\n"
                "- Упоминание страницы автора ВКонтакте [id197539390|контекстное упоминание авторства], Автор - Александр Белмонт. \n"
                "- Упоминание сообщества [club{group_id}|упоминание сообщества в контексте], привлекающее новых подписчиков, не раздражая текущих.\n\n"

                "Учитывай текущую дату, информацию о сообществе, тональность и стиль, дополнительные инструкции.\n"
                "Верни готовый к публикации текст без комментариев."
            )
        }
    ]

    logger.info(f"[Генерация поста] community_id={community_id}, day_theme='{day_theme}'")
    response = openai_request(messages, max_tokens=10000)
    content = clean_markdown(response.choices[0].message.content)
    tokens_in, tokens_out, tokens_total = response.usage.prompt_tokens, response.usage.completion_tokens, response.usage.prompt_tokens + response.usage.completion_tokens
    tokens_cost = calculate_token_cost(tokens_in, tokens_out)

    # Планирование поста с ретраем при ошибке 214
    max_attempts = 5
    attempt = 0
    vk_response = None
    publish_timestamp = int(publish_dt.timestamp())

    while attempt < max_attempts and not vk_response:
        vk_response = schedule_vk_post(vk, community_id, content, publish_timestamp)
        if vk_response:
            publish_dt = datetime.fromtimestamp(publish_timestamp)
            publish_time = publish_dt.strftime("%H:%M")
            break
        else:
            publish_timestamp += 600
            attempt += 1
            logger.warning(f"Попытка {attempt}/{max_attempts} не удалась, новое время: {datetime.fromtimestamp(publish_timestamp)}")

    if vk_response:
        append_row(sheet, [community_id, month_theme, day_theme, content, publish_dt.isoformat(), publish_time,
                           vk_response["post_id"], "Done", datetime.now().isoformat(), tokens_in, tokens_out, tokens_total, tone_and_style, additional_instructions, tokens_cost])
        update_status(spreadsheet.worksheet("daily_content_plan"),
                      {"Community_ID": community_id, "Month_Theme": month_theme, "Day_Theme": day_theme},
                      "Done")
        logger.info(f"[Генерация поста] Запланирован пост на {publish_dt}")
    else:
        logger.error(f"[Генерация поста] Не удалось запланировать пост для '{day_theme}' после {max_attempts} попыток.")

    return vk_response

# Логика перезапуска
def get_last_progress(community_id):
    post_sheet = spreadsheet.worksheet("posts")
    day_sheet = spreadsheet.worksheet("daily_content_plan")
    month_sheet = spreadsheet.worksheet("monthly_content_plan")
    
    posts = post_sheet.get_all_records()
    day_records = day_sheet.get_all_records()
    month_records = month_sheet.get_all_records()
    
    # Находим последний запланированный пост для community_id
    community_posts = [p for p in posts if p["Community_ID"] == community_id and p["Publish_Date"]]
    if community_posts:
        last_post = max(community_posts, key=lambda x: x["Publish_Date"])
        last_publish_dt = datetime.fromisoformat(last_post["Publish_Date"])
        now = datetime.now()
        
        # Если последний пост в прошлом, начинаем с завтра
        if last_publish_dt < now:
            day_offset = 1  # Завтра
            month_theme = None
            day_theme = None
            logger.info(f"Последний пост в прошлом ({last_publish_dt}), начинаем с завтра: day_offset={day_offset}")
        else:
            # Считаем смещение от последнего поста
            day_offset = (last_publish_dt - now).days + 1  # +1 для следующего дня
            month_theme = last_post["Month_Theme"]
            day_theme = last_post["Day_Theme"]
            logger.info(f"Последний пост в будущем ({last_publish_dt}), продолжаем с day_offset={day_offset}")
    else:
        # Нет постов — начинаем с завтра
        day_offset = 1
        month_theme = None
        day_theme = None
        logger.info(f"Нет запланированных постов, начинаем с завтра: day_offset={day_offset}")
    
    # Проверяем незавершённые дни
    pending_days = [row for row in day_records if row["Community_ID"] == community_id and row["Status"] == "Pending" and row["Day_Theme"]]
    if pending_days and not day_theme:
        day_theme = pending_days[0]["Day_Theme"]
        month_theme = pending_days[0]["Month_Theme"]
        logger.info(f"Найден незавершенный день: month_theme={month_theme}, day_theme={day_theme}")
    
    # Проверяем незавершённые месяцы, если нет дня
    if not month_theme:
        pending_months = [row for row in month_records if row["Community_ID"] == community_id and row["Status"] == "Pending" and row["Month_Theme"]]
        if pending_months:
            month_theme = pending_months[0]["Month_Theme"]
            logger.info(f"Найден незавершенный месяц: month_theme={month_theme}")
        else:
            month_themes = [row["Month_Theme"] for row in month_records if row["Community_ID"] == community_id and row["Month_Theme"]]
            month_theme = month_themes[-1] if month_themes else None
            logger.info(f"Нет незавершенных задач: month_theme={month_theme}")
    
    return month_theme, day_theme, day_offset

def main():
    global spreadsheet
    spreadsheet = init_sheets()
    community_id = select_or_create_community()
    month_theme, day_theme, day_offset = get_last_progress(community_id)
    
    month_sheet = spreadsheet.worksheet("monthly_content_plan")
    day_sheet = spreadsheet.worksheet("daily_content_plan")
    
    month_themes = [row["Month_Theme"] for row in month_sheet.get_all_records() if row["Community_ID"] == community_id and row["Month_Theme"]]
    if not month_themes:
        month_themes = generate_monthly_content_plan(community_id)
    month_theme = month_themes[0] if not month_theme else month_theme
    
    post_counter = day_offset
    is_first_post = (post_counter == 0)
    month_theme_idx = month_themes.index(month_theme) if month_theme in month_themes else 0
    
    for m_theme in month_themes[month_theme_idx:]:
        m_theme = m_theme.strip()
        day_themes = [(row["Day_Theme"], row["Publish_Time"]) for row in day_sheet.get_all_records() 
                      if row["Community_ID"] == community_id and row["Month_Theme"] == m_theme and row["Day_Theme"]]
        if not day_themes and not day_theme:
            day_themes = generate_daily_content_plan(community_id, m_theme)
        elif day_theme:
            day_idx = [dt[0] for dt in day_themes].index(day_theme) if day_theme in [dt[0] for dt in day_themes] else 0
            day_themes = day_themes[day_idx:]
        
        for d_theme, publish_time in day_themes:
            generate_post(community_id, m_theme, d_theme, publish_time, post_counter, is_first_post)
            is_first_post = False
            post_counter += 1
            
            day_records = day_sheet.get_all_records()
            pending_days = [row for row in day_records if row["Community_ID"] == community_id and row["Month_Theme"] == m_theme and row["Status"] == "Pending" and row["Day_Theme"]]
            if not pending_days:
                update_status(month_sheet, {"Community_ID": community_id, "Month_Theme": m_theme}, "Done")
            
            if post_counter >= 360:
                logger.info(f"Достигнут лимит в 360 постов. Остановка выполнения.")
                return
        day_theme = None
    
    logger.info(f"Завершено с {post_counter} запланированными постами.")

if __name__ == "__main__":
    main()
