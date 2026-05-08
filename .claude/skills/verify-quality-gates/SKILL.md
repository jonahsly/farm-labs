---
name: verify-quality-gates
description: Playbook para correr los quality gates del proyecto FarmLabs (lint, tests y build) y reportar el resultado con severidad y acciones sugeridas. Úsalo siempre como paso final tras una implementación (después de execute-sprint-task), antes de declarar una tarea "done", o cuando el usuario pida explícitamente "verifica", "corre los gates", "lint+test+build" o "está listo para PR". Puede ejecutar comandos pero no edita código fuente.
---

# Skill: verify-quality-gates

Verificas que el repo está sano antes de declarar una tarea o sprint como completado. Los tres gates son: **lint**, **tests**, **build**. Si alguno rompe, la tarea NO está done.

## Cuándo se invoca

1. **Automáticamente** al final de `execute-sprint-task`.
2. **Bajo petición** del usuario: "verifica", "corre los gates", "está listo".
3. **Antes de un deploy** (junto con `audit-security`).

## Inputs esperados

- Opcional: lista de archivos modificados recientemente para acotar el reporte.
- Opcional: ID de tarea de §10 (para enlazar el reporte al DoD del sprint).

## Pasos

### 1. Lint

```bash
npm run lint
```

- Reporta el código de salida.
- Si hay warnings nuevos respecto a la baseline, lístalos con archivo + regla.
- No corras `lint:fix` automáticamente; el usuario decide.

### 2. Tests

```bash
npm test -- --watchAll=false --coverage=false
```

- Reporta tests pasados, fallados y skipped.
- Si hay tests fallados, incluye el nombre del test y el error de la primera línea de stack.
- Si la tarea era de Sprint 4 (cobertura), añade `--coverage` y reporta el porcentaje en `utils/` y `hooks/`.

### 3. Build

```bash
npm run build
```

- Reporta éxito/fallo.
- Si falla, incluye el primer error completo (no solo "Failed to compile").
- Si tiene éxito, opcionalmente reporta tamaño del bundle (`du -sh build/static/js/*.js`).

### 4. Clasificar resultado

| Resultado | Estado |
|-----------|--------|
| Lint OK + Tests OK + Build OK | ✅ Verde — listo para commit/deploy |
| Lint con warnings nuevos, resto OK | 🟡 Amarillo — done con deuda menor |
| Tests fallados o build roto | 🔴 Rojo — NO declarar done |

## Formato del reporte

```
# Quality gates — <fecha/hora>

## Lint
<✅ Pasa | 🟡 Warnings | 🔴 Errores>
<si hay warnings/errores: lista con archivo + regla + línea>

## Tests
<✅ Pasan N/N | 🔴 Fallan X/N>
<si fallan: nombre del test + primer error>

## Build
<✅ Compila | 🔴 Falla>
<si falla: error completo>
<si compila: opcional, tamaño de bundle>

## Veredicto
<✅ VERDE — listo / 🟡 AMARILLO — listo con deuda / 🔴 ROJO — bloqueado>

## Acciones sugeridas
- <si verde y la tarea era de §10: declarar DoD parcial cubierto>
- <si amarillo: abrir tarea de limpieza si no existe>
- <si rojo: NO commitear, arreglar primero>
```

## Reglas

- **Si el resultado es 🔴, el agente padre NO puede marcar la tarea como done.** Esto es un bloqueo.
- **No edites código** para arreglar warnings; eso lo hace `execute-sprint-task` en una tarea explícita.
- **No corras `npm install`** salvo que la tarea actual lo requiera (Sprint 1.3, 1.5, etc.); el comando `npm test` puede pedirlo si `node_modules/` está fresco — repórtalo y deja que el usuario decida.
- **No bypasees tests rotos** con `--testPathIgnorePatterns` salvo instrucción explícita del usuario.
- **No inflas el output**: si todo pasa, una sola línea por gate basta.

## Casos especiales

- **Sprint 4 (cobertura):** añade `--coverage` y reporta cobertura por carpeta. Falla 🔴 si `utils/` o `hooks/` < 70%.
- **Sprint 7.3 (Prettier masivo):** después del reformat, valida con `npx prettier --check src/` antes de los gates.
- **Sprint 8 (Vite):** los comandos cambian: `npm run build` produce `dist/` no `build/`. Ajusta los paths del reporte.
- **Sprint 8.6 (Vitest):** sustituye `npm test` por `npm run test -- --run` (modo no-watch de Vitest).

## Anti-patrones

- Declarar verde sin haber corrido los tres gates.
- Saltar tests porque "el cambio era solo de CSS".
- Editar código para "arreglar" warnings al pasar.
- Esconder fallos en el reporte: si rompe, se reporta en rojo y al inicio.
