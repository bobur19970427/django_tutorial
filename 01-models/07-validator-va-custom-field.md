# 7. Validatorlar va Custom Field yozish

## 7.1. Validator nima?

Validator — qiymatni qabul qilib, noto'g'ri bo'lsa `ValidationError` ko'taradigan **callable**. Hech narsa qaytarishi shart emas.

```python
rating = models.PositiveSmallIntegerField(validators=[MinValueValidator(1), MaxValueValidator(5)])
```

Eslatma (6-darsdan): validatorlar `save()` da **ishlamaydi**, faqat `full_clean()`, ModelForm, Admin orqali.

---

## 7.2. O'rnatilgan validatorlar (`django.core.validators`)

| Validator | Vazifasi | Misol |
|-----------|----------|-------|
| `MinValueValidator(x)` | Qiymat ≥ x | `MinValueValidator(1)` |
| `MaxValueValidator(x)` | Qiymat ≤ x | `MaxValueValidator(100)` |
| `StepValueValidator(step, offset=None)` | Qiymat `step` ga karrali | `StepValueValidator(5)` — 0, 5, 10... |
| `MinLengthValidator(n)` | Uzunlik ≥ n | Parol, kod |
| `MaxLengthValidator(n)` | Uzunlik ≤ n | `CharField` avtomatik qo'shadi |
| `RegexValidator(regex, message, code, inverse_match, flags)` | Regex ga mos | Telefon, kod |
| `EmailValidator` | Email formati | `EmailField` avtomatik qo'shadi |
| `URLValidator(schemes=[...])` | URL formati | `URLField` avtomatik |
| `validate_slug` / `validate_unicode_slug` | Slug formati | `SlugField` avtomatik |
| `validate_ipv4_address`, `validate_ipv6_address`, `validate_ipv46_address` | IP | |
| `validate_comma_separated_integer_list` | `"1,2,3"` | |
| `DecimalValidator(max_digits, decimal_places)` | Decimal raqamlar soni | `DecimalField` avtomatik |
| `FileExtensionValidator(allowed_extensions)` | Fayl kengaytmasi | `["pdf", "docx"]` |
| `validate_image_file_extension` | Rasm kengaytmasi | `ImageField` avtomatik |
| `ProhibitNullCharactersValidator` | `\x00` belgisi yo'qligi | String fieldlar avtomatik |
| `DomainNameValidator` (5.1+) | Domen nomi | `"example.com"` |

### Misollar

```python
from django.core.validators import (
    MinValueValidator, MaxValueValidator, MinLengthValidator,
    RegexValidator, FileExtensionValidator,
)

phone_validator = RegexValidator(
    regex=r"^\+998\d{9}$",
    message="Telefon raqam +998XXXXXXXXX formatida bo'lishi kerak.",
    code="invalid_phone",
)


class Student(models.Model):
    phone = models.CharField(max_length=13, validators=[phone_validator])
    passport = models.CharField(
        max_length=9,
        validators=[RegexValidator(r"^[A-Z]{2}\d{7}$", "Pasport: AA1234567 formatida")],
    )
    gpa = models.DecimalField(
        max_digits=3, decimal_places=2,
        validators=[MinValueValidator(0), MaxValueValidator(4)],
    )
    username = models.CharField(
        max_length=30,
        validators=[
            MinLengthValidator(3),
            RegexValidator(r"^[a-z0-9_]+$", "Faqat kichik lotin harflari, raqam va _"),
            RegexValidator(r"^admin", "admin bilan boshlanishi mumkin emas", inverse_match=True),
        ],
    )
    resume = models.FileField(
        upload_to="resumes/",
        validators=[FileExtensionValidator(["pdf"])],
    )
```

> ⚠️ `FileExtensionValidator` faqat **nom**ni tekshiradi. `virus.exe` ni `virus.pdf` deb nomlasa o'tib ketadi. Xavfsizlik muhim bo'lsa, fayl mazmunini (MIME, `python-magic`) ham tekshiring.

---

## 7.3. O'z validatoringizni yozish

### Funksiya ko'rinishida

```python
from django.core.exceptions import ValidationError
from django.utils import timezone


def validate_not_future(value):
    if value > timezone.localdate():
        raise ValidationError(
            "%(value)s — kelajakdagi sana bo'lishi mumkin emas.",
            code="future_date",
            params={"value": value},
        )


def validate_adult(birth_date):
    today = timezone.localdate()
    age = today.year - birth_date.year - ((today.month, today.day) < (birth_date.month, birth_date.day))
    if age < 18:
        raise ValidationError("Foydalanuvchi kamida 18 yoshda bo'lishi kerak.", code="underage")


class Person(models.Model):
    birth_date = models.DateField(validators=[validate_not_future, validate_adult])
```

Yaxshi amaliyot: `code` va `params` bering — tarjima va frontend da xatoni aniqlash osonlashadi.

### Class ko'rinishida (parametrli validator)

Parametr qabul qiladigan validator **migratsiyaga yozilishi** uchun `@deconstructible` va `__eq__` kerak:

```python
from django.utils.deconstruct import deconstructible
from django.core.exceptions import ValidationError


@deconstructible
class FileSizeValidator:
    message = "Fayl hajmi %(max_mb)s MB dan oshmasligi kerak (hozir: %(size_mb).1f MB)."
    code = "file_too_large"

    def __init__(self, max_mb):
        self.max_mb = max_mb

    def __call__(self, file):
        size_mb = file.size / (1024 * 1024)
        if size_mb > self.max_mb:
            raise ValidationError(
                self.message, code=self.code,
                params={"max_mb": self.max_mb, "size_mb": size_mb},
            )

    def __eq__(self, other):
        return isinstance(other, FileSizeValidator) and self.max_mb == other.max_mb


class Document(models.Model):
    file = models.FileField(upload_to="docs/", validators=[FileSizeValidator(5)])
```

`__eq__` bo'lmasa, har `makemigrations` da Django "o'zgarish bor" deb yangi migratsiya yaratadi.

> ⚠️ **Lambda ishlatmang:** `validators=[lambda v: ...]` migratsiyaga serialize bo'lmaydi → `ValueError: Cannot serialize function: lambda`.

---

## 7.4. Validatsiya qayerda bo'lishi kerak? (Advanced)

| Daraja | Vosita | Kafolat | Qachon |
|--------|--------|---------|--------|
| **Baza** | `null`, `unique`, `CheckConstraint`, `UniqueConstraint`, FK | 💯 Hech kim chetlab o'tolmaydi | Ma'lumot yaxlitligi uchun muhim qoidalar |
| **Model** | `validators`, `clean()` | ModelForm/Admin/`full_clean()` dagina | Foydalanuvchiga chiroyli xato kerak bo'lganda |
| **Forma / Serializer** | `clean_<field>()`, `validate_<field>()` | Faqat shu kirish nuqtasida | Kontekstga bog'liq tekshiruv (masalan, `request.user`) |
| **Service layer** | Oddiy Python | Siz chaqirgan joyda | Murakkab biznes qoidalar, bir nechta model |

**Eng yaxshi yondashuv — "chuqur himoya":** muhim qoida (narx > 0) **bazada constraint** sifatida + foydalanuvchiga tushunarli xato uchun **validator** sifatida.

```python
price = models.DecimalField(
    max_digits=10, decimal_places=2,
    validators=[MinValueValidator(Decimal("0.01"))],    # forma uchun chiroyli xato
)

class Meta:
    constraints = [
        models.CheckConstraint(condition=models.Q(price__gt=0), name="price_gt_0"),   # bazada kafolat
    ]
```

---

## 7.5. Custom Model Field yozish

Qachon kerak: biror turdagi qiymatni bazada maxsus formatda saqlash va Python da qulay obyekt sifatida olish (masalan, telefon, pul, shifrlangan matn, rang).

### Field qanday ishlaydi — asosiy metodlar

```
Python obyekt  ──get_prep_value()──►  baza qiymati  ──► SQL
     ▲                                                    │
     └────────from_db_value()──────── baza qiymati ◄──────┘

Forma / serializer qiymati ──to_python()──► Python obyekt
```

| Metod | Qachon chaqiriladi | Vazifasi |
|-------|--------------------|----------|
| `__init__` | Field e'lon qilinganda | Parametrlarni qabul qilish |
| `deconstruct()` | `makemigrations` | Fieldni migratsiyaga yozish uchun argumentlarni qaytarish |
| `db_type(connection)` / `get_internal_type()` | Migratsiya | SQL turini aniqlash |
| `from_db_value(value, expression, connection)` | Bazadan o'qilganda | baza → Python |
| `to_python(value)` | Validatsiya / deserializatsiya | istalgan → Python |
| `get_prep_value(value)` | Saqlash va filter da | Python → baza |
| `formfield(**kwargs)` | ModelForm | Qaysi forma maydoni ishlatilishini aniqlash |
| `value_to_string(obj)` | Serializatsiya (`dumpdata`) | Python → string |

### Misol 1: `LowercaseEmailField` — oddiy kengaytirish

```python
class LowercaseEmailField(models.EmailField):
    """Emailni doim kichik harfda saqlaydi."""

    def get_prep_value(self, value):
        value = super().get_prep_value(value)
        return value.lower().strip() if isinstance(value, str) else value

    def to_python(self, value):
        value = super().to_python(value)
        return value.lower().strip() if isinstance(value, str) else value
```

```python
class User(models.Model):
    email = LowercaseEmailField(unique=True)

User.objects.create(email="  Ali@Mail.COM ")
User.objects.get(email="ALI@mail.com")    # ✅ topadi — filter da ham get_prep_value ishlaydi
```

### Misol 2: `MoneyField` — Python obyekti bilan ishlash

```python
from dataclasses import dataclass
from django.db import models


@dataclass(frozen=True)
class Money:
    amount: int          # tiyinlarda
    currency: str = "UZS"

    def __str__(self):
        return f"{self.amount / 100:,.2f} {self.currency}"


class MoneyField(models.Field):
    """Pulni '12500:UZS' ko'rinishida varchar sifatida saqlaydi."""

    description = "Pul miqdori va valyuta"

    def __init__(self, *args, default_currency="UZS", **kwargs):
        self.default_currency = default_currency
        kwargs.setdefault("max_length", 40)
        super().__init__(*args, **kwargs)

    def deconstruct(self):
        name, path, args, kwargs = super().deconstruct()
        if self.default_currency != "UZS":
            kwargs["default_currency"] = self.default_currency
        if kwargs.get("max_length") == 40:
            del kwargs["max_length"]
        return name, path, args, kwargs

    def get_internal_type(self):
        return "CharField"            # bazada varchar(max_length) sifatida

    def from_db_value(self, value, expression, connection):
        if value is None:
            return None
        return self._parse(value)

    def to_python(self, value):
        if value is None or isinstance(value, Money):
            return value
        return self._parse(value)

    def get_prep_value(self, value):
        if value is None:
            return None
        if isinstance(value, Money):
            return f"{value.amount}:{value.currency}"
        return f"{int(value)}:{self.default_currency}"   # oddiy raqam berilsa

    def value_to_string(self, obj):
        return self.get_prep_value(self.value_from_object(obj))

    def _parse(self, value):
        amount, _, currency = str(value).partition(":")
        return Money(int(amount), currency or self.default_currency)
```

```python
class Invoice(models.Model):
    total = MoneyField()

inv = Invoice.objects.create(total=Money(1_250_000, "UZS"))
inv.refresh_from_db()
inv.total            # Money(amount=1250000, currency='UZS')
str(inv.total)       # '12,500.00 UZS'
```

> Bu ta'limiy misol. Real loyihada pul uchun `django-money` paketi yoki ikkita ustun (`DecimalField` + `CharField` valyuta) ishlatiladi — chunki string ustunda `SUM`, `ORDER BY` ishlamaydi. Custom field yozishdan oldin **har doim** shu savolni bering: "Bazada bu qiymat bo'yicha hisob/saralash kerak bo'ladimi?"

### `deconstruct()` nima uchun kerak?

`makemigrations` fieldni migratsiya fayliga Python kodi sifatida yozadi:

```python
("total", myapp.fields.MoneyField(default_currency="USD")),
```

Buning uchun Django fieldni qanday qayta yaratishni bilishi kerak — `deconstruct()` shuni qaytaradi: `(name, "import.path.ClassName", args, kwargs)`. `__init__` ga yangi parametr qo'shsangiz, uni `deconstruct()` ga ham qo'shishingiz **shart**, aks holda migratsiyada yo'qoladi.

### Misol 3: Custom lookup (bonus)

```python
from django.db.models import Lookup


@models.CharField.register_lookup
class NotEqual(Lookup):
    lookup_name = "ne"

    def as_sql(self, compiler, connection):
        lhs, lhs_params = self.process_lhs(compiler, connection)
        rhs, rhs_params = self.process_rhs(compiler, connection)
        return f"{lhs} <> {rhs}", [*lhs_params, *rhs_params]


Post.objects.filter(status__ne="draft")
```

---

## Topshiriqlar

1. O'zbekiston telefon raqami uchun `UzPhoneValidator` (class, `@deconstructible`) yozing: `+998` + operator kodi (90, 91, 93, 94, 95, 97, 98, 99, 33, 88, 77 ...) + 7 raqam.
2. `FileSizeValidator` va `FileExtensionValidator` bilan avatar fieldini himoyalang (max 2 MB, faqat jpg/png/webp).
3. `PhoneNumberField` custom field yozing: kiritilgan `"90 123-45-67"`, `"+998901234567"`, `"998 90 123 45 67"` ni bir xil formatga (`+998901234567`) normalizatsiya qilib saqlasin, filter da ham ishlasin.
4. `__eq__` siz `@deconstructible` validator yozing va `makemigrations` ni ikki marta ketma-ket ishga tushiring. Nima bo'ladi?
5. Validatsiya darajalarini taqqoslang: `Product.price > 0` qoidasini (a) faqat validator, (b) faqat constraint, (c) ikkalasi bilan yozing va `Product.objects.create(price=-1)`, `full_clean()`, `bulk_create()` holatlarida natijani jadvalga yozing.

⬅️ [06-model-metodlari.md](06-model-metodlari.md) | ➡️ [08-amaliyot.md](08-amaliyot.md)
