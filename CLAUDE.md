# CLAUDE.md — bootcamp-notas

Repo personal de repaso del bootcamp Cloud/DevOps (instructor Joan Amengual, blockstellart.com). Fuente del contenido: PDF oficial del curso (uso personal, no redistribuible) — el contenido de cada módulo debe redactarse con **frase propia**, sin copiar ni parafrasear de cerca el PDF.

## Estructura de cada módulo

- Una carpeta por módulo: `modulo-XX-nombre/README.md`, basada en `_plantilla.md`.
- Secciones fijas: **Resumen**, **Comandos clave**, **Notas y gotchas**, **Recursos**.
- Estilo del Resumen: **esquemático** (bullets/tablas cortas), no prosa larga.
- Referencias cruzadas entre módulos: enlaces markdown relativos, `[Módulo XX — Tema](../modulo-XX-nombre/README.md)` — nunca `[[wikilinks]]` (no renderizan en GitHub).

## Comandos clave

- Solo se incluyen si aportan valor real (aunque no vengan en el PDF, se pueden buscar en la documentación de AWS).
- Todo valor sustituible va entre `<placeholder>` (p. ej. `<bucket-name>`, `<region>`, `<account-id>`) — nunca nombres de ejemplo concretos ni ARNs inventados con formato realista.
- Antes de darlos por buenos, verificar la sintaxis con AWS CLI real usando credenciales dummy (`AWS_ACCESS_KEY_ID=dummy`, `AWS_SECRET_ACCESS_KEY=dummy`, `AWS_DEFAULT_REGION=eu-west-1`) — un fallo de auth (`AuthFailure`/`InvalidClientTokenId`) confirma que el comando es sintácticamente válido.

## Recursos

- Enlaces a documentación oficial de AWS, verificados (curl status-code) antes de incluirlos.

## Flujo de trabajo con git

- Cuando el usuario dice que ha terminado un módulo, crear la carpeta + README **solo en local**. No commitear ni hacer push todavía.
- Esperar a que el usuario revise el contenido y dé instrucción explícita de subirlo.
- El mensaje de commit lo dicta el usuario textualmente cada vez — no redactarlo por iniciativa propia.
- Nunca incluir el archivo `CLAUDE.txt` ni `CLAUDE.md` (suelto en la raíz, sin trackear) al hacer `git add`.
- El README raíz (`README.md`) mantiene la tabla de "Progreso" y el árbol de "Estructura" — actualizar ambos al añadir un módulo nuevo, dentro del mismo push.
