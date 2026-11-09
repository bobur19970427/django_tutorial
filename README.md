# Django Advanced — O'quv qo'llanma

> Ushbu qo'llanma Django asoslarini biladigan, lekin chuqur tushunmoqchi bo'lgan dasturchilar uchun.
> Har bir mavzu: **nazariya → kod misol → "tuzoq"lar (pitfalls) → amaliy topshiriq** tartibida yozilgan.

Versiya: **Django 5.x / 6.x**, Python 3.12+, PostgreSQL (asosiy) va SQLite.

---

## 1-modul: Modellar (Models)

| # | Mavzu | Fayl |
|---|-------|------|
| 1 | Model asoslari, ORM, migratsiyalar | [01-models/01-model-asoslari.md](01-models/01-model-asoslari.md) |
| 2 | Field options (`null`, `blank`, `default`, `choices` ...) | [01-models/02-field-options.md](01-models/02-field-options.md) |
| 3 | Barcha field turlari va ularning vazifasi | [01-models/03-field-turlari.md](01-models/03-field-turlari.md) |
| 4 | Bog'lanish fieldlari (`ForeignKey`, `OneToOne`, `ManyToMany`) | [01-models/04-relationship-fieldlar.md](01-models/04-relationship-fieldlar.md) |
| 5 | `Meta` class, constraint, index, model inheritance | [01-models/05-meta-va-inheritance.md](01-models/05-meta-va-inheritance.md) |
| 6 | Model metodlari (`save`, `clean`, `delete` ...) | [01-models/06-model-metodlari.md](01-models/06-model-metodlari.md) |
| 7 | Validatorlar va Custom Field yozish | [01-models/07-validator-va-custom-field.md](01-models/07-validator-va-custom-field.md) |
| 8 | Amaliy loyiha + topshiriqlar + intervyu savollari | [01-models/08-amaliyot.md](01-models/08-amaliyot.md) |

---

## Keyingi modullar (reja)

- 2-modul: QuerySet API va ORM (filter, Q, F, annotate, aggregate, subquery, N+1 muammo)
- 3-modul: Managers va Custom QuerySet
- 4-modul: Migratsiyalar chuqur (data migration, RunPython, squash)
- 5-modul: Signals, Transactions
- 6-modul: Views (FBV, CBV), Forms
- 7-modul: Django REST Framework

---

## Qanday o'qish kerak?

1. Har bir faylni ketma-ket o'qing — keyingi mavzu oldingisiga tayanadi.
2. Kod misollarni **o'zingiz yozib**, `python manage.py shell` da sinab ko'ring.
3. Har bir faylning oxiridagi **topshiriqlarni** albatta bajaring.
4. `sqlmigrate` buyrug'i bilan Django qanday SQL yaratayotganini doim tekshiring — bu ORM ni chuqur tushunishning eng yaxshi yo'li.
