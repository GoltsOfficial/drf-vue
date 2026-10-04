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