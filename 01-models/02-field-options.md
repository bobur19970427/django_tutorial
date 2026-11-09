# 2. Field options — barcha fieldlar uchun umumiy parametrlar

Har bir field turi (CharField, IntegerField ...) quyidagi **umumiy parametrlarni** qabul qiladi. Field turlarini o'rganishdan oldin bularni yaxshi bilish shart, chunki ular har joyda ishlatiladi.

Parametrlar ikki guruhga bo'linadi:

| Guruh | Ta'siri | Parametrlar |
|-------|---------|-------------|
| **Baza darajasi** (migratsiya yaratadi) | SQL jadval tuzilmasini o'zgartiradi | `null`, `unique`, `db_index`, `db_column`, `db_default`, `db_comment`, `primary_key`, `db_tablespace` |
| **Python / validatsiya darajasi** | Faqat Django formalari, admin, `full_clean()` ga ta'sir qiladi | `blank`, `choices`*, `validators`, `editable`, `help_text`, `verbose_name`, `error_messages`, `default` |

\* `choices` migratsiya fayliga yoziladi, lekin bazada hech qanday cheklov yaratmaydi.

---

## 2.1. `null` va `blank` — eng ko'p chalkashtiriladigan juftlik

```python
class Profile(models.Model):
    bio = models.TextField(blank=True)               # bo'sh string "" bo'lishi mumkin
    birth_date = models.DateField(null=True, blank=True)  # NULL bo'lishi mumkin
```

| Parametr | Nima qiladi | Daraja |
|----------|-------------|--------|
| `null=True` | Bazada ustun `NULL` qiymat qabul qiladi | **Baza** |
| `blank=True` | Forma/validatsiyada maydon bo'sh qoldirilishi mumkin | **Validatsiya** |

### Qoida (Django rasmiy tavsiyasi)

- **String fieldlar** (`CharField`, `TextField`, `SlugField`, `EmailField`, `URLField`) uchun `null=True` **ishlatmang**. Faqat `blank=True` yetarli. Aks holda "bo'sh qiymat" ning **ikki xil** ko'rinishi paydo bo'ladi: `NULL` va `""` — bu filter yozishda xatoliklarga olib keladi.
- **String bo'lmagan** fieldlar (`DateField`, `IntegerField`, `ForeignKey` ...) ixtiyoriy bo'lsa → `null=True, blank=True` birga.

```python
# YOMON
nickname = models.CharField(max_length=50, null=True, blank=True)
# Endi bo'sh nickname larni topish uchun:
Profile.objects.filter(Q(nickname__isnull=True) | Q(nickname=""))

# YAXSHI
nickname = models.CharField(max_length=50, blank=True)
Profile.objects.filter(nickname="")
```

**Istisno:** `CharField` da `unique=True` va `blank=True` bo'lsa, `null=True` kerak bo'ladi — chunki bir nechta `""` qiymat unique ni buzadi, `NULL` lar esa buzmaydi.

```python
# Masalan: ixtiyoriy, lekin kiritilsa unikal bo'lishi kerak bo'lgan telefon raqam
phone = models.CharField(max_length=20, unique=True, null=True, blank=True)
```

### 4 ta kombinatsiya

| `null` | `blank` | Ma'nosi |
|--------|---------|---------|
| False | False | Majburiy (default) |
| False | True | Formada ixtiyoriy, bazada `""` saqlanadi (string fieldlar uchun ideal) |
| True | True | To'liq ixtiyoriy (string bo'lmagan fieldlar uchun ideal) |
| True | False | Bazada NULL mumkin, lekin formada majburiy — kam ishlatiladi (masalan, API orqali to'ldiriladigan, lekin admin da majburiy) |

---

## 2.2. `default`

Yangi obyekt yaratilganda qiymat berilmasa ishlatiladi.

```python
import uuid
from django.utils import timezone

class Order(models.Model):
    status = models.CharField(max_length=20, default="new")
    quantity = models.PositiveIntegerField(default=1)
    code = models.UUIDField(default=uuid.uuid4)       # callable — QAVSSIZ!
    ordered_at = models.DateTimeField(default=timezone.now)  # callable — QAVSSIZ!
```

### ⚠️ Tuzoq 1: callable ni chaqirib yubormang

```python
# XATO — timezone.now() server ishga tushgan vaqtda BIR MARTA hisoblanadi,
# barcha obyektlarga bir xil vaqt yoziladi!
ordered_at = models.DateTimeField(default=timezone.now())

# TO'G'RI — har bir yangi obyekt uchun funksiya qayta chaqiriladi
ordered_at = models.DateTimeField(default=timezone.now)
```

### ⚠️ Tuzoq 2: mutable default (list, dict)

```python
# XATO — barcha obyektlar BITTA dict ni bo'lishadi
settings = models.JSONField(default={})

# TO'G'RI
settings = models.JSONField(default=dict)

# Murakkab default — modul darajasidagi funksiya (lambda EMAS! lambda migratsiyaga serialize bo'lmaydi)
def default_settings():
    return {"theme": "light", "notifications": True}

settings = models.JSONField(default=default_settings)
```

### `default` — bu Python darajasi!

`default` SQL ga yozilmaydi. Agar boshqa dastur (masalan, raw SQL yoki boshqa mikroservis) bazaga to'g'ridan-to'g'ri yozsa, default qo'llanmaydi. Buning uchun → `db_default`.

---

## 2.3. `db_default` (Django 5.0+)

Default qiymatni **bazaning o'zida** (`DEFAULT ...` SQL) belgilaydi.

```python
from django.db.models.functions import Now, Pi

class Event(models.Model):
    created_at = models.DateTimeField(db_default=Now())     # SQL: DEFAULT CURRENT_TIMESTAMP
    priority = models.IntegerField(db_default=0)            # SQL: DEFAULT 0
    is_active = models.BooleanField(db_default=True)
```

| | `default` | `db_default` |
|--|-----------|--------------|
| Qayerda hisoblanadi | Python | Baza (SQL) |
| Raw SQL INSERT da ishlaydimi | ❌ | ✅ |
| `save()` dan keyin obyektda qiymat bormi | ✅ darhol | ⚠️ Ba'zi bazalarda `refresh_from_db()` kerak (PostgreSQL da `RETURNING` orqali avtomatik keladi) |
| Ifoda (expression) qabul qiladimi | Faqat Python callable | `Now()`, `Lower(...)` kabi DB funksiyalar |

Ikkalasini birga bersangiz, Python da `default` ishlatiladi, `db_default` esa bazada zaxira bo'lib turadi.

---

## 2.4. `choices` — tanlov ro'yxati

### Oddiy usul (eski)

```python
class Order(models.Model):
    STATUS_CHOICES = [
        ("new", "Yangi"),          # (bazadagi qiymat, foydalanuvchiga ko'rinadigan nom)
        ("paid", "To'langan"),
        ("shipped", "Jo'natilgan"),
    ]
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default="new")
```

### Zamonaviy usul — `TextChoices` / `IntegerChoices` (tavsiya etiladi)

```python
from django.db import models


class Order(models.Model):
    class Status(models.TextChoices):
        NEW = "new", "Yangi"
        PAID = "paid", "To'langan"
        SHIPPED = "shipped", "Jo'natilgan"

    class Priority(models.IntegerChoices):
        LOW = 1, "Past"
        MEDIUM = 2, "O'rta"
        HIGH = 3, "Yuqori"

    status = models.CharField(
        max_length=20,
        choices=Status,            # Django 5.0+: .choices yozish shart emas
        default=Status.NEW,
    )
    priority = models.PositiveSmallIntegerField(choices=Priority, default=Priority.MEDIUM)
```

> Django 5.0 dan oldin `choices=Status.choices` deb yozilardi. Hozir ham ishlaydi.

Ko'p tilli loyihada label larni tarjima qilish uchun `gettext_lazy` ishlating:

```python
from django.utils.translation import gettext_lazy as _

class Status(models.TextChoices):
    NEW = "new", _("Yangi")
```

Label berilmasa, Django uni nomdan yasaydi: `IN_PROGRESS = "in_progress"` → label `"In Progress"`.

### Choices bilan ishlash

```python
order = Order.objects.create()
order.status                       # 'new'
order.get_status_display()         # 'Yangi'  ← Django avtomatik yaratadigan metod: get_<field>_display()
order.status == Order.Status.NEW   # True

Order.objects.filter(status=Order.Status.PAID)   # magic string o'rniga konstantadan foydalaning

Order.Status.choices   # [('new', 'Yangi'), ('paid', "To'langan"), ...]
Order.Status.labels    # ['Yangi', "To'langan", ...]
Order.Status.values    # ['new', 'paid', ...]
Order.Status.NEW.label # 'Yangi'
Order.Status("paid")   # Order.Status.PAID
```

### Guruhlangan choices

```python
MEDIA_CHOICES = {
    "Audio": {"vinyl": "Vinyl", "cd": "CD"},
    "Video": {"vhs": "VHS", "dvd": "DVD"},
    "unknown": "Noma'lum",
}
media = models.CharField(max_length=10, choices=MEDIA_CHOICES)   # Django 5.0+: dict ham qabul qilinadi
```

### ⚠️ Muhim: `choices` bazada cheklov yaratMAYDI

```python
Order.objects.create(status="nonsense")   # Hech qanday xato bermaydi!
order.full_clean()                         # Faqat shu yerda ValidationError
```

Bazada ham himoya kerak bo'lsa → `CheckConstraint` (5-darsga qarang):

```python
class Meta:
    constraints = [
        models.CheckConstraint(
            condition=models.Q(status__in=["new", "paid", "shipped"]),
            name="order_status_valid",
        )
    ]
```

---

## 2.5. `unique`

```python
email = models.EmailField(unique=True)
```

- Bazada `UNIQUE` constraint yaratadi (va avtomatik index ham).
- `unique=True` bo'lsa `db_index=True` yozish **ortiqcha**.
- `ManyToManyField` va `OneToOneField` da ishlatilmaydi (O2O allaqachon unique).

### `unique_for_date`, `unique_for_month`, `unique_for_year`

```python
class Article(models.Model):
    slug = models.SlugField(unique_for_date="published_at")
    published_at = models.DateTimeField()
```

Bir kunda bir xil `slug` bo'lmasligini tekshiradi. ⚠️ **Faqat validatsiya darajasida** (`full_clean()`, ModelForm) — bazada constraint yo'q.

---

## 2.6. `db_index`

```python
class Post(models.Model):
    status = models.CharField(max_length=20, db_index=True)
```

Ustun bo'yicha **B-tree index** yaratadi → `filter()`, `order_by()` tezlashadi, lekin INSERT/UPDATE biroz sekinlashadi va joy egallaydi.

**Qachon index qo'yish kerak:**
- Tez-tez `filter()`, `order_by()`, `JOIN` qilinadigan ustunlar.
- Yuqori selektivlikka ega ustunlar (ko'p xil qiymatli).

**Qachon kerak emas:**
- `ForeignKey` — Django avtomatik index qo'yadi.
- `unique=True` / `primary_key=True` — allaqachon index bor.
- Faqat 2–3 xil qiymatli boolean ustunlar (odatda samarasiz; buning uchun partial index yaxshiroq — 5-dars).

> Advanced: murakkab indexlar (bir nechta ustun, partial, GIN) uchun `Meta.indexes` ishlatiladi — 5-darsda.

---

## 2.7. `primary_key`

```python
import uuid

class Invoice(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
```

- `primary_key=True` → `null=False` va `unique=True` ni avtomatik bildiradi.
- Modelda faqat **bitta** primary key bo'lishi mumkin (ko'p ustunli PK uchun Django 5.2+ da `CompositePrimaryKey` bor — 3-dars).
- ⚠️ Mavjud obyektning PK sini o'zgartirib `save()` qilsangiz, Django eskisini o'zgartirmaydi — **yangi qator** yaratadi!

---

## 2.8. `editable`

```python
slug = models.SlugField(editable=False)
```

`editable=False` → field admin va ModelForm da ko'rinmaydi, model validatsiyasida o'tkazib yuboriladi. Avtomatik hisoblanadigan qiymatlar uchun (slug, hash, token). `auto_now`/`auto_now_add` bo'lgan fieldlar avtomatik `editable=False` bo'ladi.

---

## 2.9. `verbose_name` va `help_text`

```python
class Product(models.Model):
    name = models.CharField("Mahsulot nomi", max_length=200)          # 1-positional argument = verbose_name
    sku = models.CharField(
        verbose_name="Artikul",
        max_length=50,
        help_text="Ombordagi unikal kod, masalan: ABC-001",
    )
    category = models.ForeignKey(
        "Category",
        on_delete=models.PROTECT,
        verbose_name="Kategoriya",   # FK, M2M, O2O da 1-argument model bo'lgani uchun verbose_name keyword bilan beriladi
    )
```

- `verbose_name` — admin va formalarda ko'rinadigan nom. Berilmasa, field nomidan yasaladi (`created_at` → "created at").
- `help_text` — forma maydoni ostida chiqadigan yordamchi matn. HTML escape qilinmaydi — foydalanuvchi kiritgan ma'lumotni bu yerga qo'ymang.

---

## 2.10. `validators`

```python
from django.core.validators import MinValueValidator, MaxValueValidator

rating = models.PositiveSmallIntegerField(
    validators=[MinValueValidator(1), MaxValueValidator(5)]
)
```

⚠️ Validatorlar `save()` da **avtomatik ishlamaydi**! Ular faqat:
- `model.full_clean()` chaqirilganda,
- `ModelForm` / admin / DRF serializer orqali saqlanganda ishlaydi.

Batafsil — 7-darsda.

---

## 2.11. `error_messages`

Standart xato xabarlarini almashtirish:

```python
email = models.EmailField(
    unique=True,
    error_messages={
        "unique": "Bu email bilan foydalanuvchi allaqachon mavjud.",
        "invalid": "Email noto'g'ri formatda.",
        "blank": "Email kiritish majburiy.",
    },
)
```

Kalitlar: `null`, `blank`, `invalid`, `invalid_choice`, `unique`, `unique_for_date`, `max_length` va boshqalar.

---

## 2.12. `db_column`, `db_comment`, `db_tablespace`

```python
class LegacyUser(models.Model):
    full_name = models.CharField(
        max_length=100,
        db_column="FULL_NAME",                         # bazadagi ustun nomi boshqacha bo'lsa (legacy DB)
        db_comment="Foydalanuvchining to'liq ismi",    # SQL COMMENT (Django 4.2+), DBA lar uchun foydali
    )
```

- `db_column` — mavjud (legacy) baza bilan ishlaganda yoki SQL kalit so'zi bilan to'qnashuvda.
- `db_comment` — bazadagi ustunga izoh. Model darajasida `Meta.db_table_comment` ham bor.
- `db_tablespace` — index uchun tablespace (PostgreSQL/Oracle, kam ishlatiladi).

---

## Xulosa jadvali

| Parametr | Default | Baza o'zgaradimi | Qisqacha |
|----------|---------|------------------|----------|
| `null` | False | ✅ | NULL ga ruxsat |
| `blank` | False | ❌ | Formada bo'sh qoldirish |
| `default` | — | ❌ | Python default qiymat |
| `db_default` | — | ✅ | SQL DEFAULT |
| `choices` | — | ❌ | Tanlov ro'yxati, `get_X_display()` |
| `unique` | False | ✅ | UNIQUE constraint |
| `db_index` | False | ✅ | Index |
| `primary_key` | False | ✅ | PK |
| `editable` | True | ❌ | Admin/formada ko'rinishi |
| `verbose_name` | — | ❌ | Inson o'qiydigan nom |
| `help_text` | "" | ❌ | Yordamchi matn |
| `validators` | [] | ❌ | Qo'shimcha tekshiruvlar |
| `error_messages` | — | ❌ | Xato matnlari |
| `db_column` | field nomi | ✅ | Ustun nomi |
| `db_comment` | — | ✅ | SQL izoh |
| `unique_for_date` | — | ❌ | Sana bo'yicha unikal |

---

## Topshiriqlar

1. `User` profili uchun model yozing: `phone` (ixtiyoriy, lekin kiritilsa unikal), `bio` (ixtiyoriy), `birth_date` (ixtiyoriy), `gender` (`TextChoices`). `null`/`blank` ni to'g'ri tanlang va har birini izohlang.
2. `default=timezone.now()` va `default=timezone.now` farqini shell da amalda ko'rsating (2 ta obyekt yaratib, vaqtlarini solishtiring; serverni qayta ishga tushirmasdan).
3. `Task` modeli uchun `status` (TextChoices) va `priority` (IntegerChoices) yozing. Shell da `get_status_display()` ni sinang. Keyin `Task.objects.create(status="xyz")` qiling — nima bo'ladi? `full_clean()` chi?
4. `db_default=Now()` bo'lgan field qo'shib `sqlmigrate` natijasini `default=timezone.now` bilan solishtiring.

⬅️ [01-model-asoslari.md](01-model-asoslari.md) | ➡️ [03-field-turlari.md](03-field-turlari.md)
