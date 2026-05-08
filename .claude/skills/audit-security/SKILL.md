---
name: audit-security
description: Playbook de auditoría de seguridad del repositorio FarmLabs sin modificar archivos. Detecta credenciales en logs, PII hardcodeada, dependencias con CVEs, hot-linking sin referrerpolicy, dangerouslySetInnerHTML, almacenamiento inseguro de tokens y secrets versionados. Devuelve un reporte clasificado por severidad (🔴/🟠/🟡/🟢) con archivo + línea + acción ligada a una tarea de §10. Úsalo antes de cada deploy, antes de cerrar el Sprint 1, o cuando el usuario pida "auditar", "revisar seguridad", "buscar fugas" o "revisar CVEs". SOLO LEE.
---

# Skill: audit-security

Tu trabajo es **encontrar y reportar** problemas de seguridad. **Nunca modificas archivos.** Si te piden arreglar algo, recuérdalo y delega al `execute-sprint-task` con el ID correspondiente.

## Alcance

Auditas:

- `src/**/*.{js,jsx,ts,tsx}`
- `public/**`
- `package.json` y `package-lock.json` (versiones)
- Archivos de config en raíz: `.env*`, `.gitignore`, `*.config.*`
- Workflows de CI (`.github/workflows/*.yml`) cuando existan

Ignoras: `node_modules/`, `build/`, `dist/`, `coverage/`, `.git/`.

## Checklist de auditoría

Para cada apartado: busca con `Grep`, abre el archivo con `Read` para confirmar contexto, reporta hallazgos con archivo + línea.

### 1. Credenciales en logs 🔴

Patrones (regex):
- `console\.(log|debug|info|warn|error).*(password|token|secret|api[_-]?key|credential|authorization|jwt)`

Archivos típicos: `src/pages/Login/`, `src/pages/CreateAccount/`, `src/pages/NewPassword/`, `src/pages/PasswordRecovery/`.
Acción ligada: Sprint 1.1 (`fix(security): remove credential logging from auth forms`).

### 2. PII hardcodeada 🔴

Patrones:
- Emails reales versionados (no `@example.com`, no `farmlabs@`).
- Nombres propios como valor estático en JSX (`>Jonah Sly<`, `>Maria López<`).
- Números de teléfono (`\b\d{3}[-.\s]?\d{3}[-.\s]?\d{4}\b`).
- Direcciones físicas (calles, códigos postales).

Acción ligada: Sprint 1.2 (`fix(security): remove hardcoded PII from MyAccount`).

### 3. Dependencias con CVEs conocidas 🟠

Lee `package.json`. CVEs documentadas:
- `axios < 1.7.4` → CVE-2023-45857 (XSRF), CVE-2024-39338 (SSRF). Acción: Sprint 1.3.
- `react-scripts 5.0.1` → CRA deprecado, varias CVEs transitivas (nth-check, postcss, semver). Acción: Sprint 8 (migración a Vite).
- Si Bash está disponible: `npm audit --json` y resume `vulnerabilities` por severidad. **NO corras** `npm audit fix`.

### 4. XSS y sanitización 🟠

Patrones:
- `dangerouslySetInnerHTML`
- `eval(`
- `new Function(`
- Asignación directa a `.innerHTML`

Este repo no debería tener ninguno; cualquier hallazgo es 🟠 Alto.

### 5. Almacenamiento inseguro de tokens 🟠

Patrones:
- `localStorage\.(setItem|getItem).*(token|jwt|auth|session|access)`
- `document\.cookie\s*=` sin `Secure; HttpOnly; SameSite`

Recomendación: usar `sessionStorage` para mocks de auth (Sprint 3), o cookies `HttpOnly` + `SameSite=Lax` cuando exista backend.

### 6. Hot-linking de imágenes sin política 🟡

Patrones:
- `<img src="https?://[^"]*"` sin `referrerpolicy="no-referrer"`.

Reporta dominios externos (en este repo: `purecannastore.com`).
Acción ligada: Sprint 5.6 (`fix(images): set no-referrer policy on external images`).

### 7. Fetch / Axios inseguro 🟡

Patrones:
- `axios\.(get|post|put).*['"]http://` (HTTP plano).
- `withCredentials: true` hacia orígenes no propios.
- Falta de `timeout` en llamadas (robustez, no CVE).

### 8. Secrets versionados 🔴

Patrones (búsqueda en TODO el repo excepto `node_modules/`):
- `AKIA[0-9A-Z]{16}` (AWS).
- `sk_live_[0-9a-zA-Z]{24,}` (Stripe).
- `xox[baprs]-[0-9a-zA-Z\-]{10,}` (Slack).
- `ghp_[0-9a-zA-Z]{36}` (GitHub PAT).
- `eyJ[A-Za-z0-9_\-]+\.[A-Za-z0-9_\-]+\.[A-Za-z0-9_\-]+` (JWT — verifica si es real).
- Archivos `.env*` no listados en `.gitignore`.

Severidad: 🔴 Crítico (requiere revocación inmediata + `git filter-repo`).

### 9. `.gitignore` insuficiente 🟡

Verifica que estén ignorados (mínimo):
- `node_modules/`
- `build/` y/o `dist/`
- `.env`, `.env.local`, `.env.*.local`
- `coverage/`
- `*.log`
- `.DS_Store`

Reporta los faltantes.

### 10. Configuración de despliegue 🟢

- `homepage` en `package.json` apunta al dominio correcto (`https://jonahsly.github.io/FarmLabs/`).
- `gh-pages -d build` solo despliega lo que está en `build/` — confirmar que no se cuela código fuente con secrets.

## Formato del reporte

```
# Auditoría de seguridad — FarmLabs (<fecha>)

## Resumen
- 🔴 Crítico: <N>
- 🟠 Alto: <N>
- 🟡 Medio: <N>
- 🟢 Bajo: <N>

## Hallazgos

### 🔴 1. <Título corto>
- **Archivo:** `src/pages/Login/Login.jsx:40`
- **Patrón detectado:** `console.log({ password })`
- **Riesgo:** fuga de credenciales en consola del navegador.
- **Acción sugerida:** Sprint 1.1 (`fix(security): remove credential logging from auth forms`).

### 🟠 2. <Título corto>
- ...

## Hallazgos fuera de checklist
<si encontraste algo nuevo, descríbelo y propón añadirlo al checklist>

## Recomendación final
<una frase: "no deployar hasta cerrar críticos" / "verde para deploy" / etc.>
```

Cada hallazgo enlaza con el ID de §10 que lo resuelve.

## Lo que NO debes hacer

- **No edites archivos. NUNCA.**
- No instales paquetes ni corras `npm audit fix`.
- No abras PRs ni hagas commits.
- No marques nada como "resuelto"; tu output es solo informativo.
- No expongas valores reales de secrets en el reporte: enmascara con `****`.
- No deduzcas severidad por intuición — usa la tabla del checklist.

Si encuentras un patrón nuevo que no esté en el checklist, repórtalo en "Hallazgos fuera de checklist" y propón añadirlo a esta skill.
