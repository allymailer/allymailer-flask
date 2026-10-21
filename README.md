# AllyMailer Flask

[![Flask](https://img.shields.io/pypi/v/allymailer-flask)](https://pypi.org/project/allymailer-flask/) [![License: MIT](https://img.shields.io/badge/license-MIT-blue)](LICENSE)

Інтеграція [Python SDK AllyMailer](https://github.com/allymailer/allymailer-python-sdk) (``) з Flask: клієнт API в extension (`app.extensions`), ендпоінт вебхуків із перевіркою підпису.

[Довідник API](https://docs.allymailer.com.ua/ua/web-api-ref) · Документація для LLM: [llms.txt](https://docs.allymailer.com.ua/llms.txt)

## Встановлення

```bash
pip install allymailer-flask
```

Вимоги: Python 3.8+, Flask 2.3+.

## Налаштування

Отримайте API-ключ у [панелі керування](https://cp.allymailer.com) і зберігайте його у змінній оточення, а не в коді:

```bash
export ALLYMAILER_API_KEY="your-api-key"
```

За замовчуванням налаштування беруться зі змінних оточення:

```python
from flask import Flask
from allymailer_wrapper import AllyMailer

app = Flask(__name__)
app.config["ALLYMAILER_API_KEY"] = "YOUR_API_KEY"   # default: the environment variable
app.config["ALLYMAILER_HOST"] = "cp.allymailer.com"
AllyMailer(app)
```

| Параметр | Змінна оточення | За замовчуванням | Опис |
|---|---|---|---|
| `ALLYMAILER_API_KEY` | `ALLYMAILER_API_KEY` |  | API-ключ користувача або проєкту (обов'язковий). |
| `ALLYMAILER_HOST` | `ALLYMAILER_HOST` | `cp.allymailer.com` | Хост API (датацентр: `cp.allymailer.com`) або повний базовий URL. |
| `ALLYMAILER_TIMEOUT` | `ALLYMAILER_TIMEOUT` | `30` | Таймаут запиту, секунди. |
| `ALLYMAILER_WEBHOOK_KEY` | `ALLYMAILER_WEBHOOK_KEY` |  | API-ключ, яким підписані вебхуки (ключ користувача/проєкту вебхука), якщо він відрізняється від API-ключа. |
| `ALLYMAILER_WEBHOOK_PATH` | `ALLYMAILER_WEBHOOK_PATH` | `/allymailer/webhook` | Шлях ендпоінта вебхуків. |

`ALLYMAILER_WEBHOOK_ENABLED = False` вимикає маршрут вебхука. З `CSRFProtect` із Flask-WTF додайте виняток: `csrf.exempt("allymailer_wrapper._webhook")`.

## Вебхуки

Ендпоінт відповідає 200 OK на `GET` (перевірка URL під час [webhook/set](https://docs.allymailer.com.ua/ua/web-api-ref#webhook-set)) і приймає `POST`-сповіщення: перевіряє поле `auth` (MD5 тіла запиту, у якому значення `auth` замінено на API-ключ), на неправильний підпис відповідає 401, а сповіщення і кожну його подію передає застосунку.

```python
from allymailer_wrapper import webhook_event_received


@webhook_event_received.connect_via(app)
def on_event(sender, event, user, **kwargs):
    data = event.event_data
    app.logger.info("%s %s %s", event.event_name, data.email, data.status)
```

Зареєструйте URL ендпоінта методом `webhook/set`, наприклад `https://example.com/allymailer/webhook`.

> **Відповідайте 200 протягом 3 секунд**, а події обробляйте у фоні (через чергу): інакше AllyMailer повторює запит кожні 10 хвилин протягом доби (зростає `retry_count`), а обробник, на якому за добу не пройшов жоден запит хоча б для 10 подій, вимикається. Обробник має витримувати **щонайменше 5 паралельних запитів до 100 подій у кожному**; повтор надсилає ті самі події ще раз — прибирайте дублікати.

## Клієнт API

Усі розділи API доступні через клієнт, див. [документацію SDK](https://github.com/allymailer/allymailer-python-sdk).

```python
from allymailer_wrapper import get_client


@app.get("/usage")
def usage():
    info = get_client().system.info()
    return {"emails_sent": info.accounting.emails_sent if info.accounting else None}
```

### Надсилання через API

Клієнт викликає [email/send](https://docs.allymailer.com.ua/ua/web-api-ref#email-send) з усіма його параметрами: шаблони, підстановки, метадані, трекінг. Поля запиту та інші приклади — у [документації Python SDK](https://github.com/allymailer/allymailer-python-sdk/blob/HEAD/docs/Email.md#send).

```python
from allymailer.api.email import EmailSendRequest, MessageObject, MessageObjectRecipientsItem as Recipient
from allymailer.common import BodyObject
from allymailer_wrapper import get_client


@app.post("/send")
def send_email():
    client = get_client()
    response = client.email.send(EmailSendRequest(
        message=MessageObject(
            recipients=[Recipient.from_address("Олексій <alex@example.com>")],
            from_email="shop@example.com",
            from_name="Магазин",
            subject="Ваше замовлення",
            body=BodyObject.from_html("<p>Вітаємо, {{to_name}}! Ваше замовлення надіслано.</p>"),
        ),
    ))
    return {"job_id": response.job_id}
```

## Без свого домену: тестовий sandbox-домен

Поки ваш домен не підтверджено, листи можна надсилати із **sandbox-домену** — тестового домену, який є в кожного облікового запису: він уже підтверджений, DKIM і SPF налаштовані. Листи з нього доставляються лише на підтверджені вами адреси (посилання для підтвердження приходить листом і діє 48 годин), до 100 на день. Через 31 день після створення sandbox-домен видаляється, але його можна отримати знову: наступний виклик `domain/get-sandbox-domain` створить новий (з іншою назвою, тому беріть її з відповіді, а не прописуйте в коді).

У прикладах — клієнт API з пакета (див. налаштування вище).

```python
from allymailer.api.confirmed_email import ConfirmedEmailRequest
from allymailer.api.email import EmailSendRequest, MessageObject, MessageObjectRecipientsItem as Recipient
from allymailer.common import BodyObject
from allymailer_wrapper import get_client


@app.post("/sandbox")
def send_from_sandbox():
    client = get_client()

    # 1. Ваш sandbox-домен: створюється під час першого виклику, далі повертається той самий
    domain = client.domain.get_sandbox_domain().domain

    # 2. Підтвердіть свою скриньку: AllyMailer надішле посилання, перейдіть за ним протягом 48 годин
    client.confirmed_email.create(ConfirmedEmailRequest(email="you@example.com"))

    # 3. Надішліть із будь-якої адреси sandbox-домену на підтверджену адресу
    client.email.send(EmailSendRequest(
        message=MessageObject(
            recipients=[Recipient.from_address("you@example.com")],
            from_email=f"noreply@{domain}",
            subject="Привіт від AllyMailer",
            body=BodyObject.from_html("<p>Працює!</p>"),
        ),
    ))

    return {"domain": domain}
```

<details>
<summary>Обмеження sandbox</summary>

- Надсилання лише на підтверджені адреси: до 10 на обліковий запис.
- До 100 листів на день.
- Sandbox-домен спливає через 31 день; знову викличте `domain/get-sandbox-domain`, щоб отримати новий.
- Працює лише з API-ключем облікового запису, не з ключем проєкту.
- Не показується в `domain/list`; інші методи `domain/*` до нього не застосовуються.

Sandbox — для розробки та тестування. Не використовуйте sandbox-домен як відправника на проді.

</details>

<details>
<summary>Те саме через curl</summary>

```bash
BASE=https://cp.allymailer.com/ua/transactional/api/v1

curl -s -X POST $BASE/domain/get-sandbox-domain.json \
  -H "Content-Type: application/json" -H "X-API-KEY: $ALLYMAILER_API_KEY" -d '{}'

curl -s -X POST $BASE/confirmed-email/create.json \
  -H "Content-Type: application/json" -H "X-API-KEY: $ALLYMAILER_API_KEY" \
  -d '{"email": "you@example.com"}'

curl -s -X POST $BASE/email/send.json \
  -H "Content-Type: application/json" -H "X-API-KEY: $ALLYMAILER_API_KEY" \
  -d '{
        "message": {
          "recipients": [
            {
              "email": "you@example.com"
            }
          ],
          "from_email": "noreply@<sandbox-domain>",
          "subject": "Привіт від AllyMailer",
          "body": {
            "html": "<p>Працює!</p>"
          }
        }
      }'
```

</details>

## Запуск у продакшен

**1. Підтвердіть домен надсилання.** `domain/get-dns-records` повертає запис підтвердження, DKIM-ключ і адресу для DMARC; додайте записи у свого DNS-провайдера, потім `domain/validate-verification-record` і `domain/validate-dkim`.

| Тип | Хост | Значення | Звідки взяти |
|---|---|---|---|
| TXT | `@` | `allymailer-validate-hash=…` | поле `verification-record` відповіді, як є |
| TXT | `<селектор>._domainkey` (селектор — на сторінці домену в панелі керування, наприклад `us`) | `v=DKIM1; k=rsa; p=<dkim>` | поле `dkim` відповіді після `v=DKIM1; k=rsa; p=` |
| TXT | `@` | `v=spf1 include:spf.allymailer.com ~all` | у відповіді немає; якщо SPF-запис уже є, додайте include до нього |
| CNAME | `_dmarc` | `<домен>.dmarc.…` | поле `dmarc` відповіді (DMARC-політику публікує AllyMailer); якщо в домену вже є свій TXT-запис DMARC, залиште його |

**2. Надійне надсилання.**

| | |
|---|---|
| Повтори | `429` і `5xx` з backoff і тим самим `idempotence_key`; інші помилки виправляйте, а не повторюйте |
| Ліміти | 500 одержувачів і 10 МБ на запит разом із вкладеннями: у base64 файл стає приблизно на третину більшим |
| Тіло | завжди `html` або `plaintext` (або із шаблону) |
| Статус доставки | вебхук, а не опитування |
| Черга | надсилайте з фонового завдання, а не всередині веб-запиту |
| Налаштування | `from_email` з конфігурації: перехід із sandbox на продакшен без змін у коді |
| Типові налаштування | не прибирайте блок відписки та трекінг; змінити їх можна через підтримку AllyMailer |

## Допомога

- [Python SDK](https://github.com/allymailer/allymailer-python-sdk)
- Для ШІ-агентів: [llms.txt](https://docs.allymailer.com.ua/llms.txt), скіли: `npx skills add https://github.com/allymailer/allymailer-skills.git`
- [Підтримка](https://cp.allymailer.com/ua/support) · [Задачі](https://github.com/allymailer/allymailer-flask/issues)
- Розробникам: [AGENTS.md](AGENTS.md) — пакет генерується з OpenAPI-специфікації.
