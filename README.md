# 🏭 Weiwo

> **A Telegram-based B2B industrial directory for Uzbekistan.**
> Find manufacturers, equipment suppliers, and raw-material providers across all 14 regions — in Uzbek or English.

---

## ✨ Features

| Feature | Details |
|---|---|
| 🌐 **Bilingual** | Full Uzbek / English interface — every menu, prompt and button |
| 📍 **Regional search** | All 14 regions of Uzbekistan (Toshkent, Samarqand, Farg'ona, …) |
| 🏢 **Company directory** | Register companies with photo, Yandex Maps link, category & sub-category |
| 🔍 **Smart search** | Drill-down: Region → Category → Sub-category → Company |
| ⭐ **Crowd ratings** | "I worked here" / "I'm a partner" social-proof counters on every listing |
| 🔐 **Role-based access** | Only users with `admin` status can add new companies |
| 📱 **Contact onboarding** | One-tap phone-number sharing via Telegram's native contact button |

---

## 🗂️ Project Structure

```
Weiwo/
├── main.py                     # Entry point — executor, startup hook
├── config.py                   # DB DSN assembled from .env
├── states.py                   # FSM state groups
├── func_.py                    # i18n helpers (send_msg, send_msg_and_btns)
├── regions.csv                 # 14 Uzbek region names (Uzbek)
├── regions_en.csv              # 14 Uzbek region names (English)
│
├── dispatcher/
│   └── dispatcher.py           # Bot + Dispatcher + MemoryStorage init
│
├── database/
│   ├── database.py             # AsyncDatabaseSession + TableBase (active-record)
│   └── models/
│       ├── base.py             # Product — company listings table
│       ├── user.py             # User — accounts, lang, score, status
│       └── rating.py           # UserInCompany — worked / partner links
│
├── aiobot/
│   ├── buttons/
│   │   ├── inline.py           # All InlineKeyboardMarkup factories + CSV loaders
│   │   └── reply.py            # Contact-request ReplyKeyboard
│   ├── handlers/
│   │   ├── commands.py         # /start, language pick, phone onboarding
│   │   └── admin/              # (reserved for future admin panel)
│   └── services/
│       ├── add_company.py      # Multi-step FSM flow for company registration
│       └── search_company.py   # Multi-step FSM flow for company search + ratings
│
└── media/
    ├── img.png                 # "Not an admin" banner (Uzbek)
    └── img_1.png               # "Not an admin" banner (English)
```

---

## 🏗️ Architecture

```
Telegram API
     │
     ▼
 Dispatcher (aiogram 2.x)
     │
     ├─ /start handler  ──────────────► FSM: CreateAccount
     │       │                               lang → phone_number
     │       ▼
     │   User.create() → PostgreSQL
     │
     ├─ "Add company" callback ───────► FSM: AddCompany
     │       │                               city → name → category
     │       │                               → sub_category → yandex_url
     │       │                               → photo → description → confirm
     │       ▼
     │   Product.add_product() → PostgreSQL
     │
     └─ "Search company" callback ────► FSM: SearchCompany
             │                               city → category → sub_category → name
             ▼
         Product.get_company() → PostgreSQL
             │
             └─ Rating buttons (w_ / p_) → UserInCompany.add_staff_()
```

**Async all the way down:** aiogram's polling loop, SQLAlchemy `AsyncSession`, and `asyncpg` keep the bot non-blocking under concurrent users.

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Bot framework | [aiogram](https://github.com/aiogram/aiogram) 2.25.1 |
| Language | Python 3.10+ |
| Database | PostgreSQL |
| Async DB driver | asyncpg 0.27 |
| ORM | SQLAlchemy 2.0 (async) |
| Config | python-dotenv |
| Package manager | Poetry |

---

## 🚀 Getting Started

### Prerequisites

- Python 3.10+
- PostgreSQL running locally or remotely
- A Telegram Bot token from [@BotFather](https://t.me/BotFather)
- [Poetry](https://python-poetry.org/) installed

### 1 · Clone & install

```bash
git clone https://github.com/rashideveloperr/Weiwo.git
cd Weiwo
poetry install
```

### 2 · Configure environment

Create a `.env` file in the project root:

```env
TOKEN=your_telegram_bot_token

DB_HOST=localhost
DB_USER=postgres
DB_PASSWORD=yourpassword
DB_NAME=weiwo
```

### 3 · Run

```bash
poetry run python main.py
```

On first launch the bot auto-creates all database tables (`users`, `products`, `userincompanies`).

---

## 📋 FSM Flows

### Account creation (`/start`)
```
Language selection (🇺🇿 / 🇺🇸)
       └─► Share phone number (native contact button)
                 └─► Account created → Main menu
```

### Add a company *(admin only)*
```
Select region (14 options)
  └─► Company name
        └─► Category (Manufacturer / Equipment supplier / Raw materials)
              └─► Sub-category (multi-select + Next)
                    └─► Yandex Maps URL
                          └─► Photo (+ optional caption as description)
                                └─► Confirm (Yes ✅ / No 🚫)
                                      └─► Saved to DB
```

### Search a company
```
Select region
  └─► Category
        └─► Sub-category
              └─► Company list → tap to open
                    └─► Full card with photo + map link + rating buttons
```

---

## 🏢 Company Categories

**Categories**
- Manufacturer (`Ishlab chiqaruvchi`)
- Equipment supplier (`Uskuna ta'minotchi`)
- Supplier of raw materials (`Xom ashyo ta'minotchi`)

**Sub-categories**
- General equipment supplier
- Plastic and resin
- Textile / light industry
- Agro
- Metal
- Packing / marking / printing
- Industrial climate equipment
- Warehouse equipment

---

## 👤 User Roles

| Role | Can do |
|---|---|
| `user` | Search companies, rate listings |
| `admin` | Everything above + add new company listings |

Role is stored in `users.status` column. Promote a user manually in the database:

```sql
UPDATE users SET status = 'admin' WHERE user_id = '123456789';
```

---

## 📦 Database Schema

```
users
  pk, user_id (str), full_name, phone_number, score, lang, status, created_at

products
  pk, telegram_id, full_name, city, name, category, sub_category,
  yandex_maps_url, photo, description, explanation, created_at

userincompanies
  pk, telegram_id, company_id, type (worked | partner)
```

---

## ⚙️ Known Limitations

This project was built in early 2023 as a learning exercise. A few things to keep in mind if you fork it:

- **MemoryStorage** — FSM state is stored in RAM; restarting the bot clears all in-progress conversations. For production use [RedisStorage](https://docs.aiogram.dev/en/latest/dispatcher/fsm/storages.html).
- **aiogram v2** — The bot uses the 2.x API. aiogram 3.x introduced a significantly different router-based API and is now the recommended version.
- **Single session** — A single `AsyncSession` is shared across all requests. For high traffic, a session-per-request or connection pool pattern is preferable.
- **Admin promotion** — There is no in-bot admin management command; role changes require direct DB access.

---

## 📄 License

This project is open-source. Feel free to study, fork, and build on top of it.

---

<p align="center">Made with ❤️ for the industrial community of Uzbekistan</p>
