# Этап 6 — Honeypot & Traps

Ловушки для ботов, скрытые поля в формах, автоматический перманентный бан. Интегрирован с админкой и Cloudflare.

---

## Что внутри

### 6.1: Honeypot Endpoints (30+ ловушек)
Боты и краулеры автоматически сканируют стандартные пути — `/robots.txt`, `/wp-admin`, `/.env`. Наше приложение никогда туда не лезет. Кто зашёл — автобан.

Ловушки: `/robots.txt`, `/sitemap.xml`, `/admin`, `/wp-admin`, `/wp-login.php`, `/.env`, `/config.php`, `/.git/config`, `/phpmyadmin`, `/xmlrpc.php`, `/login`, `/debug`, `/server-status`, `/backup`, `/dump.sql` + ещё 15 путей.

Работает на двух уровнях:
- **CF Worker (Edge)** — блокирует на Edge за 2ms, до сервера не доходит
- **FastAPI (Backend)** — ловит + банит IP в Redis на 1 год

### 6.2: Honeyfields (скрытые поля в формах)
Белая страница (заглушка) содержит форму с hidden-полями. Бот их заполняет, человек — нет.

Проверки:
- `security_confirm` заполнен → бот → бан
- `email_verify` заполнен → бот → бан
- `__hp_ts` пустой → нет JS → бот → бан
- Отправка за < 500ms → слишком быстро → бот → бан

Все боты получают "Thank you!" и не подозревают что попались.

### 6.3: Автобан "Чёрная дыра" (3 уровня)

| Уровень | Хранилище | TTL | Эффект |
|---------|----------|-----|--------|
| Redis | `honeypot:ban:{ip}` | 1 год | Мгновенная проверка → silent redirect |
| PostgreSQL | `banned_ips` таблица | Навсегда | Перманентная запись |
| Cloudflare | Access Rules | Навсегда | IP блокируется на Edge — сайт "умирает" |

### 6.4: Интеграция с админкой
Вкладка "Пользователи" → "Чёрный список IP":
- Показывает реальные забаненные IP с бейджем **Honeypot**
- Причина: `honeypot:/robots.txt`, `honeyfield:security_confirm filled`
- Кнопка "Разбанить" → удаляет из всех 3 уровней

---

## Файлы

| Файл | Описание |
|------|---------|
| `honeypot.py` | 30+ trap endpoints + POST /api/form honeyfield |
| `honeypot_ban.py` | Redis + PostgreSQL + CF API автобан |
| `db_models.py` | BannedIP модель |

Эти файлы интегрированы в scoring-engine (redzov/scoring-engine).

---

## Тестирование

```bash
# Honeypot trap → автобан
curl https://api.threeamigosteam.com/engine/robots.txt -H "X-Forwarded-For: 1.2.3.4"
# → IP забанен

# Honeyfield → автобан
curl -X POST https://api.threeamigosteam.com/engine/api/form \
  -d "email=bot@x.com&security_confirm=spam&__hp_ts="
# → IP забанен

# Список банов
curl https://api.threeamigosteam.com/engine/api/bans/honeypot

# Разбан
curl -X DELETE https://api.threeamigosteam.com/engine/api/bans/honeypot/1.2.3.4
```
