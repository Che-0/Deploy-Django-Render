# 🚀 Guía de Despliegue de Django en Render

Guía técnica paso a paso para desplegar aplicaciones Django en la nube utilizando **Render**, basada en problemas reales de despliegue, errores comunes de configuración WSGI/estáticos y soluciones directas.

---

## 📋 Tabla de Contenidos
1. [Requisitos Previos](#1-requisitos-previos)
2. [Estructura del Repositorio](#2-estructura-del-repositorio)
3. [Configuración del Código](#3-configuración-del-código)
   - [requirements.txt](#dependencias-requirementstxt)
   - [settings.py](#ajustes-en-settingspy)
   - [urls.py](#enrutamiento-en-urlspy)
4. [Sembrado Automático de Usuarios (Seed Script)](#4-sembrado-automático-de-usuarios-create_adminpy)
5. [Configuración en el Panel de Render](#5-configuración-en-el-panel-de-render)
6. [Bitácora de Errores Comunes y Soluciones (Troubleshooting)](#6-bitácora-de-errores-comunes-troubleshooting)

---

## 1. Requisitos Previos

- Repositorio del proyecto subido a **GitHub**.
- Cuenta creada en [Render](https://render.com/).
- Base de datos relacional (PostgreSQL en Render o SQLite para pruebas temporales).

---

## 2. Estructura del Repositorio

Asegúrate de que la raíz de tu repositorio mantenga el archivo `manage.py` accesible:

```text
mi-repositorio-django/
├── manage.py
├── create_admin.py            # Script opcional para crear usuarios sin Shell
├── requirements.txt           # Dependencias para producción
├── .gitignore
├── mi_app/
│   ├── models.py
│   ├── views.py
│   └── ...
└── config_proyecto/           # Carpeta donde se aloja settings.py y wsgi.py
    ├── __init__.py
    ├── settings.py
    ├── urls.py
    └── wsgi.py
```

> **Nota:** Si `manage.py` se encuentra dentro de una subcarpeta (por ejemplo `repo/backend/manage.py`), deberás configurar el campo **Root Directory** en Render con el valor `backend`.

---

## 3. Configuración del Código

### Dependencias (`requirements.txt`)
Añade las dependencias esenciales de producción:

```text
Django>=5.0
gunicorn>=21.2.0
dj-database-url>=0.5.0
psycopg2-binary>=2.9.9
python-dotenv>=1.0.0
whitenoise>=6.6.0
```

---

### Ajustes en `settings.py`

#### A. Archivos Estáticos (WhiteNoise)
Evita el fallo `ImproperlyConfigured: You're using the staticfiles app without having set the STATIC_ROOT`.

```python
import os
from pathlib import Path
import dj_database_url

BASE_DIR = Path(__file__).resolve().parent.parent

# 1. Habilitar WhiteNoise en Middleware (inmediatamente después de SecurityMiddleware)
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',
    # ... demás middlewares
]

# 2. Configurar rutas de archivos estáticos
STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
```

#### B. Hosts Permitidos y Proxy SSL (Render)
Evita errores `400 Bad Request` por bloqueo de proxy o validación de host:

```python
# Permitir localhost y cualquier subdominio emitido por Render
ALLOWED_HOSTS = ['localhost', '127.0.0.1', '.onrender.com']

# Indicar a Django que el proxy de Render gestiona la terminación HTTPS
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')

# Asegurar cookies de sesión sobre HTTPS
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```

#### C. Base de Datos Dinámica
```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}

# Sobrescribir con PostgreSQL si existe DATABASE_URL en el entorno
database_url = os.environ.get('DATABASE_URL')
if database_url:
    DATABASES['default'] = dj_database_url.config(conn_max_age=600)
```

---

### Enrutamiento en `urls.py`

Para evitar errores `404 Not Found` en la raíz (`/`), asegura una ruta por defecto o redirección:

```python
from django.contrib import admin
from django.urls import path, include
from django.views.generic import RedirectView

urlpatterns = [
    path('admin/', admin.site.urls),
    # Redirige la raíz al módulo principal si no tienes vista en ''
    path('', RedirectView.as_view(url='/inicio/')), 
    path('inicio/', include('mi_app.urls')),
]
```

---

## 4. Sembrado Automático de Usuarios (`create_admin.py`)

Crea un archivo llamado `create_admin.py` en la raíz de tu proyecto (al mismo nivel que `manage.py`). Este script permite sembrar usuarios de forma no interactiva e idempotente (no duplica registros en despliegues posteriores).

```python
import os
import json
import sys
import django

# Ajusta 'config_proyecto.settings' al nombre real de tu carpeta de configuración
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config_proyecto.settings')
django.setup()

from django.contrib.auth import get_user_model

User = get_user_model()

def create_user_if_missing(user_data):
    username = user_data.get('username')
    password = user_data.get('password')

    if not username:
        print("[-] Se omitió un usuario sin username.")
        return

    if User.objects.filter(username=username).exists():
        print(f"[*] El usuario '{username}' ya existe.")
        return

    if not password:
        print(f"[-] Se omitió el usuario '{username}' porque no tiene contraseña.")
        return

    email = user_data.get('email', '')
    is_superuser = bool(user_data.get('is_superuser', False))

    if is_superuser:
        User.objects.create_superuser(
            username=username,
            email=email,
            password=password
        )
        print(f"[+] Superusuario '{username}' creado exitosamente.")
    else:
        # Usa create_user para garantizar el hashing PBKDF2 de la contraseña
        User.objects.create_user(
            username=username,
            email=email,
            password=password
        )
        print(f"[+] Usuario '{username}' creado exitosamente.")

# 1. Usuario Admin Principal
seed_users = [
    {
        'username': os.environ.get('DJANGO_SUPERUSER_USERNAME', 'admin'),
        'email': os.environ.get('DJANGO_SUPERUSER_EMAIL', 'admin@example.com'),
        'password': os.environ.get('DJANGO_SUPERUSER_PASSWORD'),
        'is_superuser': True,
    }
]

# 2. Carga opcional de usuarios secundarios vía JSON
extra_users = os.environ.get('DJANGO_SEED_USERS', '')
if extra_users.strip():
    try:
        parsed_users = json.loads(extra_users)
        if isinstance(parsed_users, list):
            seed_users.extend(parsed_users)
    except json.JSONDecodeError as exc:
        print(f"[!] Error al parsear DJANGO_SEED_USERS: {exc}")
        sys.exit(1)

for user_data in seed_users:
    create_user_if_missing(user_data)
```

---

## 5. Configuración en el Panel de Render

1. Entra a Render y selecciona **New +** $\rightarrow$ **Web Service**.
2. Conecta tu repositorio de GitHub.
3. Completa los siguientes parámetros:

| Parámetro | Valor |
| :--- | :--- |
| **Environment** | `Python 3` |
| **Branch** | `main` (o tu rama de producción) |
| **Build Command** | `pip install -r requirements.txt && python manage.py collectstatic --noinput && python manage.py migrate && python create_admin.py` |
| **Start Command** | `gunicorn config_proyecto.wsgi:application` |

> Reemplaza `config_proyecto` por el nombre de la carpeta interna donde se encuentra tu archivo `wsgi.py`.

### Variables de Entorno en Render
En la pestaña **Environment** agrega:

- `SECRET_KEY`: *(Cadena larga y aleatoria)*
- `DEBUG`: `False`
- `DJANGO_SUPERUSER_PASSWORD`: *(Contraseña para el superusuario inicial)*
- `DJANGO_SUPERUSER_USERNAME`: `admin` (opcional)
- `DJANGO_SEED_USERS`: *(Opcional - array JSON en una sola línea)*  
  `[{"username":"cliente1","password":"Password123!","email":"c1@test.com"}]`

---

## 6. Bitácora de Errores Comunes (Troubleshooting)

### Error 1: `ImproperlyConfigured: You're using the staticfiles app without having set the STATIC_ROOT`
- **Causa:** `python manage.py collectstatic` no sabe a qué carpeta exportar los archivos.
- **Solución:** Agregar `STATIC_ROOT = BASE_DIR / 'staticfiles'` en `settings.py`.

### Error 2: `bash: line 1: gunicorn: command not found`
- **Causa:** El paquete `gunicorn` no está instalado en el entorno de Render.
- **Solución:** Agregar `gunicorn` al archivo `requirements.txt`, hacer commit y volver a desplegar.

### Error 3: `ModuleNotFoundError: No module named '...'`
- **Causa:** El comando de inicio (`Start Command`) no apunta a la carpeta correcta de `wsgi.py`.
- **Solución:** Si tu archivo está en `mysite/wsgi.py`, el comando debe ser:
  ```bash
  gunicorn mysite.wsgi:application
  ```

### Error 4: `HTTP 400 Bad Request` en los logs
- **Causa:** Django rechaza la cabecera `Host` porque no está listada en `ALLOWED_HOSTS`.
- **Solución:** Asegúrate de incluir el comodín de Render en `settings.py`:
  ```python
  ALLOWED_HOSTS = ['localhost', '127.0.0.1', '.onrender.com']
  ```

### Error 5: Bucle infinito de redirección al intentar iniciar sesión
- **Causas Frecuentes:**
  1. **Contraseña en texto plano:** Si creaste usuarios usando `User.objects.create(...)` en lugar de `User.objects.create_user(...)`, Django no aplicó hash a la contraseña y `authenticate()` fallará silenciosamente.
  2. **Falta de cabecera HTTPS detrás del proxy:** Sin `SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')`, la cookie de sesión no viaja adecuadamente entre el proxy de Render y Django.
