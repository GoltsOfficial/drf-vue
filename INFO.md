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

**Следующий этап:** Views + URLs для сериализаторов, затем JWT-настройка (`djangorestframework-simplejwt`) и Vue-часть.