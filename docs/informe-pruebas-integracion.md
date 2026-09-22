# Informe de pruebas de integración — blacklist_email

**Repositorio:** [https://github.com/proyectos-universidad-2026/blacklist_app](https://github.com/proyectos-universidad-2026/blacklist_app)

Este documento describe el paso a paso para cubrir las capacidades del servicio `blacklist_email` con una colección Postman, ejecutarla en local con Newman y automatizarla en GitHub Actions.

## 1. Capacidades del servicio

El API Flask vive en `blacklist_app/src/routes/blacklist_router.py` y expone tres operaciones:

| Método | Ruta | Auth Bearer | Descripción |
| --- | --- | --- | --- |
| `GET` | `/blacklists/ping` | No | Health check. Responde `{"message": "pong"}`. |
| `POST` | `/blacklists` | Sí | Agrega un email a la lista negra. |
| `GET` | `/blacklists/<email>` | Sí | Consulta si el email está en la lista negra. |

Códigos de respuesta cubiertos en la colección:

- `200` consulta y ping
- `201` alta exitosa
- `400` body inválido o email mal formado
- `401` sin token o token inválido
- `409` email duplicado

## 2. Crear y exportar la colección Postman

1. Se inventariaron los endpoints reales del código (no se inventaron rutas).
2. Se armó la colección **blacklist_email** con carpetas:
   - Health check
   - Autenticación
   - Agregar email a blacklist
   - Consultar email en blacklist
3. Cada request incluye tests de Newman (`pm.test`) sobre status HTTP y cuerpo JSON.
4. Un script de pre-request genera un email único por corrida (`newman-<timestamp>@example.com`) para no chocar con el `409` de corridas anteriores.
5. La colección y el environment se exportaron a JSON en el repositorio:

- `postman/blacklist_email.postman_collection.json`
- `postman/blacklist_email.postman_environment.json`

Variables del environment:

- `baseUrl`: `http://127.0.0.1:5001`
- `bearerToken`: token esperado por `BEARER_TOKEN`
- `appUuid`: UUID válido para el alta
- `testEmail` / `unknownEmail`: se regeneran en cada corrida

## 3. Ejecutar Newman en local

Requisitos: Node.js/npm, Python 3.10+ y las dependencias del servicio.

```bash
npm install -g newman
pip install flask flask-sqlalchemy flask-cors marshmallow python-dotenv
```

Levantar el servicio (SQLite, sin Postgres) desde `blacklist_app/`:

```powershell
$env:DATABASE_URL = "sqlite:///local_newman.db"
$env:BEARER_TOKEN = "test-token"
python -c "from src.main import app; app.run(host='127.0.0.1', port=5001)"
```

En otra terminal, desde la raíz del repo:

```bash
newman run postman/blacklist_email.postman_collection.json -e postman/blacklist_email.postman_environment.json --env-var "baseUrl=http://127.0.0.1:5001" --env-var "bearerToken=test-token"
```

Newman debe reportar las requests en verde (assertions de status y payload).

## 4. Job de GitHub Actions `pruebas_integracion`

Archivo: `.github/workflows/pruebas_integracion.yml`

- Se dispara en `push` y `pull_request` contra `master`.
- El job se llama **`pruebas_integracion`**.
- Instala Node 20 y Newman (`npm install -g newman`).
- Instala Python 3.11 y las dependencias mínimas del API.
- Levanta el servicio con Gunicorn en `127.0.0.1:5001` usando SQLite y `BEARER_TOKEN=test-token`.
- Espera a que `GET /blacklists/ping` responda.
- Ejecuta la colección con Newman.

Para ver el job en GitHub hay que subir estos archivos al remoto (`git add`, `commit` y `push`).

## 5. Archivos agregados o modificados

| Archivo | Propósito |
| --- | --- |
| `postman/blacklist_email.postman_collection.json` | Colección exportada |
| `postman/blacklist_email.postman_environment.json` | Environment local / CI |
| `.github/workflows/pruebas_integracion.yml` | Job `pruebas_integracion` |
| `docs/informe-pruebas-integracion.md` | Este informe |
| `.gitignore` | Ignora archivos `*.db` de SQLite usados en las pruebas |

## 6. Cómo verificar el resultado

1. Correr Newman en local (sección 3) y confirmar que todas las assertions pasan.
2. Hacer push a `master` (o abrir un PR) y revisar en GitHub: **Actions → Pruebas de integración → pruebas_integracion**.
3. Abrir el repositorio: [https://github.com/proyectos-universidad-2026/blacklist_app](https://github.com/proyectos-universidad-2026/blacklist_app)
