# 4. Bog'lanish (relationship) fieldlari

Relyatsion bazaning kuchi — jadvallar orasidagi bog'lanishlarda. Django uchta asosiy turini beradi:

| Bog'lanish | Field | Misol |
|------------|-------|-------|
| **Ko'pdan-birga** (Many-to-One) | `ForeignKey` | Ko'p `Post` → bitta `Author` |
| **Birga-bir** (One-to-One) | `OneToOneField` | Bitta `User` ↔ bitta `Profile` |
| **Ko'pdan-ko'pga** (Many-to-Many) | `ManyToManyField` | Ko'p `Post` ↔ ko'p `Tag` |

---

## 4.1. `ForeignKey`

```python
from django.conf import settings
from django.db import models


class Category(models.Model):
    name = models.CharField(max_length=100)


class Post(models.Model):
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL,       # User modeliga doim shunday murojaat qiling!
        on_delete=models.CASCADE,
        related_name="posts",
    )
    category = models.ForeignKey(
        Category,
        on_delete=models.PROTECT,
        related_name="posts",
        null=True,
        blank=True,
    )
```

### Bazada nima bo'ladi?

```sql
"author_id" bigint NOT NULL REFERENCES "auth_user" ("id") DEFERRABLE INITIALLY DEFERRED,
CREATE INDEX "blog_post_author_id_..." ON "blog_post" ("author_id");
```

- Ustun nomi: `<field_nomi>_id` → `author_id`.
- Avtomatik **index** qo'yiladi.
- `FOREIGN KEY` constraint yaratiladi.

### `author` va `author_id` farqi (performance!)

```python
post.author        # Author obyekti — agar oldin yuklanmagan bo'lsa, QO'SHIMCHA SQL so'rov!
post.author_id     # shunchaki raqam — so'rovsiz

# YOMON — keraksiz so'rov
if post.author.id == request.user.id: ...

# YAXSHI
if post.author_id == request.user.id: ...

# Yaratishda ham:
Post.objects.create(title="...", author_id=5)   # Author ni bazadan olib kelish shart emas
```

### Modelga murojaat qilish usullari

```python
models.ForeignKey(Category, ...)               # 1. class ning o'zi (oldin e'lon qilingan bo'lishi kerak)
models.ForeignKey("Category", ...)             # 2. string — hali e'lon qilinmagan model uchun (lazy)
models.ForeignKey("shop.Category", ...)        # 3. boshqa app dagi model — "app_label.ModelName"
models.ForeignKey("self", ...)                 # 4. o'ziga o'zi (rekursiv)
models.ForeignKey(settings.AUTH_USER_MODEL, ...)  # 5. User uchun — HECH QACHON `from django.contrib.auth.models import User` emas
```

String murojaat **circular import** muammosini hal qiladi (ikki app bir-biriga murojaat qilganda).

---

## 4.2. `on_delete` — bog'langan obyekt o'chirilganda nima bo'ladi?

Bu **majburiy** parametr va eng muhim dizayn qarorlaridan biri.

| Variant | Nima qiladi | Qachon ishlatiladi |
|---------|-------------|-------------------|
| `CASCADE` | Bog'langan obyektlarni ham o'chiradi | Bola ota-onasiz ma'nosiz: `Comment` → `Post`, `OrderItem` → `Order` |
| `PROTECT` | O'chirishga ruxsat bermaydi (`ProtectedError`) | Muhim ma'lumot: `Order` → `Product`, `Post` → `Category` |
| `RESTRICT` | `PROTECT` ga o'xshash, lekin CASCADE zanjiri orqali o'chirishga ruxsat beradi | Murakkab ierarxiyalar (pastda misol) |
| `SET_NULL` | FK ni `NULL` qiladi (`null=True` **shart**) | Ixtiyoriy bog'lanish: `Post.category`, `Task.assignee` |
| `SET_DEFAULT` | FK ni `default` qiymatga o'rnatadi (`default` **shart**) | "Noma'lum" kategoriyaga o'tkazish |
| `SET(...)` | Berilgan qiymat yoki callable natijasiga o'rnatadi | O'chirilgan foydalanuvchi o'rniga "deleted user" |
| `DO_NOTHING` | Django hech narsa qilmaydi | Baza o'zi boshqaradi (`ON DELETE` SQL da) — ⚠️ aks holda `IntegrityError` |

### Misollar

```python
from django.contrib.auth import get_user_model


def get_sentinel_user():
    return get_user_model().objects.get_or_create(username="deleted")[0]


class Comment(models.Model):
    post = models.ForeignKey("Post", on_delete=models.CASCADE, related_name="comments")
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.SET(get_sentinel_user),   # foydalanuvchi o'chsa, izohlar "deleted" ga o'tadi
    )


class OrderItem(models.Model):
    order = models.ForeignKey("Order", on_delete=models.CASCADE, related_name="items")
    product = models.ForeignKey("Product", on_delete=models.PROTECT)   # sotilgan mahsulotni o'chirib bo'lmaydi
```

```python
from django.db.models import ProtectedError

try:
    product.delete()
except ProtectedError as e:
    print("O'chirib bo'lmaydi, bog'langan:", e.protected_objects)
```

### `PROTECT` vs `RESTRICT` farqi

```python
class Artist(models.Model): ...

class Album(models.Model):
    artist = models.ForeignKey(Artist, on_delete=models.CASCADE)

class Song(models.Model):
    artist = models.ForeignKey(Artist, on_delete=models.CASCADE)
    album = models.ForeignKey(Album, on_delete=models.RESTRICT)
```

- `album.delete()` → ❌ `RestrictedError` (unda qo'shiq bor).
- `artist.delete()` → ✅ Ishlaydi! Artist o'chirilganda Album ham, Song ham CASCADE orqali o'chadi — RESTRICT bunga ruxsat beradi, chunki qo'shiq ham bir vaqtda o'chirilyapti.
- Agar `PROTECT` bo'lganida — `artist.delete()` ham xato berardi.

### ⚠️ CASCADE xavfi

```python
user.delete()   # → uning barcha post, comment, order, payment ... lari o'chib ketadi!
```

Moliyaviy/yuridik ahamiyatga ega ma'lumotlar uchun CASCADE ishlatmang. Ko'p hollarda o'chirish o'rniga **soft delete** (`is_active=False` yoki `deleted_at`) yaxshiroq.

> **Eslatma:** yuqoridagi `on_delete` variantlari Django darajasida (Python da) ishlaydi — bazada `ON DELETE CASCADE` yozilmaydi. Django avval bog'langan obyektlarni topadi, keyin ularni o'chiradi (shuning uchun signallar ishlaydi, lekin katta hajmda sekin). Raw SQL bilan o'chirsangiz, Django qoidalari ishlamaydi. Yangi Django versiyalarida baza darajasidagi variantlar (`DB_CASCADE`, `DB_SET_NULL` ...) qo'shilgan — o'z versiyangiz hujjatini tekshiring.

---

## 4.3. `related_name` va `related_query_name` — teskari bog'lanish

```python
class Post(models.Model):
    author = models.ForeignKey(User, on_delete=models.CASCADE, related_name="posts")
```

```python
post.author            # oldinga: Post → User
user.posts.all()       # teskari: User → Post'lar (related_name tufayli)

# related_name berilmasa: user.post_set.all()
```

### `related_query_name`

Filter da teskari nom:

```python
class Post(models.Model):
    author = models.ForeignKey(
        User, on_delete=models.CASCADE,
        related_name="posts",          # user.posts.all()
        related_query_name="post",     # User.objects.filter(post__title="...")
    )

User.objects.filter(post__is_published=True).distinct()
# related_query_name berilmasa, related_name ishlatiladi: filter(posts__is_published=True)
```

### Teskari bog'lanishni o'chirish

```python
created_by = models.ForeignKey(User, on_delete=models.CASCADE, related_name="+")   # user.xxx_set yaratilmaydi
```

### ⚠️ Bitta modelga ikki FK — `related_name` majburiy

```python
class Message(models.Model):
    sender = models.ForeignKey(User, on_delete=models.CASCADE, related_name="sent_messages")
    receiver = models.ForeignKey(User, on_delete=models.CASCADE, related_name="received_messages")
```

`related_name` bo'lmasa Django xato beradi: `fields.E304 Reverse accessor clashes...`

### Abstract model da `related_name`

```python
class BaseItem(models.Model):
    owner = models.ForeignKey(
        User, on_delete=models.CASCADE,
        related_name="%(app_label)s_%(class)s_items",   # har bir child uchun unikal nom
    )
    class Meta:
        abstract = True
```

---

## 4.4. ForeignKey ning boshqa parametrlari

### `limit_choices_to`

Admin va ModelForm da tanlovlarni cheklaydi:

```python
assignee = models.ForeignKey(
    User, on_delete=models.SET_NULL, null=True,
    limit_choices_to={"is_staff": True, "is_active": True},
)
# yoki Q obyekt / callable:
limit_choices_to=models.Q(is_staff=True) | models.Q(is_superuser=True)
```

⚠️ Bu faqat **forma darajasida** — bazada cheklov yo'q.

### `to_field`

PK o'rniga boshqa unikal ustunga bog'lanish:

```python
country = models.ForeignKey("Country", to_field="iso_code", on_delete=models.PROTECT)
# Country.iso_code unique=True bo'lishi shart
```

### `db_constraint`

```python
legacy_user = models.ForeignKey(User, on_delete=models.DO_NOTHING, db_constraint=False)
```

Bazada FOREIGN KEY constraint yaratilmaydi. Faqat maxsus holatlarda: legacy baza, sharding, juda katta jadvallarda yozish tezligi. ⚠️ Ma'lumot yaxlitligi kafolati yo'qoladi.

### Rekursiv (o'ziga o'zi) bog'lanish — daraxt tuzilmasi

```python
class Category(models.Model):
    name = models.CharField(max_length=100)
    parent = models.ForeignKey(
        "self",
        on_delete=models.CASCADE,
        null=True, blank=True,
        related_name="children",
    )

electronics = Category.objects.create(name="Elektronika")
phones = Category.objects.create(name="Telefonlar", parent=electronics)
electronics.children.all()    # <QuerySet [<Category: Telefonlar>]>
phones.parent                 # <Category: Elektronika>
```

> Chuqur daraxtlar uchun: `django-mptt` yoki `django-treebeard`, yoki PostgreSQL recursive CTE.

---

## 4.5. `OneToOneField`

Aslida `ForeignKey(unique=True)` — lekin teskari tomon **obyekt** qaytaradi (QuerySet emas).

```python
class Profile(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name="profile",
    )
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to="avatars/", blank=True)
```

```python
profile.user           # User
user.profile           # Profile (QuerySet emas!)

# ⚠️ Profile yo'q bo'lsa:
user.profile           # → RelatedObjectDoesNotExist (Profile.DoesNotExist) xatosi

# Xavfsiz tekshirish:
hasattr(user, "profile")

# yoki
try:
    profile = user.profile
except Profile.DoesNotExist:
    profile = Profile.objects.create(user=user)
```

### Qachon OneToOne?

- Mavjud modelni (masalan, `User`) **o'zgartirmasdan kengaytirish** (Profile).
- Kam ishlatiladigan, og'ir ustunlarni asosiy jadvaldan ajratish (masalan, `ProductDetail` — katta matn va JSON).
- Multi-table inheritance — Django uni ichkarida `OneToOneField(parent_link=True)` orqali qiladi (5-dars).

### ForeignKey(unique=True) vs OneToOneField

```python
user = models.ForeignKey(User, on_delete=models.CASCADE, unique=True)
# Ishlaydi, lekin Django ogohlantiradi (fields.W342) va user.profile_set.get() deb yozishga majbur qiladi
```

Doim `OneToOneField` ishlating.

---

## 4.6. `ManyToManyField`

```python
class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True)


class Post(models.Model):
    title = models.CharField(max_length=200)
    tags = models.ManyToManyField(Tag, related_name="posts", blank=True)
```

### Bazada nima bo'ladi?

`blog_post` jadvalida **ustun yaratilmaydi**. O'rniga **oraliq (junction) jadval** yaratiladi:

```sql
CREATE TABLE "blog_post_tags" (
    "id" bigint PRIMARY KEY,
    "post_id" bigint REFERENCES "blog_post",
    "tag_id" bigint REFERENCES "blog_tag",
    UNIQUE ("post_id", "tag_id")
);
```

Shuning uchun:
- `null=True` M2M ga **ta'sir qilmaydi** (Django ogohlantiradi). Ixtiyoriy qilish uchun faqat `blank=True`.
- M2M ga `on_delete` yo'q — oraliq jadvaldagi qatorlar doim CASCADE bilan o'chadi.
- Obyekt **saqlanmaguncha** M2M ga qo'shib bo'lmaydi (unda hali `id` yo'q).

### M2M bilan ishlash (related manager metodlari)

```python
post = Post.objects.create(title="Django")
django_tag = Tag.objects.create(name="django")
python_tag = Tag.objects.create(name="python")

post.tags.add(django_tag, python_tag)       # qo'shish (bir nechtasini birdan)
post.tags.add(1, 2)                         # PK lar bilan ham bo'ladi
post.tags.remove(python_tag)                # oraliq jadvaldan o'chirish (Tag o'chmaydi)
post.tags.set([django_tag, python_tag])     # ro'yxatni to'liq almashtirish
post.tags.clear()                           # barcha bog'lanishlarni o'chirish
post.tags.create(name="orm")                # yangi Tag yaratib, darhol bog'lash
post.tags.all()
post.tags.filter(name__startswith="dj")
post.tags.count()
post.tags.contains(django_tag)              # Django 4.0+ — samarali tekshirish

django_tag.posts.all()                      # teskari tomondan
Post.objects.filter(tags__name="django")
```

⚠️ `add()`, `remove()`, `set()`, `clear()` darhol bazaga yoziladi, `save()` chaqirish shart emas. Ular `post_save` emas, **`m2m_changed`** signalini yuboradi.

### `ModelForm` da M2M

`form.save(commit=False)` qilsangiz, keyin `form.save_m2m()` ni **albatta** chaqiring — aks holda M2M saqlanmaydi.

---

## 4.7. `through` — oraliq jadvalga qo'shimcha ma'lumot

Oddiy M2M faqat "kim kim bilan bog'langan" ni saqlaydi. "Qachon qo'shildi?", "qanday rolda?" kabi ma'lumot kerak bo'lsa — o'z oraliq modelingizni yozasiz:

```python
class Student(models.Model):
    name = models.CharField(max_length=100)


class Course(models.Model):
    title = models.CharField(max_length=200)
    students = models.ManyToManyField(
        Student,
        through="Enrollment",
        related_name="courses",
    )


class Enrollment(models.Model):
    class Status(models.TextChoices):
        ACTIVE = "active", "Faol"
        COMPLETED = "completed", "Tugatgan"
        DROPPED = "dropped", "Chiqib ketgan"

    student = models.ForeignKey(Student, on_delete=models.CASCADE)
    course = models.ForeignKey(Course, on_delete=models.CASCADE)
    enrolled_at = models.DateTimeField(auto_now_add=True)
    status = models.CharField(max_length=20, choices=Status, default=Status.ACTIVE)
    grade = models.PositiveSmallIntegerField(null=True, blank=True)

    class Meta:
        constraints = [
            models.UniqueConstraint(fields=["student", "course"], name="unique_enrollment"),
        ]
```

```python
# 1-usul: oraliq modelni to'g'ridan-to'g'ri yaratish
Enrollment.objects.create(student=ali, course=django_course, status="active")

# 2-usul: through_defaults bilan add/create/set
django_course.students.add(ali, through_defaults={"status": "active"})

# So'rovlar
django_course.students.all()                                    # barcha talabalar
Student.objects.filter(enrollment__status="completed", enrollment__grade__gte=80)
ali.enrollment_set.select_related("course")                    # oraliq yozuvlar
```

> **Maslahat:** loyihada M2M bog'lanishga keyinchalik qo'shimcha ma'lumot kerak bo'lishi ehtimoli katta bo'lsa, boshidanoq `through` model yozing. Oddiy M2M ni keyin `through` ga o'tkazish — murakkab migratsiya.

### `through_fields`

Oraliq modelda bir xil modelga bir nechta FK bo'lsa, qaysi biri ishlatilishini aniqlaydi:

```python
class Group(models.Model):
    members = models.ManyToManyField(
        "Person",
        through="Membership",
        through_fields=("group", "person"),   # (manba, nishon)
    )


class Membership(models.Model):
    group = models.ForeignKey(Group, on_delete=models.CASCADE)
    person = models.ForeignKey("Person", on_delete=models.CASCADE)
    invited_by = models.ForeignKey("Person", on_delete=models.SET_NULL, null=True, related_name="+")
```

---

## 4.8. O'ziga M2M va `symmetrical`

```python
class Person(models.Model):
    friends = models.ManyToManyField("self", blank=True)                    # symmetrical=True (default)
    following = models.ManyToManyField(
        "self", symmetrical=False, related_name="followers", blank=True
    )
```

- **Do'stlik** — simmetrik: Ali Valining do'sti bo'lsa, Vali ham Alining do'sti. `ali.friends.add(vali)` → `vali.friends.all()` da Ali ham ko'rinadi.
- **Obuna** — simmetrik emas: Ali Valiga obuna bo'lsa, Vali Aliga obuna bo'lishi shart emas.

```python
ali.following.add(vali)
ali.following.all()     # [vali]
vali.followers.all()    # [ali]
vali.following.all()    # []
```

---

## 4.9. N+1 muammosi — relationship bilan ishlashning eng muhim mavzusi

```python
# ❌ N+1: 1 ta so'rov postlar uchun + har bir post uchun 1 ta author so'rovi
posts = Post.objects.all()
for post in posts:
    print(post.author.username)     # 100 post = 101 SQL so'rov!
```

### `select_related` — ForeignKey va OneToOne uchun (SQL JOIN)

```python
posts = Post.objects.select_related("author", "category")   # 1 ta so'rov
for post in posts:
    print(post.author.username, post.category.name)

Comment.objects.select_related("post__author")              # zanjir bo'ylab
```

### `prefetch_related` — ManyToMany va teskari FK uchun (alohida so'rov + Python da birlashtirish)

```python
posts = Post.objects.prefetch_related("tags", "comments")   # 3 ta so'rov (har qancha post bo'lsa ham)
for post in posts:
    print([t.name for t in post.tags.all()])

# Prefetch ni filtrlash
from django.db.models import Prefetch

Post.objects.prefetch_related(
    Prefetch(
        "comments",
        queryset=Comment.objects.filter(is_approved=True).select_related("author"),
        to_attr="approved_comments",
    )
)
# post.approved_comments — list
```

| | `select_related` | `prefetch_related` |
|--|------------------|--------------------|
| Bog'lanish turi | FK, O2O (oldinga yoki teskari O2O) | M2M, teskari FK, GenericFK, istalgan |
| Mexanizm | SQL `JOIN` | Alohida `WHERE id IN (...)` so'rov |
| So'rovlar soni | 1 | 1 + bog'lanishlar soni |

> So'rovlar sonini kuzatish uchun: **django-debug-toolbar** yoki testlarda `self.assertNumQueries(2)`. 2-modulda (QuerySet) bu mavzu chuqur o'rganiladi.

---

## 4.10. Generic relations (`contenttypes`)

Bitta model **istalgan** modelga bog'lanishi kerak bo'lsa (izoh postga ham, videoga ham, mahsulotga ham):

```python
from django.contrib.contenttypes.fields import GenericForeignKey, GenericRelation
from django.contrib.contenttypes.models import ContentType


class Comment(models.Model):
    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    object_id = models.PositiveBigIntegerField()
    content_object = GenericForeignKey("content_type", "object_id")
    text = models.TextField()

    class Meta:
        indexes = [models.Index(fields=["content_type", "object_id"])]


class Post(models.Model):
    title = models.CharField(max_length=200)
    comments = GenericRelation(Comment)     # teskari bog'lanish + CASCADE delete


class Video(models.Model):
    url = models.URLField()
    comments = GenericRelation(Comment)
```

```python
Comment.objects.create(content_object=post, text="Zo'r!")
Comment.objects.create(content_object=video, text="Ajoyib video")
post.comments.all()
```

⚠️ **Kamchiliklari:** bazada FK constraint yo'q, JOIN qiyin, `filter(content_object=...)` ishlamaydi, `select_related` ishlamaydi (faqat `prefetch_related`). Iloji bo'lsa oddiy FK lardan foydalaning (masalan, nullable `post` va `video` FK + `CheckConstraint` "faqat bittasi to'ldirilgan").

---

## Xulosa jadvali

| Parametr | FK | O2O | M2M |
|----------|----|-----|-----|
| `on_delete` | ✅ majburiy | ✅ majburiy | ❌ |
| `related_name` | ✅ | ✅ | ✅ |
| `related_query_name` | ✅ | ✅ | ✅ |
| `limit_choices_to` | ✅ | ✅ | ✅ |
| `to_field` | ✅ | ✅ | ❌ |
| `db_constraint` | ✅ | ✅ | ✅ |
| `null` | ✅ | ✅ | ❌ ta'sirsiz |
| `through`, `through_fields` | ❌ | ❌ | ✅ |
| `symmetrical` | ❌ | ❌ | ✅ (faqat `"self"`) |
| `parent_link` | ❌ | ✅ | ❌ |
| Teskari tomon qaytaradi | Manager (QuerySet) | Obyekt | Manager (QuerySet) |

---

## Topshiriqlar

1. **Kutubxona tizimi:** `Author`, `Book`, `Genre`, `Member`, `Loan` modellarini yozing.
   - Kitobning bir nechta muallifi va janri bo'lishi mumkin.
   - `Loan` — a'zo kitobni qachon olgani, qachon qaytarishi kerakligi, qachon qaytargani.
   - Har bir `on_delete` ni asoslab tanlang (kitob o'chirilsa, ijaralar tarixi nima bo'ladi?).
2. `Category` uchun 3 darajali daraxt yarating va "Elektronika" ning barcha bevosita bolalarini chiqaring.
3. `Person` uchun `following` (simmetrik emas) yozing. "Alining obunachilari ichida Ali o'zi ham obuna bo'lganlar" (mutual) ni toping.
4. 50 ta post va har biriga 3 ta tag yarating. `assertNumQueries` yoki `connection.queries` yordamida `prefetch_related` dan oldin va keyin so'rovlar sonini o'lchang.
5. `PROTECT` va `RESTRICT` farqini shell da amalda ko'rsating.

⬅️ [03-field-turlari.md](03-field-turlari.md) | ➡️ [05-meta-va-inheritance.md](05-meta-va-inheritance.md)
