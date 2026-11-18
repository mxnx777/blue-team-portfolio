# Migración de file server con modelo de mínimo privilegio

**Tipo:** proyecto real en entorno laboral (versión anonimizada)  
**Período:** abril – julio 2026  
**Alcance:** servidor de archivos SMB del área administrativa, shares, permisos NTFS y cuentas locales

## 1. Objetivo

Migrar el servidor de archivos a infraestructura temporal **sin perder permisos ni continuidad**, reconstruir el servidor físico desde cero y devolverlo a producción con un modelo de permisos corregido, siempre con rollback disponible.

## 2. Situación inicial (baseline)

Antes de tocar nada se exportó evidencia: hostname, configuración IP, `systeminfo`, listado de shares, permisos de cada share, usuarios locales y miembros de `Administrators`.

Problemas detectados:
- Permisos asignados **directamente a usuarios**, sin grupos.
- Usuarios funcionales con **Full Control**.
- Cuentas sin uso vigentes.

## 3. Estrategia de migración

1. Exportar permisos y shares actuales.
2. Crear VM temporal (`SRV-FILE-TEMP`).
3. Copiar datos con `robocopy` preservando ACL.
4. Validar accesos con usuarios de prueba.
5. Asignar la IP productiva a la VM (los usuarios no notan el cambio).
6. Reconstruir el servidor físico.
7. Retornar los servicios al físico.
8. Mantener rollback disponible en todo momento.

## 4. Plan de rollback

| Escenario | Acción |
|---|---|
| Usuario sin acceso a share crítico | Restaurar permiso anterior desde el export, validar acceso |
| Falla la VM temporal | Apagar VM, reactivar servidor original, restaurar IP productiva |
| Copia de datos incompleta | Mantener origen operativo, repetir `robocopy` incremental |
| Cambio de permisos genera impacto | Reaplicar export previo, validar usuarios críticos |

**Requisitos previos:** export de shares y permisos, backup reciente, ventana de trabajo definida, usuario de prueba.

## 5. Modelo de permisos implementado

**Convención de grupos:** `GG_<SHARE>_<RW|RO>`, por ejemplo `GG_CONTRATOS_RW` o `GG_ADQUISICIONES_RO`.

**Principios:**
- Los usuarios **nunca** reciben permisos directos; siempre a través de grupos.
- Full Control solo para `Administrators` y `SYSTEM`.
- Usuarios funcionales: `Modify` (RW) o `Read` (RO).

**Diseño:** un share central con permisos de share amplios, y la seguridad real aplicada en **NTFS**:
- Herencia deshabilitada en cada carpeta de área.
- Cada carpeta con sus grupos `GG_*_RW` y `GG_*_RO`.

Se valida con:

```powershell
Get-SmbShare
Get-SmbShareAccess -Name <SHARE>
icacls <ruta_carpeta>
```

## 6. Depuración de cuentas

Se construyó una matriz de usuarios × grupos que **revisó y aprobó la jefatura** antes de aplicarla: qué cuentas mantener, cuáles crear y cuáles deshabilitar.

## 7. Qué aprendí

- El acceso efectivo es la combinación de los permisos SMB y NTFS: gana el más restrictivo.
- Con permisos por grupo, agregar o quitar a una persona es un cambio de membresía, no una auditoría carpeta por carpeta.
- Una migración sin rollback probado es una apuesta.

## Pendiente de documentar

- El comando `robocopy` exacto (con sus parámetros) no quedó registrado en la documentación original.
