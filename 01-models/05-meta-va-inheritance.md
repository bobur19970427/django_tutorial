# 5. `Meta` class, constraint, index va model inheritance

## 5.1. `Meta` nima?

`Meta` — model ichidagi ichki class bo'lib, **field bo'lmagan** sozlamalarni saqlaydi: jadval nomi, tartib, indexlar, cheklovlar, ruxsatlar va h.k.

```python
class Post(models.Model):
    title = models.CharField(max_length=200)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = "posts"
        ordering = ["-created_at"]
        verbose_name = "Maqola"
        verbose_name_plural = "Maqolalar"
```

Django uslub qo'llanmasi bo'yicha model ichidagi tartib:

1. Fieldlar
2. Custom manager (`objects = ...`)
3. `class Meta`
4. `def __str__`
5. `def save`
6. `def get_absolute_url`
7. Boshqa metodlar

---

## 5.2. Asosiy `Meta` parametrlari

### `db_table`

```python
db_table = "blog_articles"     # default: <app_label>_<modelname> → blog_post
```

### `ordering`

```python
ordering = ["-created_at"]              # yangilari birinchi
ordering = ["category", "-price"]       # avval kategoriya, keyin narx kamayish tartibida
ordering = [models.F("published_at").desc(nulls_last=True)]   # NULL lar oxirida
```

⚠️ **Performance tuzog'i:** `ordering` ro'yxatni qaytaradigan **har bir** so'rovga `ORDER BY` qo'shadi — hatto sizga tartib kerak bo'lmasa ham (masalan, `for obj in qs:` bilan hammasini qayta ishlaganda yoki `.exists()`/`.count()` bo'lmagan oddiy `filter()` larda). Katta jadvallarda index bo'lmasa bu sekin bo'ladi. Ko'p tajribali jamoalar `Meta.ordering` ni qo'ymaydi va kerakli joyda `.order_by()` yozadi. Kerak bo'lmasa tartibni o'chirish: `Post.objects.order_by()`.

### `verbose_name`, `verbose_name_plural`

Admin panelda ko'rinadigan nomlar. `verbose_name_plural` berilmasa → `verbose_name + "s"` ("Maqolas" — xunuk).

### `get_latest_by`

```python
get_latest_by = "created_at"

Post.objects.latest()     # eng oxirgi
Post.objects.earliest()   # eng birinchi
```

### `default_related_name`

```python
class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
    class Meta:
        default_related_name = "comments"     # barcha FK lar uchun default related_name
```

### `permissions` va `default_permissions`

```python
class Report(models.Model):
    class Meta:
        permissions = [
            ("export_report", "Hisobotni eksport qila oladi"),
            ("approve_report", "Hisobotni tasdiqlay oladi"),
        ]
        default_permissions = ("view", "change")   # default: ("add", "change", "delete", "view")
```

```python
user.has_perm("reports.export_report")
```

### `managed`

```python
class LegacyCustomer(models.Model):
    class Meta:
        managed = False           # Django bu jadval uchun migratsiya YARATMAYDI va o'chirmaydi
        db_table = "crm_customers"
```

Qachon: mavjud (legacy) baza, **database VIEW** ustidagi model, boshqa servis boshqaradigan jadval.

> Mavjud bazadan model yaratish: `python manage.py inspectdb > models.py`

### `db_table_comment`

```python
db_table_comment = "Foydalanuvchi buyurtmalari"   # SQL COMMENT ON TABLE
```

### `unique_together` va `index_together` (eski)

```python
unique_together = [["student", "course"]]   # ⚠️ eskirgan uslub — UniqueConstraint ishlating
# index_together Django 5.1 da butunlay olib tashlangan
```

---

## 5.3. `constraints` — baza darajasidagi cheklovlar

Python validatsiyasi (`validators`, `choices`) faqat forma/`full_clean()` orqali ishlaydi. **Constraint** esa bazaning o'zida — hech kim (raw SQL, boshqa servis, `update()`, bulk_create) uni chetlab o'tolmaydi.

### `UniqueConstraint`

```python
from django.db.models import Q, UniqueConstraint
from django.db.models.functions import Lower


class Enrollment(models.Model):
    student = models.ForeignKey("Student", on_delete=models.CASCADE)
    course = models.ForeignKey("Course", on_delete=models.CASCADE)
    is_active = models.BooleanField(default=True)

    class Meta:
        constraints = [
            # 1. Oddiy: bir talaba bir kursga bir marta
            UniqueConstraint(fields=["student", "course"], name="uniq_student_course"),
        ]


class User(models.Model):
    email = models.EmailField()
    deleted_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        constraints = [
            # 2. Funksional (expression): katta-kichik harfni farqlamasdan unikal email
            UniqueConstraint(Lower("email"), name="uniq_user_email_ci"),

            # 3. Shartli (partial): faqat o'chirilmagan userlar orasida unikal (soft delete bilan)
            UniqueConstraint(
                fields=["email"],
                condition=Q(deleted_at__isnull=True),
                name="uniq_active_user_email",
            ),
        ]
```

Qo'shimcha parametrlar:
- `violation_error_message` / `violation_error_code` — `full_clean()` da chiqadigan xato matni.
- `nulls_distinct=False` (Django 5.0+, PostgreSQL 15+) — NULL larni ham teng deb hisoblash.
- `deferrable=Deferrable.DEFERRED` — tekshiruvni tranzaksiya oxiriga qoldirish (masalan, tartib raqamlarini almashtirishda).
- `include=[...]` — covering index (PostgreSQL).

### `CheckConstraint`

```python
from django.db.models import CheckConstraint, F, Q


class Product(models.Model):
    price = models.DecimalField(max_digits=10, decimal_places=2)
    discount_price = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)
    discount_percent = models.PositiveSmallIntegerField(default=0)

    class Meta:
        constraints = [
            CheckConstraint(condition=Q(price__gt=0), name="product_price_positive"),
            CheckConstraint(condition=Q(discount_percent__lte=100), name="product_discount_max_100"),
            CheckConstraint(
                condition=Q(discount_price__isnull=True) | Q(discount_price__lt=F("price")),
                name="product_discount_lt_price",
            ),
        ]


class Booking(models.Model):
    start = models.DateTimeField()
    end = models.DateTimeField()

    class Meta:
        constraints = [
            CheckConstraint(condition=Q(end__gt=F("start")), name="booking_end_after_start"),
        ]
```

> Django 5.1 dan boshlab parametr nomi `condition=`. Eski kodda `check=` ko'rasiz — u eskirgan (deprecated).

Buzilganda:
```python
Product.objects.create(price=-5)   # django.db.IntegrityError
p = Product(price=-5); p.full_clean()   # ValidationError (Django 4.1+ constraint larni validatsiya qiladi)
```

### `ExclusionConstraint` (PostgreSQL)

"Bitta xona bir vaqtda ikki marta band qilinmasin":

```python
from django.contrib.postgres.constraints import ExclusionConstraint
from django.contrib.postgres.fields import DateTimeRangeField, RangeOperators


class RoomBooking(models.Model):
    room = models.ForeignKey("Room", on_delete=models.CASCADE)
    period = DateTimeRangeField()

    class Meta:
        constraints = [
            ExclusionConstraint(
                name="no_overlapping_bookings",
                expressions=[
                    ("room", RangeOperators.EQUAL),
                    ("period", RangeOperators.OVERLAPS),
                ],
            ),
        ]
```

(`btree_gist` extension kerak: migratsiyada `BtreeGistExtension()`.)

---

## 5.4. `indexes` — murakkab indexlar

```python
from django.db.models import Index, Q
from django.db.models.functions import Lower
from django.contrib.postgres.indexes import GinIndex


class Post(models.Model):
    title = models.CharField(max_length=200)
    status = models.CharField(max_length=20)
    author = models.ForeignKey("auth.User", on_delete=models.CASCADE)
    created_at = models.DateTimeField()
    metadata = models.JSONField(default=dict)

    class Meta:
        indexes = [
            # 1. Ko'p ustunli (composite) — WHERE author_id=? ORDER BY created_at DESC uchun
            Index(fields=["author", "-created_at"], name="post_author_created_idx"),

            # 2. Partial index — faqat published postlar indekslanadi (kichik va tez)
            Index(
                fields=["-created_at"],
                condition=Q(status="published"),
                name="post_published_idx",
            ),

            # 3. Funksional index — filter(title__iexact=...) / Lower("title") uchun
            Index(Lower("title"), name="post_title_lower_idx"),

            # 4. GIN — JSONField / ArrayField / full-text search uchun (PostgreSQL)
            GinIndex(fields=["metadata"], name="post_metadata_gin"),
        ]
```

### Composite index qoidasi (muhim!)

`Index(fields=["author", "created_at"])` quyidagilarda ishlaydi:
- ✅ `filter(author=...)`
- ✅ `filter(author=...).order_by("created_at")`
- ❌ `filter(created_at__gte=...)` — birinchi ustunsiz index ishlatilmaydi (chap prefiks qoidasi).

So'rov index ishlatayotganini tekshirish:

```python
print(Post.objects.filter(author_id=1).order_by("-created_at").explain(analyze=True))
```

> Index `name` maksimal 30 belgi va harf bilan boshlanishi kerak.

---

## 5.5. Model inheritance — meros olish

Django da 3 xil meros bor:

| Tur | Bazada | Qachon |
|-----|--------|--------|
| **Abstract base class** | Ota uchun jadval **yo'q**, fieldlar har bir bolaga ko'chiriladi | Umumiy fieldlarni qayta ishlatish (eng ko'p ishlatiladi) |
| **Multi-table inheritance** | Ota va har bir bola uchun **alohida jadval**, `OneToOne` bilan bog'langan | Ota model mustaqil ham ishlatilsa |
| **Proxy model** | Yangi jadval **yo'q**, ota jadvali ishlatiladi | Faqat Python xatti-harakatini o'zgartirish (manager, metod, ordering) |

### 5.5.1. Abstract base class

```python
import uuid
from django.db import models


class TimeStampedModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True


class UUIDModel(models.Model):
    public_id = models.UUIDField(default=uuid.uuid4, unique=True, editable=False)

    class Meta:
        abstract = True


class SoftDeleteModel(models.Model):
    deleted_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        abstract = True

    def soft_delete(self):
        from django.utils import timezone
        self.deleted_at = timezone.now()
        self.save(update_fields=["deleted_at"])


class Post(TimeStampedModel, UUIDModel, SoftDeleteModel):   # bir nechta abstract dan meros
    title = models.CharField(max_length=200)

    class Meta(TimeStampedModel.Meta):    # ota Meta dan meros (abstract=False avtomatik bo'ladi)
        ordering = ["-created_at"]
```

`blog_post` jadvalida: `id, created_at, updated_at, public_id, deleted_at, title` — barchasi bitta jadvalda. **JOIN yo'q** — tez.

> **Best practice:** har bir loyihada `core/models.py` da `TimeStampedModel` kabi abstract modellar yozing.

### 5.5.2. Multi-table inheritance

```python
class Place(models.Model):
    name = models.CharField(max_length=100)
    address = models.CharField(max_length=255)


class Restaurant(Place):
    serves_pizza = models.BooleanField(default=False)
```

Bazada:
- `app_place`: `id, name, address`
- `app_restaurant`: `place_ptr_id` (PK va `Place` ga OneToOne), `serves_pizza`

```python
r = Restaurant.objects.create(name="Pizza Hub", address="Toshkent", serves_pizza=True)
# 2 ta INSERT bajariladi!

Place.objects.all()               # restoranlar ham shu yerda (Place sifatida)
place = Place.objects.get(name="Pizza Hub")
place.restaurant                  # Restaurant obyekti (bo'lmasa — DoesNotExist)
Restaurant.objects.filter(name="Pizza Hub")   # avtomatik JOIN
```

⚠️ **Kamchiliklari:** har bir so'rovda JOIN, yaratishda bir nechta INSERT, chuqur meros — juda sekin. Ko'p hollarda abstract model yoki aniq `OneToOneField` (kompozitsiya) yaxshiroq.

Ota-bola bog'lanishini o'zingiz nomlash:

```python
class Restaurant(Place):
    place = models.OneToOneField(Place, on_delete=models.CASCADE, parent_link=True, primary_key=True)
```

### 5.5.3. Proxy model

```python
class Order(models.Model):
    class Status(models.TextChoices):
        NEW = "new"
        PAID = "paid"
    status = models.CharField(max_length=20, choices=Status, default=Status.NEW)
    total = models.DecimalField(max_digits=12, decimal_places=2)


class PaidOrderManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(status=Order.Status.PAID)


class PaidOrder(Order):
    objects = PaidOrderManager()

    class Meta:
        proxy = True
        ordering = ["-total"]
        verbose_name = "To'langan buyurtma"

    def send_receipt(self):
        ...
```

```python
PaidOrder.objects.all()     # faqat to'langanlar, o'sha `order` jadvalidan
```

**Qachon:** admin da bitta jadvalni turli ko'rinishda alohida bo'lim qilib ko'rsatish, turli manager/metodlar. Proxy da yangi field qo'shib **bo'lmaydi**.

### Taqqoslash

| | Abstract | Multi-table | Proxy |
|--|----------|-------------|-------|
| Ota jadvali | ❌ | ✅ | ✅ (o'sha jadval) |
| Bola jadvali | ✅ | ✅ | ❌ |
| Yangi field qo'shish | ✅ | ✅ | ❌ |
| Ota orqali so'rov (`Parent.objects`) | ❌ | ✅ | ✅ |
| JOIN | ❌ | ✅ | ❌ |
| Ishlatilishi | Juda ko'p | Kam | O'rtacha |

---

## Topshiriqlar

1. `core` app yarating va `TimeStampedModel`, `UUIDModel`, `SoftDeleteModel` abstract modellarini yozing. Ularni 2 ta modelga qo'llang va `sqlmigrate` bilan jadvallarni tekshiring.
2. `Product` uchun constraint lar yozing: narx > 0; chegirma narxi asosiy narxdan kichik; `sku` katta-kichik harfga qaramay unikal. Har birini shell da buzishga harakat qilib, qanday xato chiqishini ko'ring.
3. Soft delete bilan ishlaydigan `User` uchun "faqat faol foydalanuvchilar orasida unikal username" ni partial `UniqueConstraint` bilan qiling.
4. `Post` uchun `filter(status="published").order_by("-created_at")` so'rovini tezlashtiradigan partial index yozing va `.explain()` bilan oldin/keyin solishtiring (PostgreSQL da, ~100 000 qator bilan).
5. `Place → Restaurant` multi-table inheritance yozing, `Restaurant.objects.create()` qanday SQL bajarishini `connection.queries` orqali ko'ring.
6. `Order` uchun `PaidOrder` va `NewOrder` proxy modellarini yozib, admin da alohida bo'lim qilib ro'yxatdan o'tkazing.

⬅️ [04-relationship-fieldlar.md](04-relationship-fieldlar.md) | ➡️ [06-model-metodlari.md](06-model-metodlari.md)
