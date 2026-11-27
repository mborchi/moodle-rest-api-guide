# 1. Cómo conseguir el token de Moodle (`wstoken`)

Cuando un centro usa **Google Workspace / SSO + verificación en dos pasos (2FA)**, automatizar el login clásico por formulario y cookies se rompe en cada expiración o reto 2FA. Moodle incluye el endpoint oficial de la app móvil (`/admin/tool/mobile/launch.php`) que te permite autenticarte una vez en tu navegador normal y entrega un `wstoken` persistente para usar en scripts y APIs.

---

## Paso 1 · Abrir la URL de lanzamiento móvil

Inicia sesión con tu cuenta en el Moodle de tu centro y abre esta URL en el navegador (sustituye `<tu-moodle>` por el dominio de tu plataforma):

```text
https://<tu-moodle>/admin/tool/mobile/launch.php?service=moodle_mobile_app&passport=moodle_client_1
```

- Si ya tienes la sesión abierta, Moodle redirigirá al instante.
- Si no, te pedirá entrar (Google SSO / 2FA incluido) y después redirigirá automáticamente a una URL con esquema personalizado:

```text
moodlemobile://token=<CADENA_BASE64>
```

---

## Paso 2 · Capturar el `moodlemobile://token=...`

Como los navegadores de escritorio no tienen registrada la app móvil para abrir `moodlemobile://`, tienes dos formas rápidas de copiarlo:

### Opción A · Barra de direcciones
Algunos navegadores dejan visible el enlace `moodlemobile://token=...` en la barra superior o en el diálogo de confirmación *"¿Abrir aplicación externa?"*. Cópialo entero.

### Opción B · DevTools (`F12`) cuando la página se queda en blanco
1. Abre **Herramientas de desarrollo** (`F12`) antes de pulsar la URL y entra en la pestaña **Red / Network**.
2. Marca **Preservar registro / Preserve log**.
3. Abre la URL de `launch.php`.
4. Haz clic en la petición `launch.php` y revisa la cabecera de respuesta **`Location`** (o el error de redirección en consola): ahí aparece completo el `moodlemobile://token=...`.

---

## Paso 3 · Por qué hay que decodificar el token

Lo que viene después de `token=` **no es el `wstoken` directo**, sino una cadena codificada en **Base64** que Moodle empaqueta separada por triple dos puntos (`:::`):

```text
<prefijo_o_firma>:::<wstoken>[:::<privatetoken>]
```

- Si envías ese Base64 tal cual a `server.php`, Moodle devolverá `invalidtoken`.
- El `wstoken` válido que espera la API es únicamente el **segundo bloque** (`parts[1]`): normalmente una cadena hexadecimal de **32 caracteres**.
- En sitios HTTPS, Moodle puede incluir un tercer bloque opcional (`privatetoken`, `parts[2]`) usado para autologin web sin contraseña. Descártalo y **no** lo guardes ni lo imprimas si solo necesitas la API REST.

---

## Paso 4 · Decodificarlo

### En Python (cualquier script)

```python
import base64

raw = "moodlemobile://token=<CADENA_BASE64>"
b64 = raw.split("token=", 1)[1].split("&", 1)[0].strip()
b64 += "=" * (-len(b64) % 4)
decoded = base64.b64decode(b64).decode("utf-8")

wstoken = decoded.split(":::")[1].strip()
print(wstoken)
```

### En terminal Linux (`bash`)

```bash
RAW="<CADENA_BASE64>"
RAW="${RAW#moodlemobile://token=}"
RAW="${RAW%%&*}"
MOD=$((${#RAW} % 4))
[ "$MOD" -gt 0 ] && RAW="$RAW$(printf '%.*s' $((4 - MOD)) '====')"
printf '%s' "$RAW" | base64 -d | awk -F':::' '{print $2}'
```

---

## Paso 5 · Verificar que funciona

Prueba tu `wstoken` llamando a `core_webservice_get_site_info` (léelo con `read -rs` para que no quede guardado en el historial de la shell):

```bash
read -rs -p "wstoken: " WSTOKEN && echo
curl -sS -X POST "https://<tu-moodle>/webservice/rest/server.php" \
  -d "wstoken=$WSTOKEN" \
  -d "wsfunction=core_webservice_get_site_info" \
  -d "moodlewsrestformat=json"
```

Si todo está bien, devolverá un JSON con tu `userid`, `fullname` y `sitename`.

---

## Caducidad y renovación

El `wstoken` suele durar meses, pero puede caducar o revocarse si cambias contraseña o el administrador limpia tokens móviles. En ese caso `server.php` devuelve `errorcode: "invalidtoken"` (o `"tokenexpired"`): vuelve al Paso 1 para generar uno nuevo.
