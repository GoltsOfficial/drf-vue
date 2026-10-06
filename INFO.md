# 📘 Django (DRF) + Vue — Этап: Аутентификация / Авторизация

---

## 🔹 Часть 1. Модель `User`

```python
from django.contrib.auth.models import AbstractUser
from django.db import models


class User(AbstractUser):
    """Custom user model"""
    email = models.EmailField(unique=True)  # уникальный email (логин)
    first_name = models.CharField(max_length=30, blank=True)  # имя
    last_name = models.CharField(max_length=30, blank=True)  # фамилия
    avatar = models.ImageField(upload_to='avatars/', blank=True, null=True)  # аватар
    bio = models.TextField(max_length=500, blank=True)  # биография
    created_at = models.DateTimeField(auto_now_add=True)  # дата создания
    updated_at = models.DateTimeField(auto_now=True)  # дата обновления
    is_active = models.BooleanField(default=True)  # флаг активности

    USERNAME_FIELD = 'email'  # вход по email
    REQUIRED_FIELDS = ["username"]  # обязательные поля для createsuperuser

    class Meta:
        db_table = 'user'  # имя таблицы в БД
        verbose_name = 'User'
        verbose_name_plural = 'Users'

    def __str__(self):
        return self.email  # строковое представление

    @property
    def full_name(self):
        return f'{self.first_name} {self.last_name}'.strip()  # "Имя Фамилия"
```

**Пример queryset:** `User.objects.filter(is_active=True)` → `<QuerySet [<User: ivan@example.com>, ...]>`

---

## 🔹 Часть 2. Сериализаторы для аутентификации

### 📌 `UserRegistrationSerializer` — регистрация

```python
from django.contrib.auth.password_validation import validate_password
from rest_framework import serializers

from .models import User


class UserRegistrationSerializer(serializers.ModelSerializer):
    """Serializer for users registration"""
    password = serializers.CharField(
        write_only=True,  # пароль не возвращается в ответе
        validators=[validate_password]  # встроенная валидация Django
    )
    password_confirm = serializers.CharField(write_only=True)  # подтверждение пароля

    class Meta:
        model = User
        fields = (
            'username', 'email', 'password', 'password_confirm',
            'first_name', 'last_name'
        )

    def validate(self, attrs):
        """Проверяем, что пароли совпадают"""
        if attrs['password'] != attrs['password_confirm']:
            raise serializers.ValidationError(
                {"password": "Password does not match."}
            )
        return attrs

    def create(self, validated_data):
        """Создаём пользователя через create_user (пароль хешируется)"""
        validated_data.pop("password_confirm")  # убираем лишнее поле
        user = User.objects.create_user(**validated_data)
        return user
```

#### 🧪 Пример данных на вход

```json
POST /api/auth/register/
{
  "username": "ivan",
  "email": "ivan@example.com",
  "password": "StrongPass123!",
  "password_confirm": "StrongPass123!",
  "first_name": "Иван",
  "last_name": "Иванов"
}
```

#### 🧪 Пример ответа (201 Created)

```json
{
  "username": "ivan",
  "email": "ivan@example.com",
  "first_name": "Иван",
  "last_name": "Иванов"
}
```

#### ❌ Пример ошибки (400 Bad Request)

```json
{
  "password": [
    "Password does not match."
  ]
}
```

---

### 📌 `UserLoginSerializer` — вход

```python
from django.contrib.auth import authenticate


class UserLoginSerializer(serializers.ModelSerializer):
    """Serializer for user login"""
    email = serializers.EmailField()
    password = serializers.CharField(write_only=True)

    def validate(self, attrs):
        """Проверяем email + пароль через authenticate()"""
        email = attrs.get('email')
        password = attrs.get('password')

        if email and password:
            user = authenticate(
                request=self.context.get('request'),
                email=email,
                password=password
            )
            if not user:
                raise serializers.ValidationError(
                    "Wrong username or password."
                )
            if not user.is_active:
                raise serializers.ValidationError(
                    "Your account is disabled."
                )
            attrs['user'] = user  # передаём user во view для генерации JWT
            return attrs
        else:
            raise serializers.ValidationError(
                "Must include email and password."
            )
```

#### 🧪 Пример данных на вход

```json
POST /api/auth/login/
{
  "email": "ivan@example.com",
  "password": "StrongPass123!"
}
```

#### 🧪 Пример ответа (200 OK)

```json
{
  "refresh": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "access": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

#### ❌ Пример ошибки (400 Bad Request)

```json
{
  "non_field_errors": [
    "Wrong username or password."
  ]
}
```

---

## 🔹 Часть 3. Сериализаторы профиля

### 📌 `UserProfileSerializer` — просмотр профиля

```python
class UserProfileSerializer(serializers.ModelSerializer):
    """Serializer for user profile"""
    full_name = serializers.ReadOnlyField()  # из @property модели
    posts_count = serializers.SerializerMethodField()  # считаем посты
    comments_count = serializers.SerializerMethodField()  # считаем комментарии

    class Meta:
        model = User
        fields = (
            'id', 'username', 'email', 'first_name', 'last_name',
            'full_name', 'avatar', 'bio', 'created_at', 'updated_at',
            'posts_count', 'comments_count'
        )
        read_only_fields = ('id', 'created_at', 'updated_at')  # нельзя менять

    def get_posts_count(self, obj):
        return obj.posts.count()  # связанные посты

    def get_comments_count(self, obj):
        return obj.comments.count()  # связанные комментарии
```

#### 🧪 Пример ответа (200 OK)

```json
GET /api/auth/profile/
{
  "id": 1,
  "username": "ivan",
  "email": "ivan@example.com",
  "first_name": "Иван",
  "last_name": "Иванов",
  "full_name": "Иван Иванов",
  "avatar": "/media/avatars/ivan.png",
  "bio": "Backend-разработчик",
  "created_at": "2026-10-03T12:00:00Z",
  "updated_at": "2026-10-03T12:30:00Z",
  "posts_count": 12,
  "comments_count": 47
}
```

---

### 📌 `UserUpdateSerializer` — обновление профиля

```python
class UserUpdateSerializer(serializers.ModelSerializer):
    """Serializer for user update"""

    class Meta:
        model = User
        fields = (
            'first_name', 'last_name', 'avatar', 'bio'
        )

    def update(self, instance, validated_data):
        for attr, value in validated_data.items():
            setattr(instance, attr, value)
        instance.save()
        return instance
```

#### 🧪 Пример данных на вход (PATCH)

```json
PATCH /api/auth/profile/update/
{
  "first_name": "Иван",
  "bio": "Python / Django разработчик"
}
```

#### 🧪 Пример ответа (200 OK)

```json
{
  "first_name": "Иван",
  "last_name": "Иванов",
  "avatar": "/media/avatars/ivan.png",
  "bio": "Python / Django разработчик"
}
```

---

## 🔹 Часть 4. Смена пароля

### 📌 `ChangePasswordSerializer`

```python
class ChangePasswordSerializer(serializers.ModelSerializer):
    """Serializer for user password change"""
    old_password = serializers.CharField()
    new_password = serializers.CharField(
        required=True,
        validators=[validate_password]  # проверка надёжности
    )
    new_password_confirm = serializers.CharField(required=True)

    def validate_old_password(self, value):
        """Проверяем, что старый пароль верный"""
        user = self.context['request'].user
        if not user.check_password(value):
            raise serializers.ValidationError(
                "Old password is incorrect."
            )
        return value

    def validate(self, attrs):
        """Проверяем, что новый пароль и подтверждение совпадают"""
        if attrs['new_password'] != attrs['new_password_confirm']:
            raise serializers.ValidationError(
                {"new_password": "New password is incorrect."}
            )
        return attrs

    def save(self):
        """Устанавливаем новый пароль (хешируется автоматически)"""
        user = self.context['request'].user
        user.set_password(self.validated_data['new_password'])
        user.save()
        return user
```

#### 🧪 Пример данных на вход

```json
POST /api/auth/change-password/
{
  "old_password": "StrongPass123!",
  "new_password": "EvenStronger456!",
  "new_password_confirm": "EvenStronger456!"
}
```

#### 🧪 Пример ответа (200 OK)

```json
{
  "detail": "Password changed successfully."
}
```

#### ❌ Пример ошибки (400 Bad Request)

```json
{
  "old_password": [
    "Old password is incorrect."
  ]
}
```

---

## ✅ Итог этапа

| Компонент                    | Назначение             | Endpoint (пример)                 |
|------------------------------|------------------------|-----------------------------------|
| `User` (модель)              | Кастомный пользователь | —                                 |
| `UserRegistrationSerializer` | Регистрация            | `POST /api/auth/register/`        |
| `UserLoginSerializer`        | Вход (JWT)             | `POST /api/auth/login/`           |
| `UserProfileSerializer`      | Просмотр профиля       | `GET /api/auth/profile/`          |
| `UserUpdateSerializer`       | Обновление профиля     | `PATCH /api/auth/profile/update/` |
| `ChangePasswordSerializer`   | Смена пароля           | `POST /api/auth/change-password/` |

**Следующий этап:** Views.

---
Используем классовые представления в данном случае вместо функциональных. Можно было и функциональным стилем обойтись но
мы будем идти через классовый формат.

# 📘 Django (DRF) + Vue — Шаблон для Views (классовый стиль)

> Ниже — **универсальный модульный шаблон** для написания class-based views.  
> Каждый блок можно **включать/отключать** по ситуации (доступ, фильтрация, сериализаторы и т.д.).

---

## 🔹 Базовый каркас (всегда присутствует)

```python
from rest_framework import generics, permissions, status
from rest_framework.response import Response

from .models import ModelName
from .serializers import ModelNameSerializer


class ModelNameView(generics.GenericAPIView):
    """Краткое описание view"""

    # ─────────────────────────────────────────
    # 1. QUERYSET — какие данные берём из БД
    # ─────────────────────────────────────────
    queryset = ModelName.objects.all()

    # ─────────────────────────────────────────
    # 2. SERIALIZER — как отдаём/принимаем данные
    # ─────────────────────────────────────────
    serializer_class = ModelNameSerializer

    # ─────────────────────────────────────────
    # 3. PERMISSIONS — кто имеет доступ
    # ─────────────────────────────────────────
    permission_classes = [permissions.IsAuthenticated]

    # ─────────────────────────────────────────
    # 4. HTTP METHODS — какие методы обрабатываем
    # ─────────────────────────────────────────
    def get(self, request, *args, **kwargs):
        ...
```

---

## 🔹 Модуль 1. Выбор generic-класса (что делает endpoint)

Выберите **один** базовый класс под задачу:

| Класс                                   | HTTP                 | Что делает            | Когда использовать          |
|-----------------------------------------|----------------------|-----------------------|-----------------------------|
| `generics.ListAPIView`                  | GET                  | Список объектов       | Показать все записи         |
| `generics.CreateAPIView`                | POST                 | Создание объекта      | Регистрация, добавление     |
| `generics.RetrieveAPIView`              | GET                  | Один объект по ID     | Профиль, детальная страница |
| `generics.UpdateAPIView`                | PUT/PATCH            | Обновление            | Редактирование              |
| `generics.DestroyAPIView`               | DELETE               | Удаление              | Удалить запись              |
| `generics.ListCreateAPIView`            | GET+POST             | Список + создание     | Лента + публикация          |
| `generics.RetrieveUpdateAPIView`        | GET+PUT+PATCH        | Просмотр + обновление | Профиль                     |
| `generics.RetrieveDestroyAPIView`       | GET+DELETE           | Просмотр + удаление   | Удаление из списка          |
| `generics.RetrieveUpdateDestroyAPIView` | GET+PUT+PATCH+DELETE | Полный CRUD           | Админка                     |

**Пример выбора:**

```python
# Если нужен только список:
class UserListView(generics.ListAPIView):


# Если список + создание:
class PostListCreateView(generics.ListCreateAPIView):


# Если один объект + редактирование:
class ProfileView(generics.RetrieveUpdateAPIView):
```

---

## 🔹 Модуль 2. Настройка доступа (`permission_classes`)

Выберите **один или несколько** вариантов:

```python
from rest_framework import permissions

# ─────────────────────────────────────────────
# Вариант A: Доступ для всех (регистрация, логин)
# ─────────────────────────────────────────────
permission_classes = [permissions.AllowAny]

# ─────────────────────────────────────────────
# Вариант B: Только авторизованные (профиль)
# ─────────────────────────────────────────────
permission_classes = [permissions.IsAuthenticated]

# ─────────────────────────────────────────────
# Вариант C: Только админы
# ─────────────────────────────────────────────
permission_classes = [permissions.IsAdminUser]

# ─────────────────────────────────────────────
# Вариант D: Только чтение для всех, запись для авторизованных
# ─────────────────────────────────────────────
permission_classes = [permissions.IsAuthenticatedOrReadOnly]


# ─────────────────────────────────────────────
# Вариант E: Своё правило (только владелец объекта)
# ─────────────────────────────────────────────
class IsOwnerOrReadOnly(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True
        return obj.author == request.user


permission_classes = [permissions.IsAuthenticated, IsOwnerOrReadOnly]


# ─────────────────────────────────────────────
# Вариант F: Только определённая группа
# ─────────────────────────────────────────────
class IsManager(permissions.BasePermission):
    def has_permission(self, request, view):
        if not request.user.is_authenticated:
            return False
        return request.user.groups.filter(name='Manager').exists()


permission_classes = [IsManager]
```

---

## 🔹 Модуль 3. Динамический queryset (фильтрация, поиск)

**Когда использовать:** нужно фильтровать данные по параметрам запроса или пользователю.

```python
class PostListView(generics.ListAPIView):
    serializer_class = PostSerializer
    permission_classes = [permissions.IsAuthenticated]

    def get_queryset(self):
        # Базовый queryset
        queryset = Post.objects.all()

        # ─────────────────────────────────────────
        # Фильтр 1: только опубликованные
        # ─────────────────────────────────────────
        queryset = queryset.filter(is_published=True)

        # ─────────────────────────────────────────
        # Фильтр 2: по параметру из URL (?category=tech)
        # ─────────────────────────────────────────
        category = self.request.query_params.get('category')
        if category:
            queryset = queryset.filter(category=category)

        # ─────────────────────────────────────────
        # Фильтр 3: только свои записи
        # ─────────────────────────────────────────
        if self.request.query_params.get('my') == 'true':
            queryset = queryset.filter(author=self.request.user)

        # ─────────────────────────────────────────
        # Сортировка
        # ─────────────────────────────────────────
        return queryset.order_by('-created_at')
```

---

## 🔹 Модуль 4. Динамический сериализатор (разные данные для разных ролей)

**Когда использовать:** админ видит одно, обычный пользователь — другое.

```python
class UserDetailView(generics.RetrieveAPIView):
    queryset = User.objects.all()
    permission_classes = [permissions.IsAuthenticated]

    def get_serializer_class(self):
        # ─────────────────────────────────────────
        # Админ получает полные данные
        # ─────────────────────────────────────────
        if self.request.user.is_staff:
            return UserFullSerializer

        # ─────────────────────────────────────────
        # Обычный пользователь — базовые данные
        # ─────────────────────────────────────────
        return UserPublicSerializer
```

---

## 🔹 Модуль 5. Хуки создания/обновления/удаления

**Когда использовать:** нужно добавить логику при сохранении (например, привязать автора).

```python
class PostCreateView(generics.CreateAPIView):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [permissions.IsAuthenticated]

    # ─────────────────────────────────────────
    # Вызывается при создании
    # ─────────────────────────────────────────
    def perform_create(self, serializer):
        serializer.save(author=self.request.user)

    # ─────────────────────────────────────────
    # Вызывается при обновлении
    # ─────────────────────────────────────────
    def perform_update(self, serializer):
        instance = serializer.save()
        send_email_confirmation(instance)

    # ─────────────────────────────────────────
    # Вызывается при удалении
    # ─────────────────────────────────────────
    def perform_destroy(self, instance):
        instance.is_deleted = True
        instance.save()
```

---

## 🔹 Модуль 6. Переопределение ответа (своя структура JSON)

**Когда использовать:** нужно добавить поля в ответ (токены, сообщения).

```python
class RegisterView(generics.CreateAPIView):
    queryset = User.objects.all()
    serializer_class = UserRegistrationSerializer
    permission_classes = [permissions.AllowAny]

    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        user = serializer.save()

        # ─────────────────────────────────────────
        # Дополнительная логика
        # ─────────────────────────────────────────
        refresh = RefreshToken.for_user(user)

        # ─────────────────────────────────────────
        # Свой формат ответа
        # ─────────────────────────────────────────
        return Response({
            'user': UserProfileSerializer(user).data,
            'refresh': str(refresh),
            'access': str(refresh.access_token),
            'message': 'User registered successfully'
        }, status=status.HTTP_201_CREATED)
```

---

## 🔹 Модуль 7. Кастомный lookup (поиск по другому полю)

**Когда использовать:** объект ищется не по `pk`, а по `slug` или `email`.

```python
class PostDetailView(generics.RetrieveAPIView):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    # ─────────────────────────────────────────
    # Ищем по slug, а не по pk
    # ─────────────────────────────────────────
    lookup_field = 'slug'
```

---

## 🔹 Модуль 8. Пагинация

**Когда использовать:** длинные списки.

```python
class PostListView(generics.ListAPIView):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [permissions.AllowAny]

    # ─────────────────────────────────────────
    # Включаем пагинацию
    # ─────────────────────────────────────────
    pagination_class = PageNumberPagination
    # или свой класс:
    # pagination_class = CustomPagination
```

**Настройка в `settings.py`:**

```python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10
}
```

---

## 🔹 Модуль 9. Аутентификация (для APIView)

**Когда использовать:** не стандартный JWT, а Token или Session.

```python
from rest_framework import authentication


class CustomAuthView(generics.GenericAPIView):
    # ─────────────────────────────────────────
    # Кастомная аутентификация
    # ─────────────────────────────────────────
    authentication_classes = [authentication.TokenAuthentication]
    permission_classes = [permissions.IsAuthenticated]
```

**Варианты:**

- `[authentication.TokenAuthentication]` — токен в заголовке
- `[authentication.SessionAuthentication]` — сессия Django
- `[JWTAuthentication]` — JWT (по умолчанию, если настроен)

---

## 🔹 Модуль 10. Кастомные методы API (не CRUD)

**Когда использовать:** нестандартное действие (`POST /users/1/activate/`).

```python
from rest_framework.decorators import action
from rest_framework import viewsets


class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer

    # ─────────────────────────────────────────
    # POST /users/{id}/activate/
    # ─────────────────────────────────────────
    @action(detail=True, methods=['post'])
    def activate(self, request, pk=None):
        user = self.get_object()
        user.is_active = True
        user.save()
        return Response({'status': 'activated'})
```

---

## 🎯 Полный пример: модульный view

```python
# ─────────────────────────────────────────────
# ИМПОРТЫ
# ─────────────────────────────────────────────
from rest_framework import generics, permissions, status
from rest_framework.response import Response

from .models import Post
from .serializers import PostSerializer, PostCreateSerializer


# ─────────────────────────────────────────────
# КЛАСС: Список + создание
# ─────────────────────────────────────────────
class PostListCreateView(generics.ListCreateAPIView):
    """Список постов + создание нового"""

    # ─────────────────────────────────────────
    # QUERYSET (можно переопределить в get_queryset)
    # ─────────────────────────────────────────
    queryset = Post.objects.all()

    # ─────────────────────────────────────────
    # PERMISSIONS: чтение всем, запись авторизованным
    # ─────────────────────────────────────────
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]

    # ─────────────────────────────────────────
    # SERIALIZER: динамический по методу
    # ─────────────────────────────────────────
    def get_serializer_class(self):
        if self.request.method == 'POST':
            return PostCreateSerializer
        return PostSerializer

    # ─────────────────────────────────────────
    # QUERYSET: фильтрация
    # ─────────────────────────────────────────
    def get_queryset(self):
        queryset = Post.objects.filter(is_published=True)

        category = self.request.query_params.get('category')
        if category:
            queryset = queryset.filter(category=category)

        return queryset.order_by('-created_at')

    # ─────────────────────────────────────────
    # HOOK: привязка автора при создании
    # ─────────────────────────────────────────
    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
```

---

## 📚 Ссылки на документацию

| Тема                           | Django                                                                                                         | DRF                                                                                                                |
|--------------------------------|----------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| **Class-based views (основы)** | [Django Docs](https://docs.djangoproject.com/en/6.0/topics/class-based-views/)                                 | [DRF Views](https://www.django-rest-framework.org/api-guide/views/)                                                |
| **Generic views**              | [Django Built-in CBV API](https://docs.djangoproject.com/en/5.2/ref/class-based-views/)                        | [DRF Generic Views](https://www.django-rest-framework.org/api-guide/generic-views/)                                |
| **Mixins**                     | [Using mixins](https://docs.djangoproject.com/en/6.0/topics/class-based-views/mixins/)                         | [DRF Mixins (GitHub)](https://github.com/encode/django-rest-framework/blob/master/docs/api-guide/generic-views.md) |
| **Permissions**                | [Django Permissions](https://docs.djangoproject.com/en/5.2/topics/auth/default/#permissions-and-authorization) | [DRF Permissions](https://www.django-rest-framework.org/api-guide/permissions/)                                    |
| **Authentication**             | [Django Auth](https://docs.djangoproject.com/en/5.2/topics/auth/)                                              | [DRF Authentication](https://www.django-rest-framework.org/api-guide/authentication/)                              |
| **Pagination**                 | —                                                                                                              | [DRF Pagination](https://www.django-rest-framework.org/api-guide/pagination/)                                      |

**Полезный ресурс:** [Classy DRF](https://ccbv.co.uk/) — интерактивный справочник по всем CBV Django и DRF.

---

**Следующий этап:** URLs — связываем views с endpoint'ами через `path()` и `as_view()`.

## В моём случае итоговый код стал следующим

---

## 🔹 `RegisterView`

```python
class RegisterView(generics.CreateAPIView):
    """Регистрация нового пользователя"""
    queryset = User.objects.all()  # Берём всех пользователей из БД (нужно для внутренних механизмов DRF)
    serializer_class = UserRegistrationSerializer  # Какой сериализатор использовать
    permission_classes = [permissions.AllowAny]  # Доступ всем (регистрация открыта)

    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)  # Получаем данные в сериализаторе
        serializer.is_valid(raise_exception=True)  # Проверяем через validator нет ли ошибок
        user = serializer.save()  # Сохраняем данные обращаясь к объекту пользователя 'user'
        # через умный метод DRF serializer.save()

        refresh = RefreshToken.for_user(user)  # Записываем в переменную refresh наш токен,
        # который мы привязываем под нашего user

        # Далее идёт ответ словарь по формату который мы укажем и с токеном
        return Response({
            'user': UserProfileSerializer(user).data,  # Данные пользователя по сериализатору которые указали показывать
            'refresh': str(refresh),  # Знакомый нам токен который мы прописали
            'access': str(refresh.access_token),  # Встроенный метод из refresh используем
            # (refresh.access_token — уникальные токены для ОДНОГО пользователя конкретно)
            'message': 'User registered successfully'
        }, status=status.HTTP_201_CREATED)  # 201 — объект создан
```

---

## 🔹 `LoginView`

```python
class LoginView(generics.GenericAPIView):
    """Вход пользователя"""
    serializer_class = UserLoginSerializer  # Сериализатор для проверки email + пароля
    permission_classes = [permissions.AllowAny]  # Доступ всем (вход открыт)

    def post(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)  # Получаем данные в сериализаторе
        serializer.is_valid(raise_exception=True)  # Проверяем нет ли ошибок
        user = serializer.validated_data['user']  # Достаём user из проверенных данных
        # (сериализатор положил его туда в validate())

        login(request, user)  # Логиним пользователя в Django-сессии
        refresh = RefreshToken.for_user(user)  # Генерируем refresh-токен под user

        return Response({
            'user': UserProfileSerializer(user).data,  # Данные пользователя по сериализатору
            'refresh': str(refresh),  # Refresh-токен
            'access': str(refresh.access_token),  # Access-токен из refresh
            'message': 'User login successfully'
        }, status=status.HTTP_200_OK)  # 200 — успешный вход
```

---

## 🔹 `ProfileView`

```python
class ProfileView(generics.RetrieveUpdateAPIView):
    """Просмотр и обновление профиля"""
    serializer_class = UserProfileSerializer  # Сериализатор по умолчанию
    permission_classes = [permissions.IsAuthenticated]  # Только авторизованные

    def get_object(self):
        return self.request.user  # Возвращаем текущего пользователя (а не по pk из URL)

    def get_serializer_class(self):
        if self.request.method == 'PUT' or self.request.method == 'PATCH':  # Если обновление
            return UserUpdateSerializer  # Используем сериализатор обновления
        return UserProfileSerializer  # Иначе — сериализатор просмотра
```

---

## 🔹 `ChangePasswordView`

```python
class ChangePasswordView(generics.UpdateAPIView):
    """Смена пароля"""
    serializer_class = ChangePasswordSerializer  # Сериализатор смены пароля
    permission_classes = [permissions.IsAuthenticated]  # Только авторизованные

    def get_object(self):
        return self.request.user  # Меняем пароль текущему пользователю

    def update(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)  # Получаем данные в сериализаторе
        serializer.is_valid(raise_exception=True)  # Проверяем нет ли ошибок
        serializer.save()  # Сохраняем (внутри — set_password + save)

        return Response({
            'message': 'Password changed successfully'
        }, status=status.HTTP_200_OK)  # 200 — пароль изменён
```

---

## 🔹 `logout_view`

```python
@api_view(['POST'])  # Функциональный view, только POST
@permission_classes([permissions.IsAuthenticated])  # Только авторизованные
def logout_view(request):
    """Выход пользователя"""
    try:
        refresh_token = request.data.get('refresh_token')  # Достаём refresh-токен из тела запроса
        if refresh_token:
            token = RefreshToken(refresh_token)  # Создаём объект токена
            token.blacklist()  # Добавляем в blacklist (токен больше нельзя использовать)
        return Response({
            'message': 'Logout successful'
        }, status=status.HTTP_200_OK)  # 200 — успешный выход
    except Exception:
        return Response({
            'error': 'Invalid token'
        }, status=status.HTTP_400_BAD_REQUEST)  # 400 — токен невалидный
```

## Прописываем URLS для нашего приложения

```python
from django.urls import path
from rest_framework_simplejwt.views import TokenRefreshView

from . import views

urlpatterns = [
    path('register/', views.RegisterView.as_view(), name='register'),
    path('login/', views.LoginView.as_view(), name='login'),
    path('logout/', views.logout_view, name='logout'),
    path('profile/', views.ProfileView.as_view(), name='profile'),
    path('change-password/', views.ChangePasswordView.as_view(), name='change_password'),
    path('token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
]

```

# 📘 Django (DRF) + Vue — Этап: Основное приложение (Main)

---

## 🔹 Часть 1. Модель `Category`

```python
from django.db import models
from django.utils.text import slugify


class Category(models.Model):
    """Модель категории для постов блога"""
    name = models.CharField(max_length=100, unique=True)  # название категории (уникальное)
    slug = models.SlugField(max_length=100, unique=True, blank=True)  # URL-имя (заполнится автоматически)
    description = models.TextField(blank=True)  # описание (необязательное)
    created_at = models.DateTimeField(auto_now_add=True)  # дата создания (ставится 1 раз)

    class Meta:
        db_table = 'categories'  # имя таблицы в БД
        verbose_name = 'Category'  # человекочитаемое имя (ед. ч.)
        verbose_name_plural = 'Categories'  # человекочитаемое имя (мн. ч.)
        ordering = ['name']  # сортировка по умолчанию — по имени

    def __str__(self):
        return self.name  # строковое представление — имя категории

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.name)  # автогенерация slug из имени (если не задан)
        super().save(*args, **kwargs)
```

**Пример queryset:** `Category.objects.all()` → `<QuerySet [<Category: Tech>, <Category: Life>]>`

---

## 🔹 Часть 2. Менеджер `PostManager`

```python
class PostManager(models.Manager):
    """Менеджер для модели Post с дополнительными методами"""

    def published(self):
        return self.filter(status='published')  # только опубликованные посты

    def pinned_posts(self):
        """Закреплённые посты с активной подпиской автора"""
        return self.filter(
            pin_info__isnull=False,  # есть запись о закреплении
            pin_info__user__subscription__status='active',  # подписка активна
            pin_info__user__subscription__end_date__gt=models.functions.Now(),  # не истекла
            status='published'  # пост опубликован
        ).order_by('pin_info__pinned_at')  # сортировка по дате закрепления

    def with_subscription_info(self):
        """Оптимизация: подтягиваем связанные данные одним запросом"""
        return self.select_related(
            'author', 'author__subscription', 'category'  # JOIN по author, subscription, category
        ).prefetch_related('pin_info')  # отдельный запрос для pin_info
```

**Пример queryset:** `Post.objects.published()` → `<QuerySet [<Post: Django tips>, ...]>`

---

## 🔹 Часть 3. Модель `Post`

```python
class Post(models.Model):
    """Модель поста блога с поддержкой закрепления"""
    STATUS_CHOICES = [
        ('draft', 'Draft'),  # черновик
        ('published', 'Published'),  # опубликован
    ]

    title = models.CharField(max_length=200)  # заголовок поста
    slug = models.SlugField(max_length=200, unique=True, blank=True)  # URL-имя (автогенерация)
    content = models.TextField()  # содержимое поста
    image = models.ImageField(upload_to='posts/', blank=True, null=True)  # картинка (необязательна)
    category = models.ForeignKey(
        Category, on_delete=models.SET_NULL,  # при удалении категории — NULL
        null=True, blank=True, related_name='posts'  # обратная связь: category.posts
    )
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE,  # при удалении автора — удалить посты
        related_name='posts'  # обратная связь: author.posts
    )
    status = models.CharField(max_length=10, choices=STATUS_CHOICES, default='published')  # статус
    created_at = models.DateTimeField(auto_now_add=True)  # дата создания (1 раз)
    updated_at = models.DateTimeField(auto_now=True)  # дата обновления (каждый save)
    views_count = models.PositiveIntegerField(default=0)  # счётчик просмотров

    objects = PostManager()  # кастомный менеджер

    class Meta:
        db_table = 'posts'  # имя таблицы в БД
        verbose_name = 'Post'
        verbose_name_plural = 'Posts'
        ordering = ['-created_at']  # сортировка по умолчанию — новые сверху
        indexes = [  # индексы для быстрых запросов
            models.Index(fields=['-created_at']),  # по дате
            models.Index(fields=['status', '-created_at']),  # по статусу + дате
            models.Index(fields=['category', '-created_at']),  # по категории + дате
            models.Index(fields=['author', '-created_at']),  # по автору + дате
        ]

    def __str__(self):
        return self.title  # строковое представление — заголовок

    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.title)  # автогенерация slug из заголовка
        super().save(*args, **kwargs)

    def get_absolute_url(self):
        return reverse('post-detail', kwargs={'slug': self.slug})  # URL поста по slug

    @property
    def comments_count(self):
        """Количество активных комментариев"""
        return self.comments.filter(is_active=True).count()  # только активные

    @property
    def is_pinned(self):
        """Проверяет, закреплён ли пост"""
        return hasattr(self, 'pin_info') and self.pin_info is not None  # есть запись о закреплении

    def can_be_pinned_by(self, user):
        """Может ли пользователь закрепить этот пост"""
        if not user or not user.is_authenticated:  # не авторизован → нет
            return False
        if self.author != user:  # не автор → нет
            return False
        if self.status != 'published':  # не опубликован → нет
            return False
        if not hasattr(user, 'subscription') or not user.subscription.is_active:  # нет подписки → нет
            return False
        return True  # все условия выполнены

    def increment_views(self):
        """Увеличивает счётчик просмотров на 1"""
        self.views_count += 1
        self.save(update_fields=['views_count'])  # сохраняем только поле views_count

    def get_pinned_info(self):
        """Возвращает информацию о закреплении"""
        if self.is_pinned:
            return {
                'is_pinned': True,
                'pinned_at': self.pin_info.pinned_at,  # дата закрепления
                'pinned_by': {
                    'id': self.pin_info.user.id,
                    'username': self.pin_info.user.username,
                    'has_active_subscription': self.pin_info.user.subscription.is_active
                }
            }
        return {'is_pinned': False}  # не закреплён
```

**Пример queryset:** `Post.objects.pinned_posts()` → `<QuerySet [<Post: Django tips>]>`

---

## 🔹 Часть 4. Сериализаторы

### 📌 `CategorySerializer`

```python
class CategorySerializer(serializers.ModelSerializer):
    """Сериализатор для категорий"""
    posts_count = serializers.SerializerMethodField()  # вычисляемое поле: количество постов

    class Meta:
        model = Category
        fields = ['id', 'name', 'slug', 'description', 'posts_count', 'created_at']
        read_only_fields = ['slug', 'created_at']  # нельзя менять вручную

    def get_posts_count(self, obj):
        return obj.posts.filter(status='published').count()  # только опубликованные

    def create(self, validated_data):
        validated_data['slug'] = slugify(validated_data['name'])  # автогенерация slug
        return super().create(validated_data)
```

#### 🧪 Пример ответа (200 OK)

```json
{
  "id": 1,
  "name": "Tech",
  "slug": "tech",
  "description": "Технологии",
  "posts_count": 12,
  "created_at": "2026-10-03T12:00:00Z"
}
```

---

### 📌 `PostListSerializer` — список

```python
class PostListSerializer(serializers.ModelSerializer):
    """Сериализатор для списка постов"""
    author = serializers.StringRelatedField()  # имя автора строкой
    category = serializers.StringRelatedField()  # имя категории строкой
    comments_count = serializers.ReadOnlyField()  # берётся из @property модели
    is_pinned = serializers.ReadOnlyField()  # берётся из @property модели
    pinned_info = serializers.SerializerMethodField()  # детали закрепления

    class Meta:
        model = Post
        fields = [
            'id', 'title', 'slug', 'content', 'image', 'category',
            'author', 'status', 'created_at', 'updated_at',
            'views_count', 'comments_count', 'is_pinned', 'pinned_info'
        ]
        read_only_fields = ['slug', 'author', 'views_count']  # нельзя менять

    def get_pinned_info(self, obj):
        return obj.get_pinned_info()  # берём из метода модели

    def to_representation(self, instance):
        data = super().to_representation(instance)
        if len(data['content']) > 200:
            data['content'] = data['content'][:200] + '...'  # обрезаем контент для списка
        return data
```

---

### 📌 `PostDetailSerializer` — детально

```python
class PostDetailSerializer(serializers.ModelSerializer):
    """Сериализатор для детального просмотра поста"""
    author_info = serializers.SerializerMethodField()  # расширенные данные автора
    category_info = serializers.SerializerMethodField()  # расширенные данные категории
    comments_count = serializers.ReadOnlyField()  # из @property
    is_pinned = serializers.ReadOnlyField()  # из @property
    pinned_info = serializers.SerializerMethodField()  # детали закрепления
    can_pin = serializers.SerializerMethodField()  # может ли текущий user закрепить

    class Meta:
        model = Post
        fields = [
            'id', 'title', 'slug', 'content', 'image', 'category',
            'category_info', 'author', 'author_info', 'status',
            'created_at', 'updated_at', 'views_count', 'comments_count',
            'is_pinned', 'pinned_info', 'can_pin'
        ]
        read_only_fields = ['slug', 'author', 'views_count']

    def get_author_info(self, obj):
        author = obj.author
        return {
            'id': author.id,
            'username': author.username,
            'full_name': author.full_name,  # из @property модели User
            'avatar': author.avatar.url if author.avatar else None  # URL аватара
        }

    def get_category_info(self, obj):
        if obj.category:
            return {
                'id': obj.category.id,
                'name': obj.category.name,
                'slug': obj.category.slug,
            }
        return None  # если категории нет

    def get_pinned_info(self, obj):
        return obj.get_pinned_info()  # берём из метода модели

    def get_can_pin(self, obj):
        request = self.context.get('request')  # текущий запрос
        if not request or not request.user.is_authenticated:
            return False
        return obj.can_be_pinned_by(request.user)  # проверяем права
```

---

### 📌 `PostCreateUpdateSerializer` — создание/обновление

```python
class PostCreateUpdateSerializer(serializers.ModelSerializer):
    """Сериализатор для создания и обновления постов"""

    class Meta:
        model = Post
        fields = ['title', 'content', 'image', 'category', 'status']  # только эти поля

    def create(self, validated_data):
        validated_data['author'] = self.context['request'].user  # автор = текущий пользователь
        validated_data['slug'] = slugify(validated_data['title'])  # автогенерация slug
        return super().create(validated_data)

    def update(self, instance, validated_data):
        if 'title' in validated_data:
            validated_data['slug'] = slugify(validated_data['title'])  # пересоздаём slug
        return super().update(instance, validated_data)
```

---

## 🔹 Часть 5. Permissions

```python
from rest_framework import permissions


class IsAuthorOrReadOnly(permissions.BasePermission):
    """Разрешение: редактировать может только автор, читать — все"""

    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:  # GET, HEAD, OPTIONS — всем
            return True
        return obj.author == request.user  # запись — только автору
```

---

## 🔹 Часть 6. Views

### 📌 `CategoryListCreateView`

```python
class CategoryListCreateView(generics.ListCreateAPIView):
    """API endpoint для категорий"""
    queryset = Category.objects.all()  # все категории
    serializer_class = CategorySerializer  # какой сериализатор
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]  # чтение всем, запись — авториз.
    filter_backends = [filters.SearchFilter, filters.OrderingFilter]  # поиск + сортировка
    search_fields = ['name', 'description']  # по каким полям искать
    ordering_fields = ['name', 'created_at']  # по каким полям сортировать
    ordering = ['name']  # сортировка по умолчанию
```

---

### 📌 `CategoryDetailView`

```python
class CategoryDetailView(generics.RetrieveUpdateDestroyAPIView):
    """API endpoint для конкретной категории"""
    queryset = Category.objects.all()  # все категории
    serializer_class = CategorySerializer  # сериализатор
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]  # чтение всем
    lookup_field = 'slug'  # ищем по slug, а не по pk
```

---

### 📌 `PostListCreateView`

```python
class PostListCreateView(generics.ListCreateAPIView):
    """
    API endpoint для постов с поддержкой закреплённых постов.
    Закреплённые посты отображаются первыми.
    """
    serializer_class = PostListSerializer  # сериализатор по умолчанию
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]  # чтение всем
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_fields = ['category', 'author', 'status']  # фильтрация по полям
    search_fields = ['title', 'content']  # поиск по тексту
    ordering_fields = ['created_at', 'updated_at', 'views_count', 'title']  # сортировка
    ordering = ['-created_at']  # новые сверху

    def get_queryset(self):
        """Возвращает посты с учётом прав доступа"""
        queryset = Post.objects.select_related('author', 'category')  # оптимизация JOIN

        # Фильтрация по правам
        if not self.request.user.is_authenticated:
            queryset = queryset.filter(status='published')  # аноним — только опубликованные
        else:
            queryset = queryset.filter(
                Q(status='published') | Q(author=self.request.user)  # свои + опубликованные
            )

        # Если сортировка не задана — показываем закреплённые первыми
        ordering = self.request.query_params.get('ordering', '')
        show_pinned_first = not ordering or ordering in ['-created_at', 'created_at']

        if show_pinned_first:
            return Post.get_posts_for_feed().filter(
                Q(status='published') | (
                    Q(author=self.request.user) if self.request.user.is_authenticated else Q()
                )
            )
        return queryset

    def get_serializer_class(self):
        if self.request.method == 'POST':
            return PostCreateUpdateSerializer  # при создании — свой сериализатор
        return PostListSerializer  # при чтении — список

    def list(self, request, *args, **kwargs):
        response = super().list(request, *args, **kwargs)

        # Добавляем статистику закреплённых
        if hasattr(response, 'data') and 'results' in response.data:
            pinned_count = sum(1 for post in response.data['results'] if post.get('is_pinned', False))
            response.data['pinned_posts_count'] = pinned_count  # сколько закреплённых

        return response
```

---

### 📌 `PostDetailView`

```python
class PostDetailView(generics.RetrieveUpdateDestroyAPIView):
    """API endpoint для конкретного поста"""
    queryset = Post.objects.select_related('author', 'category')  # оптимизация
    serializer_class = PostDetailSerializer  # сериализатор по умолчанию
    permission_classes = [IsAuthorOrReadOnly]  # только автор может править
    lookup_field = 'slug'  # ищем по slug

    def get_serializer_class(self):
        if self.request.method in ['PUT', 'PATCH']:
            return PostCreateUpdateSerializer  # при обновлении — свой сериализатор
        return PostDetailSerializer  # при чтении — детальный

    def retrieve(self, request, *args, **kwargs):
        """Увеличивает счётчик просмотров при GET"""
        instance = self.get_object()
        if request.method == 'GET':
            instance.increment_views()  # +1 к просмотрам
        serializer = self.get_serializer(instance)
        return Response(serializer.data)
```

---

### 📌 `MyPostsView`

```python
class MyPostsView(generics.ListAPIView):
    """API endpoint для постов текущего пользователя"""
    serializer_class = PostListSerializer  # сериализатор списка
    permission_classes = [permissions.IsAuthenticated]  # только авторизованные
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_fields = ['category', 'status']  # фильтрация
    search_fields = ['title', 'content']  # поиск
    ordering_fields = ['created_at', 'updated_at', 'views_count', 'title']  # сортировка
    ordering = ['-created_at']  # новые сверху

    def get_queryset(self):
        return Post.objects.filter(
            author=self.request.user  # только посты текущего пользователя
        ).select_related('author', 'category')  # оптимизация
```

---

### 📌 Функциональные views

```python
@api_view(['GET'])  # только GET
@permission_classes([permissions.AllowAny])  # доступ всем
def post_by_category(request, category_slug):
    """Посты определённой категории"""
    category = get_object_or_404(Category, slug=category_slug)  # 404 если нет
    posts = Post.objects.with_subscription_info().filter(  # оптимизация
        category=category,
        status='published'
    )
    serializer = PostListSerializer(posts, many=True, context={'request': request})
    return Response({
        'category': CategorySerializer(category).data,  # данные категории
        'posts': serializer.data,  # список постов
        'pinned_posts_count': sum(1 for post in serializer.data if post.get('is_pinned', False))
    })


@api_view(['GET'])
@permission_classes([permissions.AllowAny])
def popular_posts(request):
    """10 самых популярных постов"""
    posts = Post.objects.with_subscription_info().filter(
        status='published'  # только опубликованные
    ).order_by('-views_count')[:10]  # топ-10 по просмотрам
    serializer = PostListSerializer(posts, many=True, context={'request': request})
    return Response(serializer.data)


@api_view(['GET'])
@permission_classes([permissions.AllowAny])
def recent_posts(request):
    """10 последних опубликованных постов"""
    posts = Post.objects.with_subscription_info().filter(
        status='published'
    ).order_by('-created_at')[:10]  # топ-10 по дате
    serializer = PostListSerializer(posts, many=True, context={'request': request})
    return Response(serializer.data)


@api_view(['GET'])
@permission_classes([permissions.AllowAny])
def pinned_posts_only(request):
    """Только закреплённые посты"""
    posts = Post.objects.pinned_posts()  # из кастомного менеджера
    serializer = PostListSerializer(posts, many=True, context={'request': request})
    return Response({
        'count': posts.count(),  # сколько всего
        'results': serializer.data  # список
    })


@api_view(['GET'])
@permission_classes([permissions.AllowAny])
def featured_posts(request):
    """Рекомендуемые посты для главной: закреплённые + популярные за неделю"""
    from django.utils import timezone
    from datetime import timedelta

    pinned_posts = Post.objects.pinned_posts()[:3]  # 3 закреплённых
    week_ago = timezone.now() - timedelta(days=7)  # неделя назад
    popular_posts = Post.objects.with_subscription_info().filter(
        status='published',
        created_at__gte=week_ago  # за последнюю неделю
    ).exclude(
        id__in=[post.id for post in pinned_posts]  # исключаем уже закреплённые
    ).order_by('-views_count')[:6]  # топ-6

    return Response({
        'pinned_posts': PostListSerializer(pinned_posts, many=True, context={'request': request}).data,
        'popular_posts': PostListSerializer(popular_posts, many=True, context={'request': request}).data,
        'total_pinned': Post.objects.pinned_posts().count()  # всего закреплённых
    })


@api_view(['POST'])  # только POST
@permission_classes([permissions.IsAuthenticated])  # только авторизованные
def toggle_post_pin_status(request, slug):
    """Переключает статус закрепления поста"""
    post = get_object_or_404(Post, slug=slug, author=request.user, status='published')

    # Проверяем подписку
    if not hasattr(request.user, 'subscription') or not request.user.subscription.is_active:
        return Response({
            'error': 'Active subscription required to pin posts'
        }, status=status.HTTP_403_FORBIDDEN)  # нет подписки → 403

    try:
        from apps.subscribe.models import PinnedPost

        if post.is_pinned:
            post.pin_info.delete()  # открепляем
            message, is_pinned = 'Post unpinned successfully', False
        else:
            if hasattr(request.user, 'pinned_post'):
                request.user.pinned_post.delete()  # удаляем старый закреп
            PinnedPost.objects.create(user=request.user, post=post)  # закрепляем новый
            message, is_pinned = 'Post pinned successfully', True

        return Response({
            'message': message,
            'is_pinned': is_pinned,
            'post': PostDetailSerializer(post, context={'request': request}).data
        })
    except Exception as e:
        return Response({'error': str(e)}, status=status.HTTP_400_BAD_REQUEST)  # 400 при ошибке
```

---

## 🔹 Часть 7. URLs

```python
from django.urls import path
from . import views

urlpatterns = [
    # Categories
    path('categories/', views.CategoryListCreateView.as_view(), name='category-list'),  # список + создание
    path('categories/<slug:slug>/', views.CategoryDetailView.as_view(), name='category-detail'),  # детали
    path('categories/<slug:category_slug>/posts/', views.post_by_category, name='posts-by-category'),  # посты категории

    # Posts
    path('', views.PostListCreateView.as_view(), name='post-list'),  # список + создание
    path('my-posts/', views.MyPostsView.as_view(), name='my-posts'),  # мои посты
    path('popular/', views.popular_posts, name='popular-posts'),  # популярные
    path('pinned/', views.pinned_posts_only, name='pinned-posts-only'),  # закреплённые
    path('featured/', views.featured_posts, name='featured-posts'),  # рекомендуемые
    path('recent/', views.recent_posts, name='recent-posts'),  # последние
    path('<slug:slug>/', views.PostDetailView.as_view(), name='post-detail'),  # детали поста
]
```

---

## ✅ Итог этапа

| Компонент                    | Назначение                  | Endpoint                                     |
|------------------------------|-----------------------------|----------------------------------------------|
| `Category` (модель)          | Категории блога             | —                                            |
| `Post` (модель)              | Посты блога                 | —                                            |
| `PostManager`                | Кастомные запросы           | —                                            |
| `CategorySerializer`         | Сериализация категорий      | —                                            |
| `PostListSerializer`         | Список постов               | —                                            |
| `PostDetailSerializer`       | Детальный пост              | —                                            |
| `PostCreateUpdateSerializer` | Создание/обновление         | —                                            |
| `IsAuthorOrReadOnly`         | Права доступа               | —                                            |
| `CategoryListCreateView`     | Список + создание категорий | `GET/POST /api/v1/categories/`               |
| `CategoryDetailView`         | Детали категории            | `GET/PUT/DELETE /api/v1/categories/<slug>/`  |
| `PostListCreateView`         | Список + создание постов    | `GET/POST /api/v1/posts/`                    |
| `PostDetailView`             | Детали поста                | `GET/PUT/DELETE /api/v1/posts/<slug>/`       |
| `MyPostsView`                | Мои посты                   | `GET /api/v1/posts/my-posts/`                |
| `post_by_category`           | Посты категории             | `GET /api/v1/posts/categories/<slug>/posts/` |
| `popular_posts`              | Популярные                  | `GET /api/v1/posts/popular/`                 |
| `recent_posts`               | Последние                   | `GET /api/v1/posts/recent/`                  |
| `pinned_posts_only`          | Только закреплённые         | `GET /api/v1/posts/pinned/`                  |
| `featured_posts`             | Рекомендуемые               | `GET /api/v1/posts/featured/`                |
| `toggle_post_pin_status`     | Закрепить/открепить         | `POST /api/v1/posts/<slug>/pin/`             |

# 📘 Django (DRF) + Vue — Справочник: библиотеки и методы (Main)

> Что именно использовалось в приложении `main` — по файлам и блокам.

---

## 🔹 1. Импорты в `views.py`

| Импорт                          | Откуда                          | Что даёт                                                                                    |
|---------------------------------|---------------------------------|---------------------------------------------------------------------------------------------|
| `generics`                      | `rest_framework`                | Готовые CBV (`ListAPIView`, `CreateAPIView`, `RetrieveUpdateDestroyAPIView`…)               |
| `permissions`                   | `rest_framework`                | Классы доступа (`AllowAny`, `IsAuthenticated`, `IsAuthenticatedOrReadOnly`)                 |
| `status`                        | `rest_framework`                | HTTP-коды (`HTTP_200_OK`, `HTTP_201_CREATED`, `HTTP_403_FORBIDDEN`, `HTTP_400_BAD_REQUEST`) |
| `filters`                       | `rest_framework`                | `SearchFilter`, `OrderingFilter` — поиск и сортировка                                       |
| `api_view`                      | `rest_framework.decorators`     | Декоратор для функциональных views                                                          |
| `permission_classes`            | `rest_framework.decorators`     | Декоратор для указания прав на функцию                                                      |
| `Response`                      | `rest_framework.response`       | Формирование JSON-ответа                                                                    |
| `DjangoFilterBackend`           | `django_filters.rest_framework` | Фильтрация по полям через `?field=value`                                                    |
| `Q`                             | `django.db.models`              | Логические условия `OR` / `AND` в запросах                                                  |
| `get_object_or_404`             | `django.shortcuts`              | Получить объект или вернуть 404                                                             |
| `Case`, `When`, `Value`         | `django.db.models`              | Условная аннотация (для сортировки закреплённых)                                            |
| `DateTimeField`, `BooleanField` | `django.db.models`              | Типы для `output_field`                                                                     |
| `timezone`                      | `django.utils`                  | Работа с датами (`timezone.now()`)                                                          |
| `timedelta`                     | `datetime`                      | Вычисления интервалов (неделя назад)                                                        |

---

## 🔹 2. Методы и атрибуты DRF

### В `generics.*`

| Метод / атрибут          | Что даёт                                              |
|--------------------------|-------------------------------------------------------|
| `queryset`               | Базовый набор объектов из БД                          |
| `serializer_class`       | Какой сериализатор использовать                       |
| `permission_classes`     | Кто имеет доступ                                      |
| `filter_backends`        | Какие фильтры применять                               |
| `filterset_fields`       | По каким полям фильтровать (`?category=1`)            |
| `search_fields`          | По каким полям искать (`?search=django`)              |
| `ordering_fields`        | По каким полям сортировать (`?ordering=-views_count`) |
| `ordering`               | Сортировка по умолчанию                               |
| `lookup_field`           | По какому полю искать объект (`slug` вместо `pk`)     |
| `get_queryset()`         | Динамический queryset (переопределяем)                |
| `get_serializer_class()` | Динамический сериализатор по методу                   |
| `get_object()`           | Получить один объект                                  |
| `list()`                 | Переопределение ответа для списка                     |
| `retrieve()`             | Переопределение ответа для одного объекта             |
| `perform_create()`       | Хук при создании                                      |
| `perform_update()`       | Хук при обновлении                                    |
| `perform_destroy()`      | Хук при удалении                                      |

### В `serializers.*`

| Метод / атрибут           | Что даёт                                       |
|---------------------------|------------------------------------------------|
| `SerializerMethodField()` | Поле, вычисляемое методом `get_<field>`        |
| `ReadOnlyField()`         | Поле только для чтения (из `@property` модели) |
| `StringRelatedField()`    | Строковое представление связанного объекта     |
| `to_representation()`     | Переопределение финального вывода              |
| `validate()`              | Общая валидация                                |
| `validate_<field>()`      | Валидация конкретного поля                     |
| `create()`                | Логика создания                                |
| `update()`                | Логика обновления                              |
| `context['request']`      | Доступ к запросу внутри сериализатора          |

### В `permissions.*`

| Класс / метод             | Что даёт                                 |
|---------------------------|------------------------------------------|
| `BasePermission`          | Базовый класс для своих правил           |
| `has_permission()`        | Доступ ко всему view                     |
| `has_object_permission()` | Доступ к конкретному объекту             |
| `SAFE_METHODS`            | `GET`, `HEAD`, `OPTIONS` — только чтение |

---

## 🔹 3. Методы моделей `Post` / `Category`

### `Post`

| Метод / свойство               | Что даёт                             |
|--------------------------------|--------------------------------------|
| `save()`                       | Автогенерация `slug` через `slugify` |
| `get_absolute_url()`           | URL объекта через `reverse()`        |
| `comments_count` (`@property`) | Количество активных комментариев     |
| `is_pinned` (`@property`)      | Закреплён ли пост                    |
| `can_be_pinned_by(user)`       | Может ли пользователь закрепить      |
| `increment_views()`            | +1 к `views_count`                   |
| `get_pinned_info()`            | Словарь с данными о закреплении      |
| `objects`                      | Кастомный `PostManager`              |

### `PostManager`

| Метод                      | Что даёт                                              |
|----------------------------|-------------------------------------------------------|
| `published()`              | Только `status='published'`                           |
| `pinned_posts()`           | Закреплённые с активной подпиской                     |
| `with_subscription_info()` | `select_related` + `prefetch_related` для оптимизации |

### `Category`

| Метод       | Что даёт             |
|-------------|----------------------|
| `save()`    | Автогенерация `slug` |
| `__str__()` | Возвращает `name`    |

---

## 🔹 4. Декораторы

| Декоратор                    | Что даёт                                  |
|------------------------------|-------------------------------------------|
| `@api_view(['GET'])`         | Превращает функцию в DRF-view             |
| `@permission_classes([...])` | Задаёт права для функционального view     |
| `@property`                  | Делает метод вычисляемым свойством модели |
| `@admin.register(Model)`     | Регистрирует модель в админке             |

---

## 🔹 5. Функции утилит

| Функция                         | Откуда              | Что даёт                                     |
|---------------------------------|---------------------|----------------------------------------------|
| `slugify()`                     | `django.utils.text` | Превращает `"Django Tips"` → `"django-tips"` |
| `reverse()`                     | `django.urls`       | Строит URL по имени маршрута                 |
| `get_object_or_404()`           | `django.shortcuts`  | Объект или 404                               |
| `timezone.now()`                | `django.utils`      | Текущее время с учётом TZ                    |
| `timedelta(days=7)`             | `datetime`          | Интервал «неделя»                            |
| `Q()`                           | `django.db.models`  | Логические условия в фильтрах                |
| `Case()` / `When()` / `Value()` | `django.db.models`  | Условная аннотация                           |
| `models.functions.Now()`        | `django.db.models`  | SQL `NOW()`                                  |

---

## 🔹 6. Библиотеки (внешние)

| Библиотека                      | Что даёт                                                  |
|---------------------------------|-----------------------------------------------------------|
| `djangorestframework`           | Основа API: views, serializers, permissions, status       |
| `djangorestframework-simplejwt` | JWT-токены: `RefreshToken`, `TokenRefreshView`, blacklist |
| `django-filter`                 | `DjangoFilterBackend` — фильтрация по полям               |
| `Pillow`                        | Работа с `ImageField` (аватары, картинки постов)          |

---

## 🔹 7. Настройки в `settings.py` (что точно надо упомянуть)

| Настройка                                                        | Что даёт                                              |
|------------------------------------------------------------------|-------------------------------------------------------|
| `AUTH_USER_MODEL = 'accounts.User'`                              | Кастомная модель пользователя                         |
| `INSTALLED_APPS` — `accounts` первым                             | Миграции `accounts` применяются раньше `auth`/`admin` |
| `REST_FRAMEWORK` → `DEFAULT_AUTHENTICATION_CLASSES`              | JWT по умолчанию                                      |
| `REST_FRAMEWORK` → `DEFAULT_PERMISSION_CLASSES`                  | Права по умолчанию                                    |
| `REST_FRAMEWORK` → `DEFAULT_PAGINATION_CLASS`, `PAGE_SIZE`       | Пагинация                                             |
| `SIMPLE_JWT` → `ACCESS_TOKEN_LIFETIME`, `REFRESH_TOKEN_LIFETIME` | Время жизни токенов                                   |
| `SIMPLE_JWT` → `BLACKLIST_AFTER_ROTATION`                        | Blacklist после ротации                               |
| `MEDIA_URL`, `MEDIA_ROOT`                                        | Загрузка аватаров и картинок постов                   |

---

## 🔹 8. Что стоит упомянуть в `INFO.md` отдельно

- **`select_related` / `prefetch_related`** — оптимизация запросов (убирает N+1).
- **`Q()`** — логика `OR` / `AND` в фильтрах.
- **`Case / When / Value`** — условная сортировка (закреплённые вверх).
- **`to_representation()`** — обрезка контента для списка.
- **`SerializerMethodField`** — вычисляемые поля (`can_pin`, `pinned_info`).
- **`@property` в модели** — `is_pinned`, `comments_count`.
- **Кастомный менеджер** `PostManager` — инкапсуляция частых запросов.
- **`lookup_field = 'slug'`** — ЧПУ-URL вместо `pk`.
- **`increment_views()`** — атомарное обновление счётчика.
- **`permission_classes`** — `IsAuthenticatedOrReadOnly` + свой `IsAuthorOrReadOnly`.
- **`filterset_fields` / `search_fields` / `ordering_fields`** — фильтрация, поиск, сортировка.
- **Декораторы `@api_view` + `@permission_classes`** — для функциональных views.

---

## ✅ Итог

| Категория      | Что упомянуть                                                                             |
|----------------|-------------------------------------------------------------------------------------------|
| **Импорты**    | `generics`, `permissions`, `filters`, `status`, `Q`, `Case/When`, `timezone`, `timedelta` |
| **DRF-методы** | `get_queryset`, `get_serializer_class`, `perform_create`, `to_representation`             |
| **Модели**     | `save()`, `@property`, кастомный менеджер                                                 |
| **Утилиты**    | `slugify`, `reverse`, `get_object_or_404`                                                 |
| **Библиотеки** | DRF, simplejwt, django-filter, Pillow                                                     |
| **Настройки**  | `AUTH_USER_MODEL`, `REST_FRAMEWORK`, `SIMPLE_JWT`, `MEDIA_*`                              |

# 📘 Django (DRF) + Vue — Этап: Комментарии (Comments)

---

## 🔹 Часть 1. Модель `Comment`

```python
from django.conf import settings
from django.db import models


class Comment(models.Model):
    """Модель комментария"""
    post = models.ForeignKey(
        'main.Post',  # ссылка на пост из приложения main
        on_delete=models.CASCADE,  # при удалении поста — удалить комментарии
        related_name='comments'  # обратная связь: post.comments
    )
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL,  # ссылка на кастомного User
        on_delete=models.CASCADE,  # при удалении автора — удалить комментарии
        related_name='comments'  # обратная связь: user.comments
    )
    parent = models.ForeignKey(
        'self',  # ссылка на самого себя (ответы)
        on_delete=models.CASCADE,
        null=True, blank=True,  # может быть пустым (основной комментарий)
        related_name='replies'  # обратная связь: comment.replies
    )
    content = models.TextField()  # текст комментария
    is_active = models.BooleanField(default=True)  # флаг активности (мягкое удаление)
    created_at = models.DateTimeField(auto_now_add=True)  # дата создания (1 раз)
    updated_at = models.DateTimeField(auto_now=True)  # дата обновления (каждый save)

    class Meta:
        db_table = 'comments'  # имя таблицы в БД
        verbose_name = 'Comment'
        verbose_name_plural = 'Comments'
        ordering = ['-created_at']  # сортировка по умолчанию — новые сверху
        indexes = [  # индексы для быстрых запросов
            models.Index(fields=['post', '-created_at']),  # по посту + дате
            models.Index(fields=['author', '-created_at']),  # по автору + дате
            models.Index(fields=['parent', '-created_at']),  # по родителю + дате
        ]

    def __str__(self):
        return f'Comment by {self.author.username} on {self.post.title}'  # строковое представление

    @property
    def replies_count(self):
        return self.replies.filter(is_active=True).count()  # количество активных ответов

    @property
    def is_reply(self):
        return self.parent is not None  # является ли комментарий ответом
```

**Пример queryset:** `Comment.objects.filter(is_active=True)` →
`<QuerySet [<Comment: Comment by ivan on Django tips>, ...]>`

---

## 🔹 Часть 2. Сериализаторы

### 📌 `CommentSerializer` — базовый

```python
class CommentSerializer(serializers.ModelSerializer):
    """Базовый сериализатор для комментариев"""
    author_info = serializers.SerializerMethodField()  # расширенные данные автора
    replies_count = serializers.ReadOnlyField()  # из @property модели
    is_reply = serializers.ReadOnlyField()  # из @property модели

    class Meta:
        model = Comment
        fields = [
            'id', 'content', 'author', 'author_info', 'parent',
            'is_active', 'replies_count', 'is_reply',
            'created_at', 'updated_at'
        ]
        read_only_fields = ['author', 'is_active']  # нельзя менять вручную

    def get_author_info(self, obj):
        return {
            'id': obj.author.id,
            'username': obj.author.username,
            'full_name': obj.author.full_name,  # из @property модели User
            'avatar': obj.author.avatar.url if obj.author.avatar else None  # URL аватара
        }
```

#### 🧪 Пример ответа (200 OK)

```json
{
  "id": 1,
  "content": "Отличный пост!",
  "author": 2,
  "author_info": {
    "id": 2,
    "username": "ivan",
    "full_name": "Иван Иванов",
    "avatar": "/media/avatars/ivan.png"
  },
  "parent": null,
  "is_active": true,
  "replies_count": 3,
  "is_reply": false,
  "created_at": "2026-10-05T12:00:00Z",
  "updated_at": "2026-10-05T12:00:00Z"
}
```

---

### 📌 `CommentCreateSerializer` — создание

```python
class CommentCreateSerializer(serializers.ModelSerializer):
    """Сериализатор для создания комментариев"""

    class Meta:
        model = Comment
        fields = ['post', 'parent', 'content']

    def validate_post(self, value):
        """Пост должен быть опубликован"""
        if not Post.objects.filter(id=value.id, status='published').exists():
            raise serializers.ValidationError('Post not found')
        return value

    def validate_parent(self, value):
        """Родительский комментарий должен быть из того же поста"""
        if value:
            post_data = self.initial_data.get('post')
            if post_data:
                if value.post.id != int(post_data):
                    raise serializers.ValidationError(
                        'Parent comment must belong to the same post.'
                    )
        return value

    def create(self, validated_data):
        validated_data['author'] = self.context['request'].user  # автор = текущий пользователь
        return super().create(validated_data)
```

#### 🧪 Пример данных на вход

```json
POST /api/v1/comments/
{
  "post": 1,
  "parent": null,
  "content": "Отличный пост!"
}
```

#### ❌ Пример ошибки (400 Bad Request)

```json
{
  "parent": [
    "Parent comment must belong to the same post."
  ]
}
```

---

### 📌 `CommentUpdateSerializer` — обновление

```python
class CommentUpdateSerializer(serializers.ModelSerializer):
    """Сериализатор для обновления комментариев"""

    class Meta:
        model = Comment
        fields = ['content']  # можно менять только текст
```

---

### 📌 `CommentDetailSerializer` — детально с ответами

```python
class CommentDetailSerializer(CommentSerializer):
    """Детальный сериализатор комментария с ответами"""
    replies = serializers.SerializerMethodField()  # вложенные ответы

    class Meta(CommentSerializer.Meta):
        fields = CommentSerializer.Meta.fields + ['replies']  # добавляем поле replies

    def get_replies(self, obj):
        if obj.parent is None:  # показываем ответы только для основных
            replies = obj.replies.filter(is_active=True).order_by('created_at')
            return CommentSerializer(replies, many=True, context=self.context).data
        return []  # для ответов — пустой список
```

---

## 🔹 Часть 3. Permissions

```python
from rest_framework import permissions


class IsAuthorOrReadOnly(permissions.BasePermission):
    """Разрешение: редактировать комментарий может только автор"""

    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:  # GET, HEAD, OPTIONS — всем
            return True
        return obj.author == request.user  # запись — только автору
```

---

## 🔹 Часть 4. Views

### 📌 `CommentListCreateView`

```python
class CommentListCreateView(generics.ListCreateAPIView):
    """Список и создание комментариев"""
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]  # чтение всем, запись — авториз.
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_fields = ['post', 'author', 'parent']  # фильтрация по полям
    search_fields = ['content']  # поиск по тексту
    ordering_fields = ['created_at', 'updated_at']  # сортировка
    ordering = ['-created_at']  # новые сверху

    def get_queryset(self):
        return Comment.objects.filter(is_active=True).select_related(  # только активные
            'author', 'post', 'parent'  # оптимизация JOIN
        )

    def get_serializer_class(self):
        if self.request.method == 'POST':
            return CommentCreateSerializer  # при создании — свой сериализатор
        return CommentSerializer  # при чтении — базовый
```

---

### 📌 `CommentDetailView`

```python
class CommentDetailView(generics.RetrieveUpdateDestroyAPIView):
    """Детальный просмотр, обновление и удаление комментария"""
    queryset = Comment.objects.filter(is_active=True).select_related('author', 'post')  # только активные
    serializer_class = CommentDetailSerializer  # сериализатор по умолчанию
    permission_classes = [IsAuthorOrReadOnly]  # править — только автор

    def get_serializer_class(self):
        if self.request.method in ['PUT', 'PATCH']:
            return CommentUpdateSerializer  # при обновлении — свой сериализатор
        return CommentDetailSerializer  # при чтении — детальный

    def perform_destroy(self, instance):
        instance.is_active = False  # мягкое удаление: помечаем неактивным
        instance.save()
```

---

### 📌 `MyCommentsView`

```python
class MyCommentsView(generics.ListAPIView):
    """Список комментариев текущего пользователя"""
    serializer_class = CommentSerializer  # базовый сериализатор
    permission_classes = [permissions.IsAuthenticated]  # только авторизованные
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_fields = ['post', 'parent', 'is_active']  # фильтрация
    search_fields = ['content']  # поиск
    ordering_fields = ['created_at', 'updated_at']  # сортировка
    ordering = ['-created_at']  # новые сверху

    def get_queryset(self):
        return Comment.objects.filter(author=self.request.user).select_related(  # только свои
            'post', 'parent'  # оптимизация
        )
```

---

### 📌 Функциональные views

```python
@api_view(['GET'])  # только GET
@permission_classes([permissions.AllowAny])  # доступ всем
def post_comments(request, post_id):
    """Получить комментарии к определённому посту"""
    post = get_object_or_404(Post, id=post_id, status='published')  # 404 если нет

    # Получаем только основные комментарии
    comments = Comment.objects.filter(
        post=post,
        parent=None,  # только корневые
        is_active=True  # только активные
    ).select_related('author').prefetch_related(  # оптимизация
        'replies__author'  # подтягиваем ответы и их авторов
    ).order_by('-created_at')  # новые сверху

    serializer = CommentDetailSerializer(comments, many=True, context={'request': request})
    return Response({
        'post': {
            'id': post.id,
            'title': post.title,
            'slug': post.slug
        },
        'comments': serializer.data,  # список комментариев с ответами
        'comments_count': post.comments.filter(is_active=True).count()  # всего активных
    })


@api_view(['GET'])
@permission_classes([permissions.AllowAny])
def comment_replies(request, comment_id):
    """Получить ответы на комментарий"""
    parent_comment = get_object_or_404(Comment, id=comment_id, is_active=True)  # 404 если нет

    replies = Comment.objects.filter(
        parent=parent_comment,  # только ответы этого комментария
        is_active=True  # только активные
    ).select_related('author').order_by('created_at')  # оптимизация + сортировка

    serializer = CommentSerializer(replies, many=True, context={'request': request})
    return Response({
        'parent_comment': CommentSerializer(parent_comment, context={'request': request}).data,
        'replies': serializer.data,  # список ответов
        'replies_count': replies.count()  # сколько всего
    })
```

---

## 🔹 Часть 5. URLs

```python
from django.urls import path

from . import views

urlpatterns = [
    path('', views.CommentListCreateView.as_view(), name='comment-list'),  # список + создание
    path('<int:pk>/', views.CommentDetailView.as_view(), name='comment-detail'),  # детали
    path('my-comments/', views.MyCommentsView.as_view(), name='my-comments'),  # мои комментарии
    path('post/<int:post_id>/', views.post_comments, name='post-comments'),  # к посту
    path('<int:comment_id>/replies/', views.comment_replies, name='comment-replies'),  # ответы
]
```

---

## 🔹 Часть 6. Admin

```python
@admin.register(Comment)
class CommentAdmin(admin.ModelAdmin):
    list_display = (
        'id', 'post_title', 'author', 'content_preview',
        'parent_comment', 'is_active', 'created_at'
    )
    list_filter = ('is_active', 'created_at', 'updated_at')
    search_fields = ('content', 'author__username', 'post__title')
    readonly_fields = ('created_at', 'updated_at')
    raw_id_fields = ('author', 'post', 'parent')  # автокомплит вместо выпадающего списка
    list_editable = ('is_active',)  # можно менять прямо из списка

    fieldsets = (
        (None, {'fields': ('post', 'author', 'parent', 'content')}),
        ('Status', {'fields': ('is_active',)}),
        ('Timestamps', {'fields': ('created_at', 'updated_at'), 'classes': ('collapse',)}),
    )

    def post_title(self, obj):
        return obj.post.title

    post_title.short_description = 'Post'

    def content_preview(self, obj):
        return obj.content[:50] + '...' if len(obj.content) > 50 else obj.content

    content_preview.short_description = 'Content Preview'

    def parent_comment(self, obj):
        if obj.parent:
            return f"Reply to: {obj.parent.content[:30]}..."
        return "Main comment"

    parent_comment.short_description = 'Parent'

    def get_queryset(self, request):
        return super().get_queryset(request).select_related('author', 'post', 'parent')  # оптимизация

    actions = ['make_active', 'make_inactive']  # массовые действия

    def make_active(self, request, queryset):
        updated = queryset.update(is_active=True)
        self.message_user(request, f'{updated} comments were marked as active.')

    make_active.short_description = "Mark selected comments as active"

    def make_inactive(self, request, queryset):
        updated = queryset.update(is_active=False)
        self.message_user(request, f'{updated} comments were marked as inactive.')

    make_inactive.short_description = "Mark selected comments as inactive"
```

---

## ✅ Итог этапа

| Компонент                 | Назначение                    | Endpoint (пример)                             |
|---------------------------|-------------------------------|-----------------------------------------------|
| `Comment` (модель)        | Комментарии к постам          | —                                             |
| `CommentSerializer`       | Базовый просмотр              | —                                             |
| `CommentCreateSerializer` | Создание                      | —                                             |
| `CommentUpdateSerializer` | Обновление                    | —                                             |
| `CommentDetailSerializer` | Детально с ответами           | —                                             |
| `IsAuthorOrReadOnly`      | Права доступа                 | —                                             |
| `CommentListCreateView`   | Список + создание             | `GET/POST /api/v1/comments/`                  |
| `CommentDetailView`       | Детали + редактирование + уд. | `GET/PUT/PATCH/DELETE /api/v1/comments/<pk>/` |
| `MyCommentsView`          | Мои комментарии               | `GET /api/v1/comments/my-comments/`           |
| `post_comments`           | Комментарии к посту           | `GET /api/v1/comments/post/<post_id>/`        |
| `comment_replies`         | Ответы на комментарий         | `GET /api/v1/comments/<comment_id>/replies/`  |

---

## 🔹 Что стоит упомянуть отдельно

- **`select_related` / `prefetch_related`** — оптимизация запросов (убирает N+1).
    - `select_related('author', 'post', 'parent')` — JOIN по FK.
    - `prefetch_related('replies__author')` — отдельные запросы для связанных.
- **`get_object_or_404()`** — получить объект или 404.
- **`SerializerMethodField()`** — вычисляемые поля (`author_info`, `replies`).
- **`ReadOnlyField()`** — поля из `@property` модели (`replies_count`, `is_reply`).
- **`perform_destroy()`** — мягкое удаление (`is_active = False`).
- **`filterset_fields` / `search_fields` / `ordering_fields`** — фильтрация, поиск, сортировка.
- **`IsAuthorOrReadOnly`** — свой permission: автор может править, все — читать.
- **`raw_id_fields`** — автокомплит для FK в админке.
- **`list_editable`** — редактирование поля прямо из списка в админке.
- **`actions`** — массовые действия (`make_active`, `make_inactive`).

---

## 📚 Ссылки на документацию

| Тема                    | Django                                                                                           | DRF                                                                             |
|-------------------------|--------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------|
| **Foreign Key**         | [Django FK](https://docs.djangoproject.com/en/5.2/ref/models/fields/#foreignkey)                 | —                                                                               |
| **Self-referencing FK** | [Self FK](https://docs.djangoproject.com/en/5.2/ref/models/fields/#foreignkey)                   | —                                                                               |
| **select_related**      | [select_related](https://docs.djangoproject.com/en/5.2/ref/models/querysets/#select-related)     | —                                                                               |
| **prefetch_related**    | [prefetch_related](https://docs.djangoproject.com/en/5.2/ref/models/querysets/#prefetch-related) | —                                                                               |
| **Serializers**         | —                                                                                                | [DRF Serializers](https://www.django-rest-framework.org/api-guide/serializers/) |
| **Permissions**         | —                                                                                                | [DRF Permissions](https://www.django-rest-framework.org/api-guide/permissions/) |
| **Filtering**           | —                                                                                                | [DRF Filtering](https://www.django-rest-framework.org/api-guide/filtering/)     |
| **Admin actions**       | [Admin actions](https://docs.djangoproject.com/en/5.2/ref/contrib/admin/actions/)                | —                                                                               |

**Следующий этап:** Subscribe — подписки и закрепление постов.