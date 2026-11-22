# ADR-001 — Estrategia de ramas: GitHub Flow con promoción por ambientes

- **Estado:** aceptada
- **Fecha:** 06/11/2026
- **Decisores:** equipo consultor (aprobada por Rodrigo Zambrana)

## Contexto

- El equipo consultor tiene hasta tres personas y trabaja en un único monorepo (`tero-devops`) con los seis
  microservicios, la infraestructura y los pipelines.
- La letra pide ambientes Dev, Test y Prod con quality gates entre ellos, y que ninguna rama de feature se fusione sin
  la aprobación de un revisor distinto del autor.
- El auditor observó que «no hay forma de saber qué versión se está ejecutando ni de repetir un despliegue de la misma
  manera dos veces».
- Tero no tiene equipo de infraestructura propio: su gente tiene que poder operar el modelo después de la consultoría,
  así que conviene el más simple que cumpla los controles.

## Opciones consideradas

1. **Git Flow** (`main`, `develop`, `feature/*`, `release/*`, `hotfix/*`), con cada rama larga atada a un ambiente.
2. **Trunk-Based Development con feature flags.**
3. **GitHub Flow** (`main` + ramas cortas + PR), y los ambientes se recorren promoviendo un mismo artefacto.

## Decisión

Usamos **GitHub Flow**, y los ambientes se recorren por **promoción del artefacto**, no con ramas:

- `main` es la única rama larga y siempre está desplegable.
- Cada tarea se hace en una rama corta `feature/<tarjeta>-descripcion` (código de la tarjeta de Trello, por ejemplo `feature/DOCK-02-dockerfiles-go`) (o `fix/…`, `docs/…`), que entra a `main` solo por
  Pull Request con al menos una aprobación de un integrante distinto del autor y todos los checks en verde.
- Cada merge a `main` construye **una sola vez** la imagen de cada servicio, etiquetada con el SHA del commit, y esa misma
  imagen avanza: **dev** (automático) → **test** (si pasa los quality gates de calidad, seguridad y tests) → **prod**
  (aprobación manual en el GitHub Environment `prod` y tag semántico `vX.Y.Z`).
- Volver a una versión anterior es redesplegar la imagen de un SHA o tag anterior; no hace falta revertir ramas.

## Justificación

- **Tamaño y plazo del equipo:** con tres personas, mantener `develop` y `release/*` en paralelo agrega merges y
  conflictos sin agregar control. Las ramas cortas y los PR chicos se revisan más rápido.
- **Trazabilidad:** construir una vez y promover garantiza que lo que corre en prod es exactamente lo que se probó en
  test. El SHA en la etiqueta de la imagen responde la pregunta «¿qué versión está corriendo?».
- **Gates verificables:** las condiciones de promoción viven en el pipeline como código, no en la disciplina de mergear
  entre ramas.
- **Trunk-Based con flags** descartado: exige un sistema de feature flags que Tero no tiene y que no es parte del alcance.

## Consecuencias

- `main` queda protegida: PR obligatorio, una aprobación que no sea del autor (se descarta si llegan commits nuevos),
  conversaciones resueltas, sin pushes directos ni force-push, y la regla aplica también a los administradores. Los
  checks obligatorios se agregan cuando exista el pipeline (CICD-02).
- **Visibilidad del repositorio:** en un repositorio privado de cuenta personal, GitHub exige GitHub Pro para la
  protección de ramas y los Environments con revisores (respondía 403). Decisión de Rodrigo (06/11/2026): «comenzar
  público y luego se revisa lo del Student Pack». El repositorio arranca **público** y más adelante se evalúa volverlo
  privado con el GitHub Student Developer Pack.
  Rodrigo confirmó (06/11/2026): «Público», sabiendo que el repositorio original de la cátedra es privado.
- Al ser público, antes de cada push se corre Gitleaks localmente y se verifica que no haya `.env`, credenciales de AWS
  ni otros secretos; el mismo control corre en CI desde SEC-03.
- **Historial sin credenciales:** el Tero original traía contraseñas de la base, `admin/admin` y un secreto JWT por
  defecto escritos en el compose, el README y el código del admin. Antes del primer push se armó un historial donde
  nunca aparecen: la línea base es el código de `ORT-ATI-CertificadoDevOps/Tero` en el commit `56fd8cb`, ya con esos
  valores reemplazados por variables de entorno obligatorias. Gitleaks se corrió sobre todo el historial antes de publicar.
- Mientras el equipo no esté formado no hay quién apruebe un PR distinto del autor, así que no se mergea nada: el
  trabajo queda en ramas con su PR abierto.
- Los dos primeros commits de `main` (la importación de Tero sin credenciales y la estructura base) se
  hicieron directo, antes de que existiera el equipo y la protección. Desde ahí, todo cambio entra por PR.
- Un `hotfix` es una rama corta más con PR prioritario; no hay rama especial.
- El pipeline necesita la lógica de promoción (job por ambiente con `needs` y `environment`), documentada en el diagrama
  del pipeline.

## Si cambiaran las condiciones

- Con varios releases vivos en paralelo, o con clientes que necesiten mantener versiones viejas, convendría Git Flow con
  `release/*`.
- Con un equipo grande y despliegue continuo a prod, convendría Trunk-Based con feature flags.
