# Olist: CTE-pipeline ознак покупців, recursive CTE та Materialized View

Навчальна робота з теми 5 (CTE, recursive CTE, Materialized View, EXPLAIN ANALYZE).
Готовий ноутбук із результатами клітинок — [notebooks/goit-rdb-hw-05.ipynb](notebooks/goit-rdb-hw-05.ipynb).

## Дані

Джерело: [Olist Brazilian E-commerce Public Dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce), версія 2.
Ліцензія датасету — CC BY-NC-SA 4.0. Беремо п'ять таблиць повністю, без вибірки.

| Typed-таблиця | CSV | Рядків |
| --- | --- | ---: |
| olist_customers | olist_customers_dataset.csv | 99 441 |
| olist_orders | olist_orders_dataset.csv | 99 441 |
| olist_order_items | olist_order_items_dataset.csv | 112 650 |
| olist_products | olist_products_dataset.csv | 32 951 |
| olist_sellers | olist_sellers_dataset.csv | 3 095 |

CSV лежать у `data/olist/`, їхні розміри та SHA-256 записано в `data/manifest.json`.
Загальний розмір п'яти файлів — приблизно 42 MiB, кожен менший за 100 MiB, тому вони у репозиторії.
Якщо раптом файлів немає поруч, ноутбук сам довантажує ту саму версію через `kagglehub`
(без облікових даних: `kagglehub.dataset_download('olistbr/brazilian-ecommerce/versions/2', path=...)`).
Якщо поруч є `data/manifest.json`, після завантаження звіряється SHA-256, щоб ознаки не порахувались на іншій версії файлів.

## Шари даних

- **raw** (`*_raw`) — копія CSV, усі колонки як `TEXT`, без ключів і перевірок.
- **typed** (`olist_*`) — контрольована схема: типи, `PRIMARY KEY`, `FOREIGN KEY`, `CHECK`.
  Саме typed-таблиці використовуються в пайплайні. У CSV є помилкові назви
  `product_name_lenght` і `product_description_lenght` — у typed-схемі вони нормалізовані.
- **службове** (`hw5_referrals`) — синтетичний referral-граф для recursive CTE. У Olist такого
  зв'язку немає, це навчальна імітація: ланцюг на 25 вершин, детермінований seed без `RANDOM()`.
- **features** (`mv_olist_customer_features`) — Materialized View з підсумковими ознаками покупця.

## Етапи CTE-pipeline

| Stage | Що робить |
| --- | --- |
| `reference_date` | стабільна дата відліку `MAX(order_purchase_timestamp)::DATE + 1` з самого датасету |
| `stage1_orders_total` | замовлення + сума (ціна + доставка) + статус |
| `stage2_rfm` | RFM на покупця: `recency_days`, `frequency`, `monetary` |
| `stage3_cohort` | `cohort_quarter` через `DATE_TRUNC('quarter', first_order_date)` |
| `stage4_rolling_aov` | rolling 30-day AOV вікном `RANGE BETWEEN INTERVAL '30 days' PRECEDING` |
| `stage5_cancel_rate` | `cancellation_rate` як `AVG(CASE WHEN ...)` по статусах |
| `stage6_category_revenue` | revenue за парою покупець × категорія |
| `stage7_preferred_category` | top-1 категорія покупця через `ROW_NUMBER()` |

Фінальний SELECT дає по одному рядку на покупця; ознаки закриті `COALESCE`, щоб покупець без
замовлень або без позицій у замовленні мав явний нуль чи `(none)`, а не `NULL`.

## Запуск у Google Colab

1. Відкрийте [Google Colab](https://colab.research.google.com/) і завантажте ноутбук через File → Upload notebook.
2. Оберіть Python 3.11 або 3.12 на Linux (для `pgserver`); GPU не потрібен.
3. Виконайте Runtime → Restart session and run all. Перша секція сама поставить залежності й підніме PostgreSQL.
4. Дані ноутбук шукає сам. Якщо перед запуском склонувати репозиторій у `/content`
   (`git clone` в окремій тимчасовій клітинці), він візьме CSV з `/content/goit-rdb-hw-05/data/olist`
   і звірить їх із `data/manifest.json`. Якщо завантажено лише сам ноутбук, CSV довантажаться
   через KaggleHub без облікових даних, а звірку SHA-256 буде пропущено, бо поруч немає маніфесту.
5. Збережіть ноутбук разом з результатами клітинок.

## Локальний запуск

Linux, Python 3.11–3.12, залежності з [requirements.txt](requirements.txt):

```bash
python -m pip install -r requirements.txt
jupyter lab notebooks/goit-rdb-hw-05.ipynb
```

Можна й без інтерфейсу:

```bash
jupyter nbconvert --to notebook --execute --inplace notebooks/goit-rdb-hw-05.ipynb
```

## Що перевіримо у ноутбуці

- Raw- і typed-таблиці мають однакову кількість рядків; зовнішні ключі та `CHECK` працюють.
- Пайплайн дає рівно один рядок на покупця, `monetary` збігається з сумою по позиціях,
  `recency_days` не від'ємна і рахована від стабільної дати відліку.
- Recursive CTE має anchor через `NOT EXISTS`, `path_ids`, `depth` і два захисти; експеримент з
  циклом показує, що трапляється без кожного з них.
- MV створено на повному пайплайні, є `UNIQUE INDEX`, `REFRESH ... CONCURRENTLY` проходить;
  окремо демонструємо помилку без унікального індексу.
- `EXPLAIN (ANALYZE, BUFFERS)` наведено для повного пайплайна, lookup з MV і top-N з MV.

## Як читати метрики

Усі суми в BRL. Ознаки рахуються на всіх статусах замовлень, включно з `canceled`, бо
`cancellation_rate` — частина сигналу. Сума замовлення включає доставку (`price + freight_value`).
Це навчальна матриця ознак, а не готовий feature store для продакшну.
