# TOPIC 14.5 — BUILDING A USER REGISTRATION API

**Course:** Django REST Framework — eCommerce Backend Development
**Phase:** Authentication and User Management 
**Technology:** Django + Django REST Framework 
**Project:** eCommerce Backend

---

## 1. INTRODUCTION

User registration is the process through which a new user creates an account on an application.

For our eCommerce application, customers should be able to register using:

- Username
- Email address
- Password

When a customer submits their registration details, our backend should:

1. Receive the registration information.
2. Validate the submitted data.
3. Check whether the username and email are already registered.
4. Validate the password.
5. Hash the password.
6. Create a new user in the database.
7. Return a successful response without exposing the password.

## 2. UNDERSTANDING THE REGISTRATION REQUEST

Our registration endpoint will be:

```http
POST /api/auth/register/
```

Example request body:

```json
{
    "username": "larry",
    "email": "larry@example.com",
    "password": "LarrySecure123!"
}
```

Expected successful response:

```json
{
    "id": 1,
    "username": "larry",
    "email": "larry@example.com"
}
```

Notice that the password is not included in the response.

Passwords should never be returned to API clients.

## 3. UNDERSTANDING THE COMPONENTS

We will use four main components.

| Component | Responsibility |
|---|---|
| Serializer | Validates incoming registration data and creates the user. |
| View | Handles the HTTP request and response. |
| URL | Connects the registration endpoint to its view. |
| User model | Stores the registered user in the database. |

We will use Django's existing User model rather than creating another user model.

This keeps our existing database configuration intact.

---

# 4. PRACTICAL IMPLEMENTATION

## STEP 1 — CREATE AN ACCOUNTS APP

We already have a `products` app.

Instead of putting authentication code inside the products app, we'll create a separate app called `accounts`.

From the project root, run:

```bash
python manage.py startapp accounts
```

This creates a new directory:

```text
accounts/
    migrations/
    __init__.py
    admin.py
    apps.py
    models.py
    tests.py
    views.py
```

The accounts app will contain our registration, login, logout, and profile functionality.

## STEP 2 — REGISTER THE ACCOUNTS APP

Open:

```text
ecommerce/settings.py
```

Locate `INSTALLED_APPS`.

Add `accounts`:

```python
INSTALLED_APPS = [
    # Django apps
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    # Third-party apps
    'rest_framework',

    # Local apps
    'products',
    'accounts',
]
```

Keep any other existing applications in your configuration.

**Why?**

Django needs to know that the accounts application is part of the project.

## STEP 3 — CREATE THE REGISTRATION SERIALIZER

Create a new file:

```text
accounts/serializers.py
```

Add:

```python
from django.contrib.auth import get_user_model
from django.contrib.auth.password_validation import validate_password
from rest_framework import serializers

User = get_user_model()


class UserRegistrationSerializer(serializers.ModelSerializer):

    password = serializers.CharField(
        write_only=True,
        required=True
    )

    class Meta:
        model = User
        fields = [
            'id',
            'username',
            'email',
            'password'
        ]

    def validate_email(self, value):
        if User.objects.filter(email__iexact=value).exists():
            raise serializers.ValidationError(
                "A user with this email already exists."
            )

        return value

    def validate_password(self, value):
        validate_password(value)
        return value

    def create(self, validated_data):
        return User.objects.create_user(**validated_data)
```

### Understanding the serializer

**A. `get_user_model()`**

```python
User = get_user_model()
```

Retrieves the user model configured for the Django project.

This is preferable to hardcoding the default User model.

**B. `write_only=True`**

```python
password = serializers.CharField(write_only=True)
```

Allows the password to be submitted but prevents it from appearing in serialized responses.

**C. `validate_email()`**

Checks whether the submitted email already exists.

We use `email__iexact` so that email comparisons are case-insensitive.

**D. `validate_password()`**

```python
validate_password(value)
```

Uses Django's configured password validators to check password strength and other rules.

These validators are configured in `settings.py` under `AUTH_PASSWORD_VALIDATORS`.

**E. `create()`**

```python
def create(self, validated_data):
    return User.objects.create_user(**validated_data)
```

Creates the user using Django's `create_user()` method.

This is important because `create_user()` automatically hashes the password.

We should not use `User.objects.create()` with a raw password.

## STEP 4 — CREATE THE REGISTRATION VIEW

Open:

```text
accounts/views.py
```

Add:

```python
from rest_framework.decorators import (
    api_view,
    permission_classes
)

from rest_framework.permissions import AllowAny
from rest_framework.response import Response
from rest_framework import status

from .serializers import UserRegistrationSerializer


@api_view(['POST'])
@permission_classes([AllowAny])
def register_user(request):

    serializer = UserRegistrationSerializer(
        data=request.data
    )

    if serializer.is_valid():
        user = serializer.save()

        return Response(
            UserRegistrationSerializer(user).data,
            status=status.HTTP_201_CREATED
        )

    return Response(
        serializer.errors,
        status=status.HTTP_400_BAD_REQUEST
    )
```

### Understanding the view

**A. HTTP method**

```python
@api_view(['POST'])
```

Registration creates a new resource, so we use POST.

**B. Public access**

```python
@permission_classes([AllowAny])
```

Anyone should be able to register, including users who haven't logged in.

**C. Serializer initialization**

```python
serializer = UserRegistrationSerializer(
    data=request.data
)
```

Receives the data submitted by the client.

**D. Validation**

```python
if serializer.is_valid():
```

Checks whether the submitted information satisfies the serializer's validation rules.

**E. Saving**

```python
user = serializer.save()
```

Calls the serializer's `create()` method and creates the database record.

**F. Successful response**

```python
status=status.HTTP_201_CREATED
```

HTTP 201 indicates that a new resource was successfully created.

**G. Validation errors**

```python
status=status.HTTP_400_BAD_REQUEST
```

Returns validation errors when the submitted information is invalid.

## STEP 5 — CREATE THE ACCOUNTS URL

Create:

```text
accounts/urls.py
```

Add:

```python
from django.urls import path
from .views import register_user

urlpatterns = [
    path('register/', register_user, name='register'),
]
```

This connects the `register/` route to our registration view.

## STEP 6 — CONNECT THE ACCOUNTS URL TO THE MAIN PROJECT

Open:

```text
ecommerce/urls.py
```

Update it to include the accounts application:

```python
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path('admin/', admin.site.urls),

    path('api/', include('products.urls')),

    path('api/auth/', include('accounts.urls')),

    path('api-auth/', include('rest_framework.urls')),
]
```

Our registration endpoint is now:

```text
http://127.0.0.1:8000/api/auth/register/
```

---

# 5. TESTING THE REGISTRATION API

Start your development server:

```bash
python manage.py runserver
```

Open:

```text
http://127.0.0.1:8000/api/auth/register/
```

Select POST in the browsable API and submit:

```json
{
    "username": "larry",
    "email": "larry@example.com",
    "password": "LarrySecure123!"
}
```

Expected response:

```json
{
    "id": 1,
    "username": "larry",
    "email": "larry@example.com"
}
```

Your ID may be different depending on the existing database records.

## 6. TESTING VALIDATION

We should test more than successful registration.

### Test 1 — Duplicate username

Submit a username that already exists.

Expected result:

```json
{
    "username": [
        "A user with that username already exists."
    ]
}
```

The exact wording may vary by DRF version.

### Test 2 — Duplicate email

Submit an email address that already exists.

Expected result:

```json
{
    "email": [
        "A user with this email already exists."
    ]
}
```

### Test 3 — Weak password

Try a weak password such as:

```text
123
```

Expected result: A password validation error if the corresponding validators are configured.

### Test 4 — Missing username

Submit a request without a username.

Expected result: HTTP 400 Bad Request.

### Test 5 — Missing password

Submit a request without a password.

Expected result: HTTP 400 Bad Request.

---

# 7. VERIFY THE PASSWORD IN THE DATABASE

After registering a user, open the Django shell:

```bash
python manage.py shell
```

Import the user model:

```python
from django.contrib.auth import get_user_model

User = get_user_model()
```

Retrieve the user:

```python
user = User.objects.get(username="larry")
```

Inspect the password:

```python
user.password
```

You should see a hashed password, not the original password.

Verify it:

```python
user.check_password("LarrySecure123!")
```

Expected:

```python
True
```

Try an incorrect password:

```python
user.check_password("wrongpassword")
```

Expected:

```python
False
```

This confirms that our registration endpoint uses Django's password hashing mechanism correctly.

---

# 8. IMPORTANT TAKEAWAYS

1. Registration creates a new user account.
2. Serializers validate incoming registration data.
3. `write_only=True` prevents passwords from appearing in API responses.
4. `validate_password()` applies Django's configured password validators.
5. `create_user()` automatically hashes passwords.
6. `AllowAny` allows unauthenticated users to register.
7. HTTP 201 indicates successful resource creation.
8. HTTP 400 indicates invalid submitted data.
9. A successful registration response should never expose the user's password.

## NEXT TOPIC

**Topic 14.6 — Building the Login API**

We will implement:

- Login serializer.
- Credential validation.
- Username/email login strategy.
- Authentication using Django's authentication system.
- Login endpoint.
- Successful and unsuccessful login responses.
- Testing with registered accounts.

After that, we will progress to JWT authentication and token management.
