# MVP веб-приложения учета аренды авто (суточный формат)

## 1) Рекомендуемый формат аренды для старта

**Рекомендация:** начинать с модели **60/40 без фиксированной ставки**.

Почему для MVP это лучше:
- минимальная логика расчета;
- проще интерфейс отчета водителя;
- меньше ошибок и спорных сценариев;
- быстрее запуск и проверка гипотезы.

Комбинированную модель (фикс + процент) лучше добавить после стабилизации процессов и накопления данных за 1–2 месяца.

---

## 2) Подход к реализации (vibe coding / no-code)

Подходит любой стек с визуальным CRUD + авторизацией + облачным хранилищем файлов:
- Supabase + low-code UI builder;
- Bubble;
- Glide;
- Retool/ToolJet (для админ-панели).

Критерии выбора платформы:
1. Роли и разграничение доступа (Admin/Driver);
2. Формулы и валидации без сложного кода;
3. Загрузка фото и хранение URL;
4. Триггеры/автоматизация (или cron/workflow) для проверки 7-дневного фотоконтроля.

---

## 3) Сущности данных (MVP)

### Drivers
- id
- name
- phone
- start_date
- car_id (nullable)
- status (`active` / `inactive`)
- balance (число)
- paid_total (число, по умолчанию 0)

### Cars
- id
- brand
- model
- plate_number
- current_mileage
- driver_id (nullable)
- status (`rented` / `free` / `repair`)

### DailyReports
- id
- driver_id
- car_id
- date
- total_income
- driver_share
- owner_share
- mileage_start
- mileage_end
- mileage_total
- income_photo_url
- mileage_photo_url
- created_at

### WeeklyInspection
- id
- car_id
- date
- front_photo
- back_photo
- interior_photo
- dashboard_photo
- created_at

---

## 4) Автоматические правила

### 4.1 При сохранении суточного отчета
1. Проверить обязательность фото (`income_photo_url`, `mileage_photo_url`);
2. Проверить `mileage_end >= mileage_start`;
3. Рассчитать:
   - `driver_share = total_income * 0.6`
   - `owner_share = total_income * 0.4`
   - `mileage_total = mileage_end - mileage_start`
4. Обновить:
   - `cars.current_mileage = mileage_end`
   - `drivers.balance = drivers.balance + owner_share`

### 4.2 Еженедельный фотоконтроль
Перед созданием нового `DailyReports`:
- найти последнюю запись `WeeklyInspection` по `car_id`;
- если записи нет или прошло более 7 дней — **запретить создание отчета** и показать CTA: «Пройдите фотоконтроль».

---

## 5) Роли и доступы

### Admin
- полный CRUD по Drivers/Cars;
- просмотр и корректировка DailyReports;
- просмотр WeeklyInspection и фото;
- просмотр агрегатов на дашборде.

### Driver
- создание своих DailyReports;
- создание WeeklyInspection по закрепленному авто;
- просмотр только своих отчетов и текущего баланса;
- без доступа к данным других водителей.

---

## 6) Экранная структура MVP

1. **Login** (email/phone + пароль)
2. **Driver: Daily Report Form**
3. **Driver: Weekly Inspection Form**
4. **Driver: My Balance**
5. **Admin: Dashboard**
6. **Admin: Drivers CRUD**
7. **Admin: Cars CRUD**
8. **Admin: Reports + Photo Viewer**

---

## 7) Формулы и виджеты дашборда (Admin)

- Доход владельца за сегодня = `SUM(owner_share WHERE date=today)`
- Доход владельца за неделю = `SUM(owner_share WHERE date>=today-6)`
- Общий доход владельца = `SUM(owner_share)`
- Авто без отчета за сегодня = Cars в `rented`, у которых нет DailyReports за today
- Авто без фотоконтроля = Cars, где `last_weekly_inspection_date < today-7` или отсутствует

---

## 8) Готовый prompt для AI-конструктора

```text
Сгенерируй web MVP для учета аренды авто в суточном формате.

Роли:
- Admin
- Driver

Сущности:
1) Drivers(id, name, phone, start_date, car_id, status[active|inactive], balance, paid_total)
2) Cars(id, brand, model, plate_number, current_mileage, driver_id, status[rented|free|repair])
3) DailyReports(id, driver_id, car_id, date, total_income, driver_share, owner_share, mileage_start, mileage_end, mileage_total, income_photo_url, mileage_photo_url, created_at)
4) WeeklyInspection(id, car_id, date, front_photo, back_photo, interior_photo, dashboard_photo, created_at)

Логика DailyReports:
- Обязательные поля: date, total_income, mileage_start, mileage_end, income_photo_url, mileage_photo_url
- Валидация: mileage_end >= mileage_start
- Формулы:
  driver_share = total_income * 0.6
  owner_share = total_income * 0.4
  mileage_total = mileage_end - mileage_start
- После сохранения:
  обновить Cars.current_mileage = mileage_end
  обновить Drivers.balance = Drivers.balance + owner_share

Логика WeeklyInspection:
- Перед созданием DailyReports проверить, что по car_id есть WeeklyInspection не старше 7 дней
- Если нет, блокировать создание DailyReports и показывать сообщение о необходимости фотоконтроля

Права доступа:
- Driver видит и создает только свои отчеты и фотоконтроль
- Admin имеет полный доступ ко всем данным

Экран Admin Dashboard:
- доход владельца за сегодня
- доход владельца за неделю
- общий доход владельца
- количество авто без отчета за сегодня
- количество авто без фотоконтроля

Сделай UI простым, mobile-friendly, без сложной аналитики и внешних интеграций.
```

---

## 9) Что добавить во 2-й итерации

- комбинированная модель (фикс + процент);
- учет штрафов;
- учет ТО и ремонтов;
- уведомления в WhatsApp/Telegram;
- базовые графики доходности по авто.
