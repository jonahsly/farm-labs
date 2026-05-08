---
name: execute-sprint-task
description: Playbook para implementar una tarea concreta del roadmap §10 de CLAUDE.md (Sprints 1–9). Recibe un ID de tarea (ej. "1.3", "2.4") o una descripción equivalente y aplica los cambios al código respetando las convenciones del proyecto. Devuelve archivos modificados, resumen de cambios, commit message exacto y DoD parcial. Úsalo siempre que la petición sea "implementa", "ejecuta", "haz la tarea", "arregla X" donde X es algo listado en §10.
---

# Skill: execute-sprint-task

Implementas tareas del plan de sprints definido en `CLAUDE.md` §10. No improvisas: cada tarea trae instrucciones precisas y un nombre de commit, los respetas literalmente.

## Inputs esperados

- **ID de tarea** (ej. `1.3`, `2.4`) o descripción equivalente.
- Opcional: contexto adicional del usuario (preferencias entre opciones A/B, etc.).

## Pasos

### 1. Localizar la fila en §10

Lee `CLAUDE.md` y busca la fila por ID. Extrae:
- Columna "Qué hacer" → instrucciones literales.
- Columna "Commit" → nombre exacto, NO lo modifiques.
- Encabezado del sprint → DoD aplicable.

Si la tarea no existe en §10, **detén y avisa** al usuario: la tarea no está en el roadmap.

### 2. Leer archivos afectados

Antes de editar, lee cada archivo que vas a tocar para conocer:
- Su indentación (tabs vs. 2 espacios — varía por archivo).
- El estilo de imports (relativo, agrupado por capa).
- Convenciones específicas (ej. `useRef` vs `event.currentTarget` para forms).

### 3. Aplicar cambios

Sigue literalmente la columna "Qué hacer". Reglas que NUNCA rompes:

1. Toda nueva ruta o `<Link>` usa `APP_ROUTES` de `src/constants/routes.js`.
2. Todo copy visible usa `UI_TEXT` de `src/constants/uiText.js`. Si añades copy nuevo, agrégalo a `UI_TEXT` primero.
3. Datos del catálogo pasan por `normalizeProduct(s)` antes del render.
4. Forms de auth importan `isValidEmail` / `isValidPassword` de `src/utils/validation.js`.
5. Prohibido `console.log` con credenciales o PII.
6. No introduces dependencias nuevas si la tarea no lo pide. Si lo pide, usa `npm install` con la versión exacta indicada.
7. No tocas `node_modules/` ni `build/`.
8. Respetas la indentación del archivo. Si abre con tabs, sigues con tabs. **Solo el Sprint 7.3 reformatea masivamente.**
9. `export default` al final del archivo.
10. Componentes nuevos viven en `src/components/<Nombre>/Nombre.jsx` + `Nombre.css` hermano.

### 4. Ajustar tests si la tarea los rompe

Si los cambios afectan a `src/App.test.js` o `src/BasenameRouting.test.js`, ajústalos en el mismo commit. No abras tareas separadas para "fix tests".

### 5. Verificar

Antes de declarar done, invoca la skill `verify-quality-gates` o ejecuta manualmente:

```bash
npm run lint
npm test -- --watchAll=false
npm run build
```

Si algo falla, **NO declares done**: reporta el bloqueo.

## Salida obligatoria

```
## Tarea
<ID>: <descripción literal de §10>

## Archivos modificados
- src/path/al/archivo.jsx
- src/path/al/otro.jsx

## Resumen de cambios
- <archivo>: <1-2 líneas>
- <archivo>: <1-2 líneas>

## Commit sugerido
<copiado tal cual de §10>

## DoD parcial (Sprint X)
✅ <criterio cubierto por esta tarea>
⏳ <criterio aún pendiente — requiere otras tareas>

## Bloqueos
<vacío si no hay; si hay, descríbelos aquí>

## Próxima tarea recomendada
<ID y commit name de la siguiente fila del mismo sprint>
```

## Casos especiales por sprint

- **Sprint 1.3 (bump axios):** después de `npm install axios@^1.7.4`, abre `src/hooks/useGetProducts.js` y verifica que `signal`/`timeout`/`abort` siguen funcionando — la API no cambió pero la verificación es obligatoria.
- **Sprint 2.5 (categorías):** la tabla ofrece dos commits (Opción A: filtro real / Opción B: ocultar). **Pregunta al usuario** cuál antes de elegir.
- **Sprint 3.x (auth):** persiste el "mock token" en `sessionStorage`, NUNCA en `localStorage` (decisión §7 #5).
- **Sprint 6.x (carrito persistente):** solo persistes el carrito; no vuelques PII en `localStorage`.
- **Sprint 7.3 (Prettier masivo):** ÚNICA tarea donde reformateas archivos no relacionados con tu cambio. Añade el hash al `.git-blame-ignore-revs`.
- **Sprint 8 (Vite):** trabaja en rama `feat/vite-migration`. Pasos secuenciales (8.2 antes de 8.3, etc.). NO ejecutes en paralelo.
- **Sprint 9 (TS):** una subcarpeta = un commit. No conviertas todo `src/` de golpe.

## Anti-patrones

- Renombrar variables fuera del scope de la tarea.
- Convertir tabs↔espacios en archivos no relacionados.
- Añadir dependencias por iniciativa propia.
- Saltar a otra tarea "porque está cerca".
- Crear archivos `.md` o `README` que no se pidieron.
- Reformular el commit message: cópialo letra por letra de §10.
- "Limpieza al pasar" en archivos que no necesitas tocar.
- Marcar DoD como cumplido si hay un test rojo o el build falla.
