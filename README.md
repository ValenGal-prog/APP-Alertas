# APP-Alertas
# Prueba técnica Colfrance – Alertas y paradas de máquina

## Ejecución

```bash
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python manage.py makemigrations maquinas   # solo si no existe maquinas/migrations/0001_initial.py
python manage.py migrate
python manage.py crear_demo --datos        # usuarios por rol + 10 alertas y 5 paradas
python manage.py runserver
```

## Usuarios de demostración (clave de todos: `Demo12345`)

| Usuario | Rol | Puede | No puede |
|---|---|---|---|
| operario1, operario2 | Operario | Crear alertas y ver solo las suyas | Editar alertas, registrar/cancelar paradas |
| supervisor | Supervisor | Ver y editar alertas, registrar paradas | Cancelar paradas |
| jefe | Jefe | Ver alertas y paradas, cancelar paradas con motivo | Editar alertas, registrar paradas |

## Cómo está resuelto

- **Roles** = grupos de Django con permisos de modelo (`add_alerta`, `change_alerta`, `view_parada`, `cancelar_parada`…). Se crean con `crear_demo`.
- **Permisos en el backend:** la página tiene una sola URL; cada acción POST (`crear_alerta`, `editar_alerta`, `crear_parada`, `cancelar_parada`) se mapea a un permiso en `views.py` y se responde **403** si falta, antes de tocar datos. Ocultar botones en la plantilla es solo comodidad.
- **Operario:** el listado se filtra por `usuario=request.user`. El usuario de una alerta/parada siempre se toma de la sesión, nunca del formulario.
- **Cancelar parada:** no se borra; se marcan `cancelada`, `cancelada_por`, `cancelada_en`, `motivo_cancelacion`. Se usa un `UPDATE ... WHERE cancelada=False`, así una segunda cancelación no cambia nada.
- **Duración:** se calcula con fecha y hora completas (`(fin - inicio)` en minutos), por eso 23:50 → 00:20 del día siguiente da 30.
- **Validaciones:** campos obligatorios sin espacios vacíos; `Parada.clean()` rechaza `fin <= inicio`; nada se guarda si el formulario es inválido.
- **CSRF** activo (`CsrfViewMiddleware` + `{% csrf_token %}`), `@login_required` en la vista, zona horaria `America/Bogota`.

Supuesto: el operario no ve la sección de paradas (el enunciado no le da ese permiso).

## Casos de comprobación (manual)

Para 3 y 4 con petición manual desde `python manage.py shell`:
```python
from django.test import Client
from maquinas.models import Alerta, Parada
c = Client(HTTP_HOST="localhost"); c.login(username="operario1", password="Demo12345")
a = Alerta.objects.first()
r = c.post("/", {"accion": "editar_alerta", "alerta_id": a.pk, "e-maquina": "X", "e-descripcion": "hackeada"})
print(r.status_code, Alerta.objects.get(pk=a.pk).descripcion)   # 403 y descripción sin cambios (repetir con "jefe")

c = Client(HTTP_HOST="localhost"); c.login(username="jefe", password="Demo12345")
p = Parada.objects.filter(cancelada=False).first()
c.post("/", {"accion": "cancelar_parada", "parada_id": p.pk, "motivo": "Primer motivo"})
c.post("/", {"accion": "cancelar_parada", "parada_id": p.pk, "motivo": "Segundo motivo"})
p.refresh_from_db(); print(p.motivo_cancelacion, p.cancelada_por, p.cancelada_en, Parada.objects.count())
```

| # | Caso | Resultado | Fallas y cómo las resolví |
|---|---|---|---|
| 1 | Descripción/motivo vacío y fin ≤ inicio se rechazan, no se guarda | ____ | ____ |
| 2 | 23:50 → 00:20 muestra 30 min | ____ | ____ |
| 3 | Editar alerta como operario/jefe por petición manual → rechazado, sin cambios | ____ | ____ |
| 4 | Cancelar dos veces → se conserva el primer usuario/fecha/motivo, sin duplicar | ____ | ____ |
| 5 | 10 alertas + 5 paradas: la página carga, conserva datos, cada operario ve solo las suyas | ____ | ____ |

## Integración en un monolito Django (3 líneas)
1. Copiar la app `maquinas`, agregarla a `INSTALLED_APPS` e incluir `maquinas.urls`; los modelos ya usan `settings.AUTH_USER_MODEL`, así que reutilizan los usuarios del monolito sin tabla propia.
2. Quitar `login/logout` propios (usar los del monolito) y asignar los grupos Operario/Supervisor/Jefe a los usuarios existentes, en lugar de los usuarios demo.
3. Correr `migrate` (solo crea tablas de `maquinas`) y probar primero en un entorno de pruebas.

## Resolución de problemas y presión

**Caso 1 – No se registran paradas en turno de alta producción.**
Aviso al supervisor en pocos minutos: "estoy investigando, actualizo en 15 min; mientras tanto registren las paradas en papel para cargarlas después".
Investigo sin tocar datos: traceback/logs, último cambio o despliegue, espacio en disco, y reproduzco el error en una copia local.
Mantengo la calma siguiendo una lista de pasos y no cambio nada sin entender la causa.
Si en ~30 min no hay causa o hay riesgo para los datos, pido apoyo a un compañero senior/TI contándole lo ya descartado.

**Caso 2 – Reporte urgente del jefe + usuario sin acceso, todos con prioridad.**
Ordeno por impacto: primero la falla que frena la producción, luego el acceso (suele resolverse en minutos), luego el reporte.
Pregunto: ¿a cuántas personas afecta?, ¿desde cuándo?, ¿qué mensaje de error sale?, ¿para cuándo se necesita el reporte y para qué decisión?
A cada persona le doy una hora estimada y cuándo le actualizo, sin prometer lo que no controlo.
Delegaría el restablecimiento de contraseña al soporte de primer nivel y la extracción del reporte a un compañero.

**Caso 3 – Mi cambio causa errores y un compañero propone tocar producción sin revisar.**
Digo que no a modificar producción sin revisión y propongo volver a la versión anterior (`git revert`), que restablece el servicio rápido y es reversible.
Aviso de inmediato a mi líder y a los usuarios afectados, y asumo que el error fue de mi cambio.
Antes de otro cambio verifico en logs que el cambio es la causa, pruebo el revert en local y compruebo login, crear alerta y registrar parada.
Después hago la corrección con revisión de un compañero y dejo documentado qué falló.

## Uso de inteligencia artificial
Usé Claude (Anthropic) como apoyo para generar la estructura inicial del proyecto y el borrador del README. Revisé el código, lo ejecuté y puedo explicarlo. ____ (ajustar a lo que realmente hice)

## Pendientes / mejoras
- Pruebas automatizadas (no requeridas en esta prueba).
- Restricción a nivel de base de datos (`CheckConstraint fin > inicio`).
- Paginación y filtros por máquina/fecha; lista de máquinas como modelo en vez de texto libre.
- Auditoría de ediciones de alertas; cambiar la clave de demo y `SECRET_KEY` antes de cualquier despliegue.
