# 6. Model metodlari

Model — bu faqat ma'lumot emas, balki **biznes logika** joylashadigan joy ham ("Fat models, thin views" tamoyili). Bu darsda o'rnatilgan metodlarni qanday override qilish va o'z metodlaringizni yozishni ko'ramiz.

---

## 6.1. `__str__`

```python
def __str__(self):
    return f"{self.title} ({self.author})"   # ⚠️ self.author — qo'shimcha SQL so'rov!
```

Admin ro'yxatida har bir qator uchun `__str__` chaqiriladi. Unda FK ga murojaat qilsangiz — N+1 muammo. Imkon qadar faqat o'z fieldlaringizdan foydalaning.

---

## 6.2. `save()` ni override qilish

```python
from django.utils.text import slugify


class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True, blank=True)
    word_count = models.PositiveIntegerField(default=0, editable=False)
    body = models.TextField()

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.title)
        self.word_count = len(self.body.split())
        super().save(*args, **kwargs)      # ⚠️ ALBATTA chaqiring va *args, **kwargs ni uzating
```

### Qoidalar

1. **`super().save(*args, **kwargs)`** — unutsangiz, hech narsa saqlanmaydi.
2. **`*args, **kwargs`** ni uzating — `update_fields`, `force_insert`, `using` kabi argumentlar yo'qolmasin.
3. `update_fields` bilan ishlaganda o'zgartirgan fieldingizni qo'shing:

```python
def save(self, *args, **kwargs):
    self.word_count = len(self.body.split())
    update_fields = kwargs.get("update_fields")
    if update_fields is not None and "body" in update_fields:
        kwargs["update_fields"] = {"word_count", *update_fields}
    super().save(*args, **kwargs)
```

### Yangi obyektmi yoki mavjudmi?

```python
def save(self, *args, **kwargs):
    is_new = self._state.adding          # ✅ ishonchli usul
    # self.pk is None — UUID default bo'lsa ishlamaydi! (pk allaqachon bor)
    super().save(*args, **kwargs)
    if is_new:
        send_welcome_email(self)
```

### Field o'zgarganini aniqlash

```python
class Order(models.Model):
    status = models.CharField(max_length=20)

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._original_status = self.status

    def save(self, *args, **kwargs):
        if not self._state.adding and self.status != self._original_status:
            OrderStatusLog.objects.create(order=self, old=self._original_status, new=self.status)
        super().save(*args, **kwargs)
        self._original_status = self.status
```

> ⚠️ `.only()` / `.defer()` bilan yuklangan obyektda `self.status` ga murojaat qo'shimcha so'rov yuboradi. Keng tarqalgan yechim — `django-model-utils` dagi `FieldTracker`.

### ⚠️ `save()` ishlamaydigan holatlar

| Amal | `save()` chaqiriladimi |
|------|------------------------|
| `obj.save()` | ✅ |
| `Model.objects.create()` | ✅ |
| `QuerySet.update()` | ❌ |
| `bulk_create()` | ❌ |
| `bulk_update()` | ❌ |
| Raw SQL | ❌ |

Shuning uchun **muhim invariantlarni** faqat `save()` ga ishonib qo'ymang — constraint lar bilan bazada ham himoya qiling.

---

## 6.3. `delete()` ni override qilish

```python
class Document(models.Model):
    file = models.FileField(upload_to="docs/")

    def delete(self, *args, **kwargs):
        storage, name = self.file.storage, self.file.name
        result = super().delete(*args, **kwargs)
        storage.delete(name)    # baza o'chgandan keyin faylni o'chiramiz
        return result           # (o'chirilganlar soni, {model: soni}) qaytariladi
```

⚠️ `QuerySet.delete()` (`Document.objects.filter(...).delete()`) modelning `delete()` metodini **chaqirmaydi**. Ishonchli yechim — `post_delete` signal (u bulk delete da ham har bir obyekt uchun ishlaydi):

```python
from django.db.models.signals import post_delete
from django.dispatch import receiver


@receiver(post_delete, sender=Document)
def delete_file(sender, instance, **kwargs):
    if instance.file:
        instance.file.delete(save=False)
```

---

## 6.4. Validatsiya: `clean()`, `clean_fields()`, `validate_unique()`, `full_clean()`

```
full_clean()
 ├── clean_fields()        → har bir field: blank, max_length, choices, validators
 ├── clean()               → bir nechta fieldga bog'liq (cross-field) tekshiruv — SIZ yozasiz
 ├── validate_unique()     → unique, unique_together, unique_for_date
 └── validate_constraints() → Meta.constraints (Django 4.1+)
```

```python
from django.core.exceptions import ValidationError
from django.utils import timezone


class Event(models.Model):
    title = models.CharField(max_length=200)
    start = models.DateTimeField()
    end = models.DateTimeField()
    max_participants = models.PositiveIntegerField()
    is_online = models.BooleanField(default=False)
    location = models.CharField(max_length=255, blank=True)

    def clean(self):
        errors = {}
        if self.start and self.end and self.end <= self.start:
            errors["end"] = "Tugash vaqti boshlanishdan keyin bo'lishi kerak."
        if not self.is_online and not self.location:
            errors["location"] = "Offline tadbir uchun manzil majburiy."
        if self.start and self.start < timezone.now() and self._state.adding:
            errors["start"] = "O'tmishdagi tadbirni yaratib bo'lmaydi."
        if errors:
            raise ValidationError(errors)
```

### ⚠️ Eng muhim tushuncha: `save()` validatsiya qilMAYDI!

```python
e = Event(title="X", start=now, end=now - timedelta(hours=1))
e.save()           # ✅ saqlanadi! clean() chaqirilmaydi

e.full_clean()     # ❌ ValidationError
```

`full_clean()` avtomatik chaqiriladigan joylar: **ModelForm**, **Django Admin**. DRF serializer **chaqirmaydi** (u o'z validatsiyasiga ega).

Agar har doim validatsiya kerak bo'lsa:

```python
def save(self, *args, **kwargs):
    self.full_clean()
    super().save(*args, **kwargs)
```

Bu yondashuvning narxi: har `save()` da qo'shimcha so'rovlar (unique tekshiruvlar). Ko'p jamoalar validatsiyani **service layer** yoki forma/serializer da qiladi, `save()` da emas.

### `full_clean` parametrlari

```python
obj.full_clean(exclude=["slug"], validate_unique=False, validate_constraints=True)
```

---

## 6.5. `get_absolute_url()`

```python
from django.urls import reverse


class Post(models.Model):
    slug = models.SlugField(unique=True)

    def get_absolute_url(self):
        return reverse("blog:post-detail", kwargs={"slug": self.slug})
```

Foydalanish:
- Template da: `<a href="{{ post.get_absolute_url }}">`
- View da: `return redirect(post)` — Django avtomatik `get_absolute_url()` ni chaqiradi.
- Admin da "Saytda ko'rish" tugmasi paydo bo'ladi.
- `CreateView`/`UpdateView` — `success_url` berilmasa shu URL ga yo'naltiradi.

URL ni template larda qo'lda yig'ish o'rniga doim shu metoddan foydalaning (DRY).

---

## 6.6. `refresh_from_db()`

Obyektni bazadagi joriy holat bilan yangilaydi:

```python
from django.db.models import F

product.stock = F("stock") - 1    # race condition dan himoyalangan kamaytirish
product.save(update_fields=["stock"])
product.stock                     # <CombinedExpression: F(stock) - Value(1)>  — raqam emas!
product.refresh_from_db(fields=["stock"])
product.stock                     # 9
```

---

## 6.7. Custom metodlar va `@property`

```python
from decimal import Decimal
from django.utils import timezone
from django.utils.functional import cached_property


class Order(models.Model):
    class Status(models.TextChoices):
        NEW = "new", "Yangi"
        PAID = "paid", "To'langan"

    status = models.CharField(max_length=20, choices=Status, default=Status.NEW)
    created_at = models.DateTimeField(auto_now_add=True)
    paid_at = models.DateTimeField(null=True, blank=True)
    discount_percent = models.PositiveSmallIntegerField(default=0)
    # OrderItem.order = ForeignKey(Order, related_name="items") deb faraz qilamiz

    # --- Hisoblanadigan qiymatlar ---
    @property
    def age_days(self) -> int:
        return (timezone.now() - self.created_at).days

    @cached_property
    def subtotal(self) -> Decimal:
        # bitta obyekt uchun bir marta hisoblanadi va keshlanadi
        return sum((i.price * i.quantity for i in self.items.all()), Decimal("0"))

    @property
    def total(self) -> Decimal:
        return self.subtotal * (100 - self.discount_percent) / 100

    # --- Biznes amallar ("action" metodlar) ---
    def mark_as_paid(self):
        if self.status != self.Status.NEW:
            raise ValueError("Faqat yangi buyurtmani to'langan qilish mumkin")
        self.status = self.Status.PAID
        self.paid_at = timezone.now()
        self.save(update_fields=["status", "paid_at"])
```

| | `@property` | `@cached_property` | `GeneratedField` / `annotate` |
|--|-------------|--------------------|-------------------------------|
| Har murojaatda qayta hisoblanadi | ✅ | ❌ (bir marta) | — |
| `filter()` / `order_by()` da ishlatsa bo'ladimi | ❌ | ❌ | ✅ |
| Qayerda hisoblanadi | Python | Python | Baza |

⚠️ `property` ichida so'rov yuborish (`self.items.all()`) — ro'yxat sahifasida N+1 ga olib keladi. Ro'yxatlarda `annotate()` ishlating:

```python
from django.db.models import Sum, F
Order.objects.annotate(subtotal_db=Sum(F("items__price") * F("items__quantity")))
```

---

## 6.8. Avtomatik yaratiladigan metodlar

Django ba'zi fieldlar uchun metodlarni o'zi qo'shadi:

| Field | Metod | Misol |
|-------|-------|-------|
| `choices` bo'lgan field | `get_FOO_display()` | `order.get_status_display()` → `"To'langan"` |
| `DateField` / `DateTimeField` (null=False) | `get_next_by_FOO()`, `get_previous_by_FOO()` | `post.get_next_by_created_at()` |
| `ForeignKey` | `FOO_id` atributi | `post.author_id` |

```python
next_post = post.get_next_by_created_at(is_published=True)   # filter kwargs qabul qiladi
```

---

## 6.9. Model "lifecycle" — obyekt saqlanganda nima bo'ladi?

`obj.save()` chaqirilganda tartib:

1. `pre_save` signal yuboriladi
2. Fieldlar tayyorlanadi (`pre_save()` — masalan, `auto_now` shu yerda qiymat oladi)
3. Qiymatlar baza formatiga o'tkaziladi (`get_db_prep_save()`)
4. `INSERT` yoki `UPDATE` SQL bajariladi
5. `post_save` signal yuboriladi (`created=True/False`)

> Signallar — 5-modulda batafsil. Qisqa qoida: o'z modelingiz logikasini **`save()` da** yozing; signallarni **boshqa app** ning modeliga reaksiya qilish uchun ishlating (masalan, `User` yaratilganda `Profile` yaratish).

---

## Topshiriqlar

1. `Post.save()` ni override qiling: slug avtomatik va unikal bo'lsin (`-2`, `-3` qo'shib), `reading_time` (daqiqa, 200 so'z/daqiqa) hisoblansin. `update_fields` bilan to'g'ri ishlashini tekshiring.
2. `Event.clean()` yozing va `save()` bilan noto'g'ri ma'lumot saqlanishini, `full_clean()` bilan esa xato chiqishini ko'rsating.
3. `Order` da status o'zgarganda `OrderStatusLog` ga yozadigan mexanizm yarating. `QuerySet.update(status=...)` qilsangiz log yoziladimi? Nima uchun?
4. `Product` uchun `decrease_stock(qty)` metodini yozing: race condition dan himoyalangan bo'lsin (`F()` yoki `select_for_update()`), stock manfiy bo'lib qolmasin.
5. `@property total` va `annotate()` bilan hisoblangan total ni 100 ta buyurtmalik ro'yxatda solishtiring — so'rovlar sonini o'lchang.

⬅️ [05-meta-va-inheritance.md](05-meta-va-inheritance.md) | ➡️ [07-validator-va-custom-field.md](07-validator-va-custom-field.md)
