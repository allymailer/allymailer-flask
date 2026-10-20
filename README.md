# AllyMailer Flask 1.0.0

Web API 1.90

Покроковий посібник: [Надсилання листів на Python](https://docs.allymailer.com.ua/ua/send-with-python) · Документація для LLM: [llms.txt](https://docs.allymailer.com.ua/llms.txt) · [Довідник API](https://docs.allymailer.com.ua/ua/web-api-ref)

Інтеграція [Python SDK AllyMailer](https://github.com/allymailer/allymailer-python-sdk) (``) з Flask: клієнт API в extension (`app.extensions`), ендпоінт вебхуків із перевіркою підпису.

## Встановлення

```bash
pip install allymailer-flask
```

Вимоги: Python 3.8+, Flask 2.3+.

## Отримайте API-ключ

1. Зареєструйтеся на [cp.allymailer.com](https://cp.allymailer.com).
2. Скопіюйте API-ключ у [панелі керування](https://cp.allymailer.com).
3. Покладіть його у змінну оточення:

```bash
export ALLYMAILER_API_KEY="your-api-key"
```

Пакет бере його з `ALLYMAILER_API_KEY` (див. налаштування нижче).

> **Лише на сервері.** Використовуйте цей пакет у серверному коді: API-маршрути, серверні функції, воркери.
> Ніколи не передавайте API-ключ у браузер чи мобільний застосунок і не комітьте його.

## Налаштування

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

## Перший лист — без налаштування DNS

Кожен обліковий запис може одразу надсилати тестові листи із **sandbox-домену**: він уже підтверджений, DKIM і SPF налаштовані.
Листи доставляються лише на підтверджені вами адреси.

У прикладах — клієнт API з пакета (див. налаштування вище).

**1. Отримайте sandbox-домен** (`domain/get-sandbox-domain`). Перший виклик створює домен, наступні повертають той самий.

```python
from allymailer_wrapper import get_client


@app.post("/sandbox-domain")
def get_sandbox_domain():
    response = get_client().domain.get_sandbox_domain()
    return {"domain": response.domain}
```

**2. Підтвердіть свою скриньку** (`confirmed-email/create`). AllyMailer надішле лист із підтвердженням; перейдіть за посиланням із нього протягом 48 годин.

```python
from allymailer_wrapper import get_client
from allymailer.api.confirmed_email import ConfirmedEmailRequest


@app.post("/confirm-email")
def confirm_email():
    response = get_client().confirmed_email.create(ConfirmedEmailRequest(
        email="you@example.com",
    ))
    return {"status": response.status}
```

Статус перевірте через `confirmed-email/list` — у вашої адреси має бути `"status": "confirmed"`.

**3. Надішліть лист** (`email/send`) з будь-якої адреси sandbox-домену на підтверджену адресу.

```python
from allymailer_wrapper import get_client
from allymailer.api.email import EmailSendRequest, MessageObject, MessageObjectRecipientsItem
from allymailer.common import BodyObject


@app.post("/send")
def send():
    response = get_client().email.send(EmailSendRequest(
        message=MessageObject(
            recipients=[
                MessageObjectRecipientsItem(
                    email="you@example.com",
                ),
            ],
            body=BodyObject(
                html="<p>Працює!</p>",
            ),
            subject="Привіт від AllyMailer",
            from_email="noreply@<sandbox-domain>",
        ),
    ))
    return {"job_id": response.job_id}
```

### Що повертається

```json
{
  "status": "success",
  "job_id": "1ZymBc-00041N-9X",
  "emails": [
    "you@example.com"
  ]
}
```

У разі помилки API повертає `status: "error"`, `message` і числовий `code`:

```json
{
  "status": "error",
  "code": 102,
  "message": "Error ID:83274F98-C7D5-11F1-B221-6214FF8707B1. Can not decode key wrongkey"
}
```

### Те саме через curl

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

## Клієнт API

Усі розділи API доступні через клієнт, див. [документацію SDK](https://github.com/allymailer/allymailer-python-sdk).

```python
from allymailer_wrapper import get_client


@app.get("/usage")
def usage():
    info = get_client().system.info()
    return {"emails_sent": info.accounting.emails_sent if info.accounting else None}
```

## Надсилання через API

Клієнт викликає [email/send](https://docs.allymailer.com.ua/ua/web-api-ref#email-send) з усіма його параметрами: шаблони, підстановки, метадані, трекінг. Поля запиту та інші приклади — у [документації Python SDK](https://github.com/allymailer/allymailer-python-sdk/blob/HEAD/docs/Email.md#send).

```python
from allymailer_wrapper import get_client
from allymailer.api.email import EmailSendRequest, MessageObject, MessageObjectRecipientsItem
from allymailer.common import BodyObject


@app.post("/send")
def send():
    response = get_client().email.send(EmailSendRequest(
        message=MessageObject(
            recipients=[
                MessageObjectRecipientsItem(
                    email="user@example.com",
                    substitutions={
                        "CustomerId": "12452",
                        "to_name": "John Smith",
                    },
                ),
            ],
            body=BodyObject(
                html="<b>Hello, {{to_name}}</b>",
                plaintext="Hello, {{to_name}}",
            ),
            subject="AllyMailer test email ",
            from_email="email@example.com",
            from_name="John Smith",
        ),
    ))
    return {"job_id": response.job_id}
```

## Обмеження sandbox

- Надсилання лише на підтверджені адреси: до 10 на обліковий запис.
- До 100 листів на день.
- Sandbox-домен спливає через 31 день; знову викличте `domain/get-sandbox-domain`, щоб отримати новий.
- Працює лише з API-ключем облікового запису, не з ключем проєкту.
- Не показується в `domain/list`; інші методи `domain/*` до нього не застосовуються.

Sandbox — для розробки та тестування. Не використовуйте sandbox-домен як відправника на проді.

## Перед продом: підтвердіть свій домен

1. `domain/get-dns-records` — повертає значення для DNS-записів.
2. Додайте ці записи у свого DNS-провайдера (їх також показує сторінка домену в панелі керування):

   | Тип | Хост | Значення |
   |---|---|---|
   | TXT | `@` | значення `verification-record` (`allymailer-validate-hash=…`) |
   | TXT | `<селектор>._domainkey` (селектор — на сторінці домену в панелі керування, наприклад `us`) | `v=DKIM1; k=rsa; p=<dkim>` |
   | TXT | `@` | `v=spf1 include:spf.allymailer.com ~all` |
   | CNAME | `_dmarc` | значення `dmarc` |

3. `domain/validate-verification-record`, потім `domain/validate-dkim`.
4. `domain/list` — дочекайтеся, поки запис підтвердження буде підтверджено, а DKIM — активний.
5. Беріть `from_email` з конфігурації, щоб перехід із sandbox не потребував змін у коді.

## Надійне надсилання

- **Повтори:** ставте `message.idempotence_key` у надсиланнях, які можуть повторюватися, щоб повторний запит не надіслав лист двічі.
- **Ліміти:** до 500 отримувачів і 10 МБ на запит; кожне вкладення до 7 МБ.
- **Тіло:** завжди передавайте `body.html` або `body.plaintext`.
- **Статус доставки:** зареєструйте вебхук (`webhook/set`) зі швидким ідемпотентним HTTPS-обробником замість опитування.
- **Типові налаштування:** не прибирайте стандартний блок відписки та трекінг; змінити їх можна через підтримку AllyMailer.
- **Помилки:** повторюйте `429` і `5xx` з backoff; інші помилки без змін не повторюйте.

## Посилання

- [Надсилання листів на Python](https://docs.allymailer.com.ua/ua/send-with-python) та інші [посібники з інтеграції](https://docs.allymailer.com.ua/ua/integrations/)
- [Довідник Web API](https://docs.allymailer.com.ua/ua/web-api-ref)
- [llms.txt](https://docs.allymailer.com.ua/llms.txt) для AI-асистентів
- [Підтримка](https://cp.allymailer.com/ua/support)

## Для розробників пакета

Див. [AGENTS.md](AGENTS.md): як цей пакет генерується з OpenAPI-специфікації, збирається, перевіряється та випускається.
