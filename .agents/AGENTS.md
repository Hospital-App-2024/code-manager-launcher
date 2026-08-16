# Reglas de Proyecto: Code Manager

Estas reglas aplican globalmente a este workspace y deben ser respetadas obligatoriamente por cualquier agente que modifique el código:

1. **Gestor de Dependencias Exclusivo**: Queda estrictamente prohibido utilizar `npm`. Si se requiere instalar dependencias, se debe utilizar `pnpm` exclusivamente.
2. **Respetar Arquitectura Unificada**: No se deben crear tablas nuevas ni endpoints aislados para futuros tipos de códigos de emergencia. Se debe seguir el patrón de "Single Table Inheritance" centralizado en `EmergencyCode` y los controladores/formularios genéricos.
3. **Mantenimiento de Documentación**: Toda modificación estructural, adición de variables de entorno o cambio de paradigma que el agente realice, DEBE estar reflejado en el archivo `DOCUMENTATION.md` ubicado en la raíz del proyecto.
4. **Skills Obligatorias**: Para actualizar la documentación de manera eficiente, invoca tu skill local `doc-updater`.
