# Desarrollo con Claude Code sin deuda técnica
 
Guía corta y accionable. Actualizada: octubre 2026. Fuentes al final.
 
---
 
## 0. Principio rector
 
> **La IA escribe rápido; tú eres responsable de lo que entra a `main`.**
> Trata el código de la IA como el de un junior muy veloz: útil, pero no confiable por defecto.
 
Dos restricciones explican casi todo:
 
1. **El contexto se llena y el rendimiento baja.** Es el recurso más valioso.
2. **Claude se detiene cuando "parece terminado".** Sin un check ejecutable, tú eres el verificador.
---
 
## 0.5 Fases de 0 a producción
 
Ningún estándar define una lista única de fases. Esta tabla es una síntesis de ISO/IEC/IEEE 12207:2026, Microsoft SDL, NIST SSDF y la checklist de lanzamiento de Google SRE. Las fases **no son lineales**: se repiten de forma iterativa.
 
| # | Fase | Qué se hace | Dónde entra la IA |
|---|---|---|---|
| 0 | Concepto / evaluar la idea | Decidir si vale la pena construirla | Investigar, comparar alternativas |
| 1 | Requisitos y planificación | Alcance, requisitos funcionales, de seguridad y privacidad | Entrevista y spec (`SPEC.md`) |
| 2 | Diseño | Arquitectura, modelo de datos, revisión de diseño y seguridad | Plan mode, ADR |
| 3 | Implementación | Código en lotes pequeños, estándares, control de versiones | Claude Code con tests y hooks |
| 4 | Verificación | Pruebas, análisis estático, revisión, validación de seguridad | Subagente revisor, SAST en CI |
| 5 | Release / despliegue | Checklist de lanzamiento, plan de respuesta, rollout por etapas | Validar configs, dry-run |
| 6 | Operación y respuesta | Monitoreo, incidentes, gestión de vulnerabilidades | Apoyo en diagnóstico (con humano) |
| 7 | Mantenimiento y retiro | Evolución, parches, fin de vida | Actualizar spec/ADR junto con el código |
 
**Por qué importa con IA:** las fases 4 a 6 son donde se atrapa lo que la IA pudo generar mal. No las acortes porque el código se escribió rápido.
 
Fuentes: [ISO/IEC/IEEE 12207:2026](https://www.iso.org/es/contents/data/standard/09/02/90219.html) · [Microsoft SDL](https://learn.microsoft.com/en-ca/compliance/assurance/assurance-microsoft-security-development-lifecycle) · [NIST SP 800-218](https://csrc.nist.gov/pubs/sp/800/218/final) · [Google SRE: Launch Checklist](https://sre.google/sre-book/launch-checklist/)
 
---
 
## 1. Flujo base: Explorar → Planear → Implementar → Verificar → Commit
 
| Fase | Cómo | Ejemplo |
|---|---|---|
| Explorar | Plan mode (`Shift+Tab` x2 o `claude --permission-mode plan`) | `lee src/auth y explica cómo manejamos sesiones` |
| Planear | Pide plan; edítalo con `Ctrl+G` | `quiero añadir Google OAuth. ¿Qué archivos cambian? Haz un plan` |
| Implementar | Aprueba el plan, con tests | `implementa el plan, escribe tests y corre la suite` |
| Verificar | Evidencia, no afirmaciones | `muéstrame el output de los tests` |
| Commit | Diff pequeño | `commit descriptivo y abre PR` |
 
**Regla:** si puedes describir el diff en una frase, salta el plan.
 
---
 
## 2. Verificación (lo más importante)
 
Siempre dale a Claude un check que dé pass/fail: tests, build, linter, typecheck, screenshot.
 
```text
❌ "implementa validateEmail"
✅ "escribe validateEmail. Casos: user@example.com → true, 'invalid' → false,
    user@.com → false. Corre los tests al terminar."
```
 
Niveles de rigor (de menor a mayor setup):
- **En el prompt:** "corre los tests e itera".
- **`/goal`:** un evaluador revisa tras cada turno hasta que se cumpla.
- **Hook `Stop`:** bloquea que termine hasta que pase el check.
- **Subagente revisor:** un contexto limpio intenta refutar el resultado.
Si no puedes verificarlo, no lo mergees.
 
---
 
## 3. CLAUDE.md: corto y útil
 
- Genera la base con `/init`, luego poda. Revisa con `/context` y `/doctor`.
- Prueba para cada línea: *"¿Quitarla haría que Claude se equivoque?"* Si no, bórrala.
- Apunta a menos de ~200 líneas; si pasa, empieza a ignorar reglas.
- Commit a git para que el equipo lo mejore.
- Es **consultivo**: no garantiza cumplimiento. Lo obligatorio va en hooks.
| ✅ Incluir | ❌ Excluir |
|---|---|
| Comandos que no puede adivinar | Lo que se deduce leyendo el código |
| Estilo distinto al default | Convenciones estándar del lenguaje |
| Cómo correr tests | Documentación larga de APIs (enlaza) |
| Convenciones de rama/PR | Info que cambia seguido |
| Decisiones de arquitectura | Descripciones archivo por archivo |
 
```markdown
# Estilo
- ES modules, no CommonJS
# Flujo
- Corre typecheck tras cambios
- Prefiere tests individuales, no toda la suite
- IMPORTANT: nunca edites /migrations a mano
```
 
Si una regla se ignora, enfatiza **solo esa** con `IMPORTANT`. Contenido solo a veces relevante → **skill**, no CLAUDE.md.
 
---
 
## 4. Hooks: lo que debe pasar siempre
 
CLAUDE.md sugiere; los hooks **garantizan**.
 
```json
// .claude/settings.json
{
  "hooks": {
    "PostToolUse": [
      { "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "npm run lint --silent" }] }
    ],
    "Stop": [
      { "hooks": [{ "type": "command", "command": "npm run typecheck && npm test --silent" }] }
    ]
  }
}
```
 
Casos de uso: lint tras cada edición, bloquear escritura en `migrations/`, tests antes de terminar, bloquear lectura de `.env`.
Tip: pídele a Claude que te escriba el hook.
 
---
 
## 5. Gestión de contexto
 
- `/clear` entre tareas no relacionadas.
- **Regla de las 2 correcciones:** si corriges 2 veces lo mismo, `/clear` y reescribe un mejor prompt.
- Investigaciones → **subagentes** (contexto aparte): `usa un subagente para investigar cómo se refresca el token`.
- `/compact Enfócate en los cambios de API` para compactar con intención.
- `Esc Esc` o `/rewind` para volver a un checkpoint (no reemplaza a git).
- `/rename oauth-migration` + `claude --resume` para tareas largas.
- Referencia con `@src/file.ts` en vez de describir dónde está.
---
 
## 6. Subagentes y revisión independiente
 
El código revisado por quien lo escribió es sesgado. Usa contexto fresco.
 
```markdown
<!-- .claude/agents/security-reviewer.md -->
---
name: security-reviewer
description: Revisa vulnerabilidades de seguridad
tools: Read, Grep, Glob, Bash
---
Revisa: inyección (SQL/XSS/comandos), authn/authz, secretos en código,
manejo inseguro de datos. Da líneas exactas y fix sugerido.
```
 
Patrón **Writer / Reviewer:** Sesión A implementa, Sesión B (limpia) revisa `@src/middleware/rateLimiter.ts`, A corrige.
Pídele al revisor **solo gaps de correctitud o requisitos**; si no, terminarás con sobre-ingeniería (abstracciones y defensas innecesarias).
 
Para trabajo masivo: `/batch` (5–30 subagentes en worktrees) o `claude -p` en bucle. Prueba en 2–3 archivos antes de escalar.
 
---
 
## 6.5 Specs antes que código
 
Para features grandes, que Claude te entreviste:
 
```text
Quiero construir [X]. Entrévistame con AskUserQuestion: implementación, edge cases,
tradeoffs. Al terminar escribe SPEC.md.
```
 
Luego **sesión nueva** para implementar. Un buen spec nombra archivos, lo que queda fuera de alcance y un check end-to-end.
 
---
 
## 7. Seguridad
 
Datos recientes (varían por estudio y metodología, úsalos como orden de magnitud):
- IOActive (abr 2026): ~31.6% de muestras de código IA fueron explotables; ningún modelo logró 100% seguro. Prompts con seguridad explícita mejoraron resultados.
- Veracode (2025-26): ~45% de muestras con vulnerabilidad OWASP Top 10, sin mejora clara entre generaciones de modelos.
- Menos del 50% de devs revisa el código IA antes de commit (reporte de terceros).
Prácticas:
- **Código IA = input no confiable**, sobre todo auth, criptografía, infra y lógica de negocio.
- Revisión humana obligatoria antes de merge.
- SAST + escaneo de dependencias en CI (más de un scanner detecta más).
- Nunca secretos en prompts o código; `.env` fuera del alcance; usa permisos y sandbox.
- Verifica que las dependencias que sugiere **existan** y sean mantenidas.
```text
Ejemplo de prompt: "Implementa login. Usa consultas parametrizadas, hashing con
argon2/bcrypt, rate limiting, y verifica roles en cada endpoint. Escribe tests de abuso."
```
 
---
 
## 8. Evitar deuda técnica (el núcleo)
 
Qué muestra la evidencia:
- **GitClear** (211M líneas, 2020–2024): menos reuso y más código duplicado con asistentes IA.
- **arXiv 2603.28592** (mar 2026): estudio a gran escala sobre cómo la deuda introducida por IA persiste en repos reales.
- **arXiv 2601.07786** (ene 2026): las deudas "auto-admitidas" ligadas a IA son sobre todo tests pospuestos, adaptación incompleta y falta de comprensión del código.
Cómo contrarrestarlo:
 
1. **Reusar antes que crear.** `busca utilidades existentes antes de escribir una nueva; si hay una similar, extiéndela`.
2. **Seguir patrones existentes.** `mira HotDogWidget.php y sigue ese patrón, sin librerías nuevas`.
3. **Diffs pequeños.** 1 tarea = 1 PR revisable. Si no lo entiendes, no lo mergees.
4. **Tests primero (o junto).** `escribe un test que reproduzca el bug, luego arréglalo`.
5. **Causa raíz, no síntoma.** `arregla y no suprimas el error`.
6. **Sin código muerto ni abstracciones especulativas.** Pasada final con un agente tipo `code-simplifier`.
7. **Entender lo que entra.** Pide `explícame este bloque` antes de aceptar.
8. **Registrar decisiones.** ADRs cortos en `docs/` y enlazados desde CLAUDE.md.
9. **Detectores automáticos.** Duplicación (jscpd), complejidad, cobertura mínima y análisis estático en CI.
10. **Etiqueta lo generado** en commits cuando ayude a auditoría (`Co-authored-by`).
11. **Deuda visible.** Ningún `TODO` sin issue y dueño.
---
 
## 9. Errores comunes
 
| Error | Solución |
|---|---|
| Sesión "cocina sucia" (tareas mezcladas) | `/clear` entre tareas |
| Corregir una y otra vez | tras 2 fallos, `/clear` + mejor prompt |
| CLAUDE.md gigante | Podar; mover a hooks o skills |
| Confiar sin verificar | Tests, scripts o screenshots obligatorios |
| Investigación sin acotar | Acotar o usar subagentes |
| Perseguir cada hallazgo del revisor | Solo correctitud y requisitos |
 
---
 
## 10. Para infraestructura (Docker, K8s, CI/CD)
 
- La IA suele generar configs que funcionan pero son permisivas: revisa usuarios root, puertos, secretos, límites de recursos, `latest` en imágenes.
- Valida con herramientas, no a ojo: `hadolint`, `kubeval`/`kubeconform`, `trivy`, `terraform validate`.
- Usa **dry-run / plan** antes de aplicar (`kubectl apply --dry-run=server`, `terraform plan`).
- Nada de acciones destructivas en producción desde agente sin aprobación manual.
---
 
## 11. Extensiones: cuándo usar cuál
 
| Necesidad | Usa |
|---|---|
| Debe ocurrir siempre | **Hook** |
| Conocimiento/flujo reutilizable bajo demanda | **Skill** (`.claude/skills/`) |
| Investigar o revisar aislado | **Subagente** |
| Servicios externos (GitHub, DB, Figma) | **CLI** (`gh`, `aws`) o **MCP** |
| Contexto del proyecto, siempre presente | **CLAUDE.md** |
| Trabajo paralelo sin choques | **Worktrees** |
 
---
 
## 12. Checklist por PR
 
- [ ] ¿Entiendo cada línea que entra?
- [ ] ¿Hay tests que fallan sin el cambio?
- [ ] ¿Pasan lint, typecheck, tests y SAST?
- [ ] ¿Reusé código existente en vez de duplicar?
- [ ] ¿Sin secretos, sin dependencias dudosas?
- [ ] ¿Revisión humana (y/o subagente en contexto limpio)?
- [ ] ¿El diff es pequeño y con un solo propósito?
- [ ] ¿Actualicé CLAUDE.md/ADR si hubo una decisión nueva?
---
 
## Rutina semanal (15 min)
 
1. Podar CLAUDE.md (`/doctor`).
2. Convertir correcciones repetidas en hooks o skills.
3. Revisar métricas: duplicación, cobertura, deuda abierta.
4. Revisar `claude --version` y notas de cambios para nuevas funciones.
---
 
## Fuentes
 
- Anthropic, Best practices for Claude Code: https://code.claude.com/docs/en/best-practices
- Debt Behind the AI Boom (arXiv, mar 2026): https://arxiv.org/abs/2603.28592
- GenAI-Induced Self-Admitted Technical Debt (arXiv, ene 2026): https://arxiv.org/abs/2601.07786
- LeadDev, How AI-generated code compounds technical debt (GitClear): https://leaddev.com/technical-direction/how-ai-generated-code-accelerates-technical-debt
- IOActive, The Security Gap in AI-Generated Code: https://www.ioactive.com/the-security-gap-in-ai-generated-code/
- Veracode GenAI Code Security Report (citado vía bsscommerce): https://bsscommerce.com/services/blog/vibe-coding-security/
- Claude Code subagents guide: https://tembo.io/blog/claude-code-subagents
- Claude Code 9-step loop: https://mer.vin/2026/07/claude-code-9-step-loop-explore-plan-hooks-and-senior-style-review/
- CLAUDE.md best practices: https://memx.app/blog/claude-md-best-practices/
> Nota: los porcentajes varían según estudio y metodología. Verifica comandos y sintaxis de hooks contra la documentación vigente, porque Claude Code cambia rápido.
 
