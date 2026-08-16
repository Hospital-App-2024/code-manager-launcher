---
name: doc-updater
description: Actualiza la documentación raíz (DOCUMENTATION.md) de forma inteligente cada vez que se realice un cambio de arquitectura o una refactorización de código en el proyecto.
---

# Instrucciones de la Skill `doc-updater`

Cuando necesites utilizar esta skill porque has realizado cambios estructurales en el proyecto:

1. Lee el estado actual de `DOCUMENTATION.md` en la raíz del proyecto.
2. Identifica en qué sección encajan los nuevos cambios (Base de datos, Frontend, Stack, etc.).
3. Utiliza la herramienta de edición de archivos correspondiente para agregar viñetas explicativas o modificar párrafos existentes en el `DOCUMENTATION.md`.
4. El tono debe ser profesional, directo y centrado en informar a futuros desarrolladores (o a ti mismo) sobre el estado de la arquitectura.
5. No redactes la documentación como si fuera un historial de git (ej. "Hoy cambié esto"). Escríbelo como un manual de arquitectura (ej. "El proyecto utiliza X tecnología para resolver Y problema").
