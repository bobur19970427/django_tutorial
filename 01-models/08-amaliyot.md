# 8. Amaliyot: modul yakuniy loyihasi

Bu darsda o'rganilgan barcha mavzularni bitta real loyihada birlashtiramiz: **Online kurs platformasi (LMS)**.

---

## 8.1. Talablar

- Foydalanuvchilar: talaba yoki o'qituvchi bo'lishi mumkin, har birining profili bor.
- Kurslar kategoriyalarga bo'linadi (kategoriyalar ichma-ich bo'lishi mumkin).
- Kursning bir yoki bir nechta o'qituvchisi, narxi, darajasi, holati (draft/published/archived) bor.
- Kurs → Modullar → Darslar. Darslar tartiblangan, video yoki matn bo'lishi mumkin.
- Talaba kursga yoziladi (`Enrollment`): yozilgan sana, to'langan summa, progress.
- Talaba darsni tugatganini belgilaydi (`LessonProgress`).
- Talaba kursga 1–5 baho va izoh qoldiradi — bitta kursga bitta marta.

---

## 8.2. ER-diagramma

```
User 1───1 Profile
 │
 ├──< Enrollment >──┐
 │                  │
 ├──< Review >──────┤
 │                  │
 └──<<(teachers)>>──Course >───── Category ──┐
                    │                 ▲      │ parent (self)
                    │                 └──────┘
                    └──< Module ──< Lesson ──< LessonProgress >── User

──<   : ForeignKey (ko'pdan-birga)
<<>>  : ManyToMany
1───1 : OneToOne
```

---

## 8.3. To'liq kod

### `core/models.py` — abstract bazalar

```python
import uuid

from django.db import models


class TimeStampedModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True


class PublicIdModel(models.Model):
    public_id = models.UUIDField(default=uuid.uuid4, unique=True, editable=False)

    class Meta:
        abstract = True
```

### `accounts/models.py`

```python
from django.conf import settings
from django.contrib.auth.models import AbstractUser
from django.core.validators import RegexValidator
from django.db import models

from core.models import TimeStampedModel


class User(AbstractUser):
    class Role(models.TextChoices):
        STUDENT = "student", "Talaba"
        TEACHER = "teacher", "O'qituvchi"

    email = models.EmailField(unique=True)
    role = models.CharField(max_length=10, choices=Role, default=Role.STUDENT)


phone_validator = RegexValidator(r"^\+998\d{9}$", "Format: +998XXXXXXXXX")


class Profile(TimeStampedModel):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="profile"
    )
    phone = models.CharField(
        max_length=13, unique=True, null=True, blank=True, validators=[phone_validator]
    )
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to="avatars/%Y/%m/", blank=True)
    birth_date = models.DateField(null=True, blank=True)

    def __str__(self):
        return f"Profile #{self.user_id}"
```

`settings.py`:

```python
AUTH_USER_MODEL = "accounts.User"   # ⚠️ Loyiha boshida, BIRINCHI migrate dan oldin qo'ying!
```

### `courses/models.py`

```python
from decimal import Decimal

from django.conf import settings
from django.core.validators import MaxValueValidator, MinValueValidator
from django.db import models
from django.db.models import F, Q
from django.urls import reverse
from django.utils.text import slugify

from core.models import PublicIdModel, TimeStampedModel


class Category(models.Model):
    name = models.CharField(max_length=100)
    slug = models.SlugField(max_length=120, unique=True)
    parent = models.ForeignKey(
        "self", on_delete=models.PROTECT, null=True, blank=True, related_name="children"
    )

    class Meta:
        verbose_name = "Kategoriya"
        verbose_name_plural = "Kategoriyalar"

    def __str__(self):
        return self.name


class Course(TimeStampedModel, PublicIdModel):
    class Level(models.IntegerChoices):
        BEGINNER = 1, "Boshlang'ich"
        INTERMEDIATE = 2, "O'rta"
        ADVANCED = 3, "Yuqori"

    class Status(models.TextChoices):
        DRAFT = "draft", "Qoralama"
        PUBLISHED = "published", "Nashr qilingan"
        ARCHIVED = "archived", "Arxivlangan"

    title = models.CharField("Nomi", max_length=200)
    slug = models.SlugField(max_length=220, unique=True, blank=True)
    description = models.TextField(blank=True)
    category = models.ForeignKey(Category, on_delete=models.PROTECT, related_name="courses")
    teachers = models.ManyToManyField(
        settings.AUTH_USER_MODEL,
        related_name="taught_courses",
        limit_choices_to={"role": "teacher"},
    )
    students = models.ManyToManyField(
        settings.AUTH_USER_MODEL,
        through="Enrollment",
        related_name="enrolled_courses",
        blank=True,
    )
    price = models.DecimalField(
        max_digits=12, decimal_places=2, default=Decimal("0"),
        validators=[MinValueValidator(Decimal("0"))],
    )
    discount_price = models.DecimalField(max_digits=12, decimal_places=2, null=True, blank=True)
    level = models.PositiveSmallIntegerField(choices=Level, default=Level.BEGINNER)
    status = models.CharField(max_length=10, choices=Status, default=Status.DRAFT)
    cover = models.ImageField(upload_to="courses/covers/", blank=True)
    published_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        verbose_name = "Kurs"
        verbose_name_plural = "Kurslar"
        constraints = [
            models.CheckConstraint(condition=Q(price__gte=0), name="course_price_gte_0"),
            models.CheckConstraint(
                condition=Q(discount_price__isnull=True) | Q(discount_price__lt=F("price")),
                name="course_discount_lt_price",
            ),
            models.CheckConstraint(
                condition=~Q(status="published") | Q(published_at__isnull=False),
                name="course_published_has_date",
            ),
        ]
        indexes = [
            models.Index(
                fields=["category", "-published_at"],
                condition=Q(status="published"),
                name="course_cat_published_idx",
            ),
        ]

    def __str__(self):
        return self.title

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.title)[:200] or str(self.public_id)
        super().save(*args, **kwargs)

    def get_absolute_url(self):
        return reverse("courses:detail", kwargs={"slug": self.slug})

    @property
    def final_price(self) -> Decimal:
        return self.discount_price if self.discount_price is not None else self.price

    @property
    def is_free(self) -> bool:
        return self.final_price == 0


class Module(models.Model):
    course = models.ForeignKey(Course, on_delete=models.CASCADE, related_name="modules")
    title = models.CharField(max_length=200)
    order = models.PositiveSmallIntegerField()

    class Meta:
        ordering = ["order"]
        constraints = [
            models.UniqueConstraint(fields=["course", "order"], name="uniq_module_order"),
        ]

    def __str__(self):
        return f"{self.order}. {self.title}"


class Lesson(TimeStampedModel):
    class Type(models.TextChoices):
        VIDEO = "video", "Video"
        TEXT = "text", "Matn"

    module = models.ForeignKey(Module, on_delete=models.CASCADE, related_name="lessons")
    title = models.CharField(max_length=200)
    order = models.PositiveSmallIntegerField()
    type = models.CharField(max_length=10, choices=Type)
    video_url = models.URLField(max_length=500, blank=True)
    content = models.TextField(blank=True)
    duration = models.DurationField(null=True, blank=True)
    is_preview = models.BooleanField(default=False, help_text="Yozilmagan foydalanuvchilarga ham ochiq")

    class Meta:
        ordering = ["order"]
        constraints = [
            models.UniqueConstraint(fields=["module", "order"], name="uniq_lesson_order"),
            models.CheckConstraint(
                condition=~Q(type="video") | ~Q(video_url=""),
                name="lesson_video_has_url",
            ),
        ]

    def __str__(self):
        return self.title


class Enrollment(TimeStampedModel):
    student = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="enrollments"
    )
    course = models.ForeignKey(Course, on_delete=models.PROTECT, related_name="enrollments")
    paid_amount = models.DecimalField(max_digits=12, decimal_places=2, default=Decimal("0"))
    completed_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        constraints = [
            models.UniqueConstraint(fields=["student", "course"], name="uniq_enrollment"),
        ]

    def __str__(self):
        return f"{self.student_id} → {self.course_id}"


class LessonProgress(models.Model):
    student = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="lesson_progress"
    )
    lesson = models.ForeignKey(Lesson, on_delete=models.CASCADE, related_name="progress")
    completed_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        constraints = [
            models.UniqueConstraint(fields=["student", "lesson"], name="uniq_lesson_progress"),
        ]


class Review(TimeStampedModel):
    course = models.ForeignKey(Course, on_delete=models.CASCADE, related_name="reviews")
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="reviews"
    )
    rating = models.PositiveSmallIntegerField(
        validators=[MinValueValidator(1), MaxValueValidator(5)]
    )
    comment = models.TextField(blank=True)

    class Meta:
        constraints = [
            models.UniqueConstraint(fields=["course", "author"], name="uniq_review_per_user"),
            models.CheckConstraint(condition=Q(rating__gte=1, rating__lte=5), name="review_rating_1_5"),
        ]

    def __str__(self):
        return f"{self.rating}★ — {self.course_id}"
```

### Dizayn qarorlari (nima uchun shunday?)

| Qaror | Sabab |
|-------|-------|
| `Category.parent` → `PROTECT` | Ichida bola kategoriya bo'lsa, tasodifan o'chirib yuborilmasin |
| `Course.category` → `PROTECT` | Kurslari bor kategoriya o'chirilmasin |
| `Enrollment.course` → `PROTECT` | To'lov tarixi bor kursni o'chirib bo'lmaydi (moliyaviy ma'lumot) |
| `Enrollment.student` → `CASCADE` | ⚠️ Munozarali — real loyihada foydalanuvchini o'chirmasdan `is_active=False` qilish yaxshiroq |
| `Course.students` → `through=Enrollment` | Yozilish sanasi, to'lov summasi kerak |
| Pul → `DecimalField` | Aniq hisob |
| `public_id` (UUID) | URL/API da ketma-ket ID ni oshkor qilmaslik |
| `published` bo'lsa `published_at` majburiy → `CheckConstraint` | Noto'g'ri holat bazada ham imkonsiz |
| `Module.order` + `UniqueConstraint` | Bitta kursda ikkita "1-modul" bo'lmasin |
| Partial index faqat `published` kurslar uchun | Asosiy sahifa so'rovi tez, index kichik |

---

## 8.4. Shell da sinab ko'ring

```python
from decimal import Decimal
from django.db.models import Avg, Count, Q
from accounts.models import User
from courses.models import *

teacher = User.objects.create_user("ustoz", email="u@mail.uz", password="x", role="teacher")
student = User.objects.create_user("talaba", email="t@mail.uz", password="x")

backend = Category.objects.create(name="Backend", slug="backend")
django_cat = Category.objects.create(name="Django", slug="django", parent=backend)

course = Course.objects.create(title="Django Advanced", category=django_cat, price=Decimal("500000"))
course.teachers.add(teacher)
course.students.add(student, through_defaults={"paid_amount": Decimal("500000")})

# Eng yuqori baholangan, nashr qilingan kurslar — reyting va talabalar soni bilan
Course.objects.filter(status="published").annotate(
    avg_rating=Avg("reviews__rating"),
    students_count=Count("enrollments", distinct=True),
).order_by("-avg_rating")

# Constraint larni sinash
Course.objects.create(title="X", category=django_cat, price=-1)                    # IntegrityError
Course.objects.create(title="Y", category=django_cat, status="published")          # IntegrityError
backend.delete()                                                                    # ProtectedError
```

---

## 8.5. Yakuniy topshiriqlar

### A. Kodni kengaytiring

1. `Lesson` ga `GeneratedField` yoki `annotate` bilan kursning umumiy davomiyligini hisoblang.
2. `Enrollment` ga `progress_percent` ni hisoblovchi metod yozing (tugatilgan darslar / jami darslar).
3. `Coupon` modelini qo'shing: kod (katta-kichik harfga qaramay unikal), chegirma foizi (1–100), amal qilish muddati (`valid_from < valid_to` constraint), maksimal foydalanish soni, ishlatilganlar soni (`used_count <= max_uses` constraint).
4. `Course` uchun `PublishedCourse` proxy modelini yozing va `publish()` metodini qo'shing: `status` va `published_at` ni birga o'zgartirsin.
5. `Review` qoldirish uchun talaba kursga yozilgan bo'lishi kerak — buni `clean()` da tekshiring. Nima uchun buni `CheckConstraint` bilan qilib bo'lmaydi?

### B. Code review

Quyidagi modelda **kamida 12 ta** xato yoki yomon amaliyotni toping va tuzating:

```python
from django.contrib.auth.models import User
from django.db import models
import datetime


class Products(models.Model):
    Name = models.CharField(max_length=100, null=True)
    price = models.FloatField()
    phone = models.IntegerField()
    category_id = models.ForeignKey("Category", on_delete=models.CASCADE)
    owner = models.ForeignKey(User, on_delete=models.CASCADE)
    editor = models.ForeignKey(User, on_delete=models.CASCADE)
    tags = models.ManyToManyField("Tag", null=True)
    created = models.DateTimeField(default=datetime.datetime.now())
    data = models.JSONField(default={})
    is_active = models.BooleanField()
    status = models.CharField(max_length=20, choices=[("a", "Active"), ("d", "Draft")])

    def save(self):
        self.Name = self.Name.title()
        super().save()
```

<details>
<summary>Javoblar (avval o'zingiz urinib ko'ring!)</summary>

1. `Products` → `Product` (birlik shakl).
2. `Name` → `name` (snake_case).
3. `CharField` da `null=True` — kerak emas, `blank=True` ishlating (yoki majburiy qoldiring).
4. `price` — `FloatField` emas, `DecimalField`.
5. `phone` — `IntegerField` emas, `CharField` (+998, boshidagi nol).
6. `category_id` → `category` (Django o'zi `_id` qo'shadi, aks holda ustun `category_id_id` bo'ladi).
7. `User` ni to'g'ridan-to'g'ri import qilish → `settings.AUTH_USER_MODEL`.
8. `owner` va `editor` — bir xil modelga ikki FK, `related_name` yo'q → `fields.E304` xatosi.
9. `on_delete=CASCADE` hamma joyda — `category` uchun `PROTECT`, `editor` uchun `SET_NULL` mantiqiyroq.
10. M2M da `null=True` — ta'sirsiz, `blank=True` kerak.
11. `default=datetime.datetime.now()` — chaqirilgan (bir marta hisoblanadi) va naive datetime → `auto_now_add=True` yoki `default=timezone.now`.
12. `JSONField(default={})` — mutable default → `default=dict`.
13. `BooleanField()` da `default` yo'q.
14. `choices` — magic string lar, `TextChoices` ishlating; `default` yo'q.
15. `save(self)` — `*args, **kwargs` qabul qilmaydi va uzatmaydi (`update_fields` buziladi).
16. `self.Name.title()` — `Name` `None` bo'lsa `AttributeError`.
17. `__str__` yo'q.
18. `Meta` yo'q (verbose_name, constraints: narx > 0).

</details>

---

## 8.6. Intervyu savollari

Shogird quyidagi savollarga og'zaki, misollar bilan javob bera olishi kerak:

1. `null=True` va `blank=True` farqi nima? `CharField` da nega `null=True` tavsiya etilmaydi? Istisnosi qanday?
2. `default` va `db_default` farqi nima?
3. `auto_now` va `auto_now_add` qachon ishlamaydi?
4. Pulni qaysi field da saqlash kerak va nima uchun `FloatField` emas?
5. `on_delete` ning barcha variantlarini sanab bering. `PROTECT` va `RESTRICT` farqi nima?
6. `related_name` nima va qachon majburiy?
7. `ForeignKey(unique=True)` va `OneToOneField` farqi?
8. M2M bazada qanday saqlanadi? `through` qachon kerak?
9. `select_related` va `prefetch_related` farqi nima? N+1 muammo nima?
10. Validatorlar `save()` da ishlaydimi? `full_clean()` qachon avtomatik chaqiriladi?
11. `choices` bazada cheklov yaratadimi? Qanday qilib bazada ham himoya qilish mumkin?
12. Abstract, multi-table va proxy inheritance farqlari? Qaysi biri performance uchun xavfli?
13. `QuerySet.update()` va `obj.save()` farqi? Qaysi biri signal yuboradi?
14. `UniqueConstraint` ning `unique=True` va `unique_together` dan qanday afzalliklari bor?
15. Partial index nima va qachon foydali?
16. `self._state.adding` nima uchun `self.pk is None` dan ishonchliroq?
17. Custom field yozishda `deconstruct()` nima uchun kerak?
18. `AUTH_USER_MODEL` ni nima uchun loyiha boshida o'rnatish kerak?
19. UUID ni primary key qilishning afzallik va kamchiliklari?
20. `GeneratedField` va `@property` farqi?

---

## 8.7. Modulni tugatganlik mezoni ✅

- [ ] Barcha 8 darsdagi topshiriqlar bajarilgan
- [ ] LMS loyihasi ishlaydi, migratsiyalar toza (`makemigrations --check` bo'sh)
- [ ] Har bir constraint shell da sinab ko'rilgan
- [ ] Code review topshirig'ida 12+ xato topilgan
- [ ] Intervyu savollarining kamida 16 tasiga to'liq javob berilgan

➡️ **Keyingi modul:** QuerySet API va ORM chuqur (filter, Q, F, annotate, aggregate, Subquery, Window functions).

⬅️ [07-validator-va-custom-field.md](07-validator-va-custom-field.md) | 🏠 [README](../README.md)
