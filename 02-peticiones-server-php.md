# 2. Cómo hacer peticiones a `/webservice/rest/server.php`

Todas las consultas REST de Moodle pasan por el mismo endpoint único:

```text
https://<tu-moodle>/webservice/rest/server.php
```

---

## Formato de petición recomendado (`POST`)

Aunque Moodle acepta `GET`, **usa `POST` con `application/x-www-form-urlencoded`** para que tu `wstoken` vaya en el cuerpo y no quede registrado en historiales ni logs de proxies como parte de la URL.

### Parámetros obligatorios en cada llamada

| Parámetro | Valor | Descripción |
|---|---|---|
| `wstoken` | `<hex_32_chars>` | Token decodificado (ver [01-conseguir-token.md](01-conseguir-token.md)) |
| `wsfunction` | `<nombre_funcion>` | Nombre exacto del servicio web de Moodle |
| `moodlewsrestformat` | `json` | Fuerza respuesta JSON en lugar de XML |

### Cómo enviar listas y booleanos

Moodle espera arrays en formato PHP (`nombre[indice]`) y booleanos como `"1"` o `"0"`:

- Lista: `courseids[0]=10&courseids[1]=11`
- Booleano: `includenotenrolledcourses=0`

Ejemplo de función auxiliar mínima en Python (`urllib`, sin dependencias externas):

```python
import json
import os
import urllib.parse
import urllib.request

MOODLE_URL = "https://<tu-moodle>"
WSTOKEN = os.environ["MOODLE_TOKEN"]

def moodle_call(wsfunction: str, **params):
    payload = {
        "wstoken": WSTOKEN,
        "wsfunction": wsfunction,
        "moodlewsrestformat": "json",
    }
    for key, val in params.items():
        if val is None:
            continue
        if isinstance(val, (list, tuple)):
            for i, item in enumerate(val):
                payload[f"{key}[{i}]"] = str(item)
        elif isinstance(val, bool):
            payload[key] = "1" if val else "0"
        else:
            payload[key] = str(val)

    data = urllib.parse.urlencode(payload).encode("utf-8")
    req = urllib.request.Request(
        f"{MOODLE_URL}/webservice/rest/server.php",
        data=data,
        headers={"Content-Type": "application/x-www-form-urlencoded"},
        method="POST",
    )
    with urllib.request.urlopen(req, timeout=15) as resp:
        res = json.loads(resp.read().decode("utf-8"))

    if isinstance(res, dict) and res.get("exception"):
        raise RuntimeError(f"{wsfunction} [{res.get('errorcode')}]: {res.get('message')}")
    return res
```

---

## Catálogo de funciones (`wsfunction`) útiles

### 1. Información de tu usuario (`core_webservice_get_site_info`)
Obtiene tu `userid` (necesario para varias llamadas posteriores) y valida el token.

```bash
curl -sS -X POST "https://<tu-moodle>/webservice/rest/server.php" \
  -d "wstoken=$WSTOKEN" \
  -d "wsfunction=core_webservice_get_site_info" \
  -d "moodlewsrestformat=json"
```

---

### 2. Asignaturas matriculadas (`core_enrol_get_users_courses`)
- **Parámetro:** `userid=<int>`
- **Devuelve:** lista de cursos (`id`, `shortname`, `fullname`).

```bash
curl -sS -X POST "https://<tu-moodle>/webservice/rest/server.php" \
  -d "wstoken=$WSTOKEN" \
  -d "wsfunction=core_enrol_get_users_courses" \
  -d "moodlewsrestformat=json" \
  -d "userid=123"
```

---

### 3. Contenidos y temario de un curso (`core_course_get_contents`)
- **Parámetro:** `courseid=<int>`
- **Devuelve:** array de secciones con sus `modules[]`:
  - `id` (`cmid`), `modname` (`assign`, `quiz`, `resource`, `folder`, `url`, `page`, `book`, `forum`, `subsection`)
  - `uservisible` (`true`/`false`: oculto o restringido para tu usuario)
  - `contents[]`: ficheros (`filename`, `filesize`, `fileurl`, `mimetype`) o enlaces externos en módulos `url`.
  - En Moodle 4.5, una sección con `component: "mod_subsection"` enlaza con su padre mediante `customdata: '{"sectionid":"..."}'`.

```bash
curl -sS -X POST "https://<tu-moodle>/webservice/rest/server.php" \
  -d "wstoken=$WSTOKEN" \
  -d "wsfunction=core_course_get_contents" \
  -d "moodlewsrestformat=json" \
  -d "courseid=42"
```

---

### 4. Tareas y fechas base (`mod_assign_get_assignments`)
- **Parámetros:** ninguno obligatorio (devuelve las de todos tus cursos).
- **Devuelve:** `courses[].assignments[]` con `id` (assign ID), `cmid`, `name`, `intro` (HTML de la descripción), `duedate` y `cutoffdate` (timestamps Unix, `0` si no hay).

```bash
curl -sS -X POST "https://<tu-moodle>/webservice/rest/server.php" \
  -d "wstoken=$WSTOKEN" \
  -d "wsfunction=mod_assign_get_assignments" \
  -d "moodlewsrestformat=json"
```

---

### 5. Estado de entrega, prórrogas y feedback (`mod_assign_get_submission_status`)
- **Parámetro:** `assignid=<int>` (el `id` de `mod_assign_get_assignments`, **no** el `cmid`).
- **Para qué sirve:** saber si ya entregaste, si te han concedido prórroga individual y leer la nota/comentarios/PDF corregido aunque el libro de notas esté oculto.

```bash
curl -sS -X POST "https://<tu-moodle>/webservice/rest/server.php" \
  -d "wstoken=$WSTOKEN" \
  -d "wsfunction=mod_assign_get_submission_status" \
  -d "moodlewsrestformat=json" \
  -d "assignid=1234"
```

Campos clave de la respuesta JSON:

```json
{
  "lastattempt": {
    "submission": {
      "status": "submitted"
    },
    "gradingstatus": "graded",
    "graded": true,
    "extensionduedate": 1795651200
  },
  "feedback": {
    "grade": { "grade": "8.50000", "timemodified": 1795700000 },
    "gradefordisplay": "8,50 / 10,00",
    "gradeddate": 1795700000,
    "plugins": [
      {
        "type": "comments",
        "editorfields": [{ "text": "<p>Buen trabajo en las consultas.</p>" }]
      },
      {
        "type": "editpdf",
        "fileareas": [
          { "area": "download", "files": [{ "filename": "correccion.pdf", "fileurl": "https://..." }] },
          { "area": "stamps", "files": [] }
        ]
      }
    ]
  }
}
```

- **Estado de envío (`submission.status`):** `"submitted"` (entregada), `"draft"` (borrador sin enviar) o `"new"` (sin intentos).
- **PDFs de corrección (`editpdf`):** lee únicamente las áreas `"download"` y `"combined"`; `"stamps"` y `"partial"` contienen sellos internos de Moodle.

---

### 6. Cuestionarios e intentos (`mod_quiz_get_quizzes_by_courses` y `mod_quiz_get_user_attempts`)
- **Listar cuestionarios:** sin parámetros obligatorios → devuelve `quizzes[]` (`id`, `coursemodule` = `cmid`, `course`, `name`, `timeclose`, `intro`).
- **Consultar tus intentos:** requiere `quizid=<int>`, `userid=<int>` y `status=all` → `attempts[]` (`state: "finished"`, `"inprogress"`, `"abandoned"`).

```bash
# 1) Listar cuestionarios de tus cursos
curl -sS -X POST "https://<tu-moodle>/webservice/rest/server.php" \
  -d "wstoken=$WSTOKEN" \
  -d "wsfunction=mod_quiz_get_quizzes_by_courses" \
  -d "moodlewsrestformat=json"

# 2) Ver si ya terminaste el cuestionario (quizid=55, userid=123)
curl -sS -X POST "https://<tu-moodle>/webservice/rest/server.php" \
  -d "wstoken=$WSTOKEN" \
  -d "wsfunction=mod_quiz_get_user_attempts" \
  -d "moodlewsrestformat=json" \
  -d "quizid=55" \
  -d "userid=123" \
  -d "status=all"
```

---

### 7. Notas de una asignatura (`gradereport_user_get_grade_items`)
- **Parámetros:** `courseid=<int>`, `userid=<int>`.
- **Nota práctica:** Algunas asignaturas desactivan el libro de calificaciones al estudiante y devuelven `nopermissiontoviewgrades`. Envuélvelo en `try/except` por curso; las tareas evaluadas igualmente devuelven su nota dentro de `mod_assign_get_submission_status -> feedback`.

```bash
curl -sS -X POST "https://<tu-moodle>/webservice/rest/server.php" \
  -d "wstoken=$WSTOKEN" \
  -d "wsfunction=gradereport_user_get_grade_items" \
  -d "moodlewsrestformat=json" \
  -d "courseid=42" \
  -d "userid=123"
```

Campos clave en `usergrades[0].gradeitems[]`:

```json
{
  "usergrades": [
    {
      "gradeitems": [
        {
          "id": 901,
          "itemname": "Práctica 1 · Consultas SQL",
          "itemtype": "mod",
          "itemmodule": "assign",
          "iteminstance": 1234,
          "cmid": 5678,
          "graderaw": 8.5,
          "grademax": 10.0,
          "gradeformatted": "8,50",
          "percentageformatted": "85,00 %",
          "feedback": "<p>Falta optimizar el último JOIN.</p>",
          "gradeddategraded": 1795700000
        }
      ]
    }
  ]
}
```

- Si `gradeformatted` es `"-"` o `""` y `feedback` está vacío, ese ítem aún no tiene nota publicada.
- Cuando `itemtype == "course"`, representa la nota total calculada de la asignatura (`itemname` suele venir vacío).

---

### 8. Eventos de calendario (`core_calendar_get_action_events_by_timesort`)
- **Parámetros:** `timesortfrom=<unix_ts>`, `limitnum=<int>` (p. ej. `50`).
- **Detalle clave:** En la respuesta `events[]`, el campo `instance` trae el **`cmid`** del módulo (no el `assign.id`), y `timesort` trae el timestamp efectivo visible en el calendario (útil si el profesor amplía la fecha global o añade fecha por grupo).

```bash
curl -sS -X POST "https://<tu-moodle>/webservice/rest/server.php" \
  -d "wstoken=$WSTOKEN" \
  -d "wsfunction=core_calendar_get_action_events_by_timesort" \
  -d "moodlewsrestformat=json" \
  -d "timesortfrom=$(date +%s)" \
  -d "limitnum=50"
```

---

### 9. Foros de avisos (`mod_forum_get_forums_by_courses` y `mod_forum_get_forum_discussions`)
Sirve para detectar avisos de profesores publicados solo en el foro de *"Avisos y noticias"* (`type == "news"`).

```bash
# 1) Obtener IDs de foros
curl -sS -X POST "https://<tu-moodle>/webservice/rest/server.php" \
  -d "wstoken=$WSTOKEN" \
  -d "wsfunction=mod_forum_get_forums_by_courses" \
  -d "moodlewsrestformat=json"

# 2) Leer últimos hilos del foro de noticias (forumid=98)
curl -sS -X POST "https://<tu-moodle>/webservice/rest/server.php" \
  -d "wstoken=$WSTOKEN" \
  -d "wsfunction=mod_forum_get_forum_discussions" \
  -d "moodlewsrestformat=json" \
  -d "forumid=98" \
  -d "perpage=10" \
  -d "page=0"
```

Cada elemento de `discussions[]` incluye `discussion` (ID del hilo para construir `/mod/forum/discuss.php?d=<id>`), `subject`, `message` (HTML), `timemodified` y `userfullname`.

---

## Descargar PDFs y archivos (`webservice/pluginfile.php`)

Dentro de `core_course_get_contents` (y en `feedback.plugins[]` de entregas corregidas), los ficheros traen URLs así en `fileurl`:

```text
https://<tu-moodle>/webservice/pluginfile.php/<contextid>/mod_resource/content/0/archivo.pdf
```

Para descargarlos sin cookies de navegador, añade el parámetro `token`. Si `fileurl` no tiene query string usa `?token=`, pero muchas URLs devueltas por `core_course_get_contents` ya traen `?forcedownload=1`, en cuyo caso debes concatenar con `&token=`:

```bash
# Si fileurl NO tiene '?'
curl -L -sS "https://<tu-moodle>/webservice/pluginfile.php/<contextid>/mod_resource/content/0/archivo.pdf?token=$WSTOKEN" -o archivo.pdf

# Si fileurl YA incluye '?forcedownload=1' (o cualquier query param)
SEP="&"
[[ "$FILE_URL" != *\?* ]] && SEP="?"
curl -L -sS "${FILE_URL}${SEP}token=$WSTOKEN" -o archivo.pdf
```

> **Importante:** Comprueba siempre que la URL pertenezca a tu host de Moodle antes de añadir `token=`. En módulos de tipo `url`, `fileurl` apunta a sitios externos (Google Drive, YouTube, GitHub) donde **nunca** debes adjuntar tu `wstoken`.

---

## Ejemplo completo en Python: listar entregas pendientes y notas

Script autocontenido (solo librería estándar de Python) que filtra los cursos activos del usuario y consulta entregas pendientes y calificaciones publicadas (requiere `MOODLE_TOKEN` en el entorno):

```python
import datetime
import json
import os
import time
import urllib.parse
import urllib.request

MOODLE_URL = "https://<tu-moodle>"
WSTOKEN = os.environ["MOODLE_TOKEN"]

def moodle_call(wsfunction: str, **params):
    payload = {"wstoken": WSTOKEN, "wsfunction": wsfunction, "moodlewsrestformat": "json"}
    for k, v in params.items():
        if v is None:
            continue
        if isinstance(v, (list, tuple)):
            for i, item in enumerate(v):
                payload[f"{k}[{i}]"] = str(item)
        elif isinstance(v, bool):
            payload[k] = "1" if v else "0"
        else:
            payload[k] = str(v)
    data = urllib.parse.urlencode(payload).encode("utf-8")
    req = urllib.request.Request(
        f"{MOODLE_URL}/webservice/rest/server.php",
        data=data,
        headers={"Content-Type": "application/x-www-form-urlencoded"},
        method="POST",
    )
    with urllib.request.urlopen(req, timeout=15) as resp:
        res = json.loads(resp.read().decode("utf-8"))
    if isinstance(res, dict) and res.get("exception"):
        raise RuntimeError(f"{wsfunction} [{res.get('errorcode')}]: {res.get('message')}")
    return res

now = int(time.time())
info = moodle_call("core_webservice_get_site_info")
uid = info["userid"]
raw_courses = moodle_call("core_enrol_get_users_courses", userid=uid)
courses = {
    c["id"]: c["shortname"]
    for c in raw_courses
    if not c.get("enddate") or c["enddate"] > now
}

print("=== ENTREGAS ===")
if courses:
    assign_data = moodle_call("mod_assign_get_assignments", courseids=list(courses))
    for course in assign_data.get("courses", []):
        cname = courses.get(course["id"], course.get("shortname", str(course["id"])))
        for assign in course.get("assignments", []):
            st = moodle_call("mod_assign_get_submission_status", assignid=assign["id"])
            last = st.get("lastattempt") or {}
            sub_status = (last.get("submission") or {}).get("status", "new")
            due_ts = last.get("extensionduedate") or assign.get("duedate") or 0
            due_txt = (
                datetime.datetime.fromtimestamp(due_ts).strftime("%d/%m %H:%M")
                if due_ts
                else "Sin fecha"
            )
            print(f"[{sub_status:9}] {due_txt:11} | {cname:16} | {assign['name']}")

print("\n=== NOTAS PUBLICADAS ===")
for cid, cname in courses.items():
    try:
        ugs = moodle_call("gradereport_user_get_grade_items", courseid=cid, userid=uid).get("usergrades", [])
    except RuntimeError:
        continue  # Curso con libro de notas oculto (nopermissiontoviewgrades)
    for gi in (ugs[0].get("gradeitems", []) if ugs else []):
        grade = (gi.get("gradeformatted") or "").strip()
        if grade and grade != "-":
            print(f"{cname:16} | {gi.get('itemname') or 'Total curso'}: {grade}")
```

---

## Manejo de errores en Moodle REST

Moodle casi siempre responde **HTTP `200 OK`**, incluso cuando algo falla. La señal de error está dentro del JSON:

```json
{
  "exception": "moodle_exception",
  "errorcode": "invalidtoken",
  "message": "Invalid token - token not found"
}
```

| `errorcode` | Significado | Qué hacer |
|---|---|---|
| `invalidtoken` / `tokenexpired` | Token inválido, caducado o usuario revocado | Regenerar con [01-conseguir-token.md](01-conseguir-token.md) |
| `nopermissiontoviewgrades` | El profesor ocultó el libro de notas en ese curso | Capturar error por curso; las tareas evaluadas siguen visibles vía `mod_assign_get_submission_status` |
| `requireloginerror` / acceso restringido | Actividad oculta o cerrada para tu grupo | Verificar `uservisible` en `core_course_get_contents` antes de consultarla |
