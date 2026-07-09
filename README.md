#  Documentación Técnica

**Asignatura:** AUY1104 — Ciclo de Vida del Software II
**Evaluación:** Evaluación Final Transversal (EFT) — Encargo
**Caso de negocio:** Operación Resiliencia en TechMarket

---

## 1. Precisión sobre la infraestructura utilizada

El enunciado del encargo usa como referencia **Amazon EKS** y **Amazon ECR**. Este proyecto resuelve los mismos requisitos usando la infraestructura trabajada durante el semestre, que es funcionalmente equivalente:

| En el enunciado | Se usó en este proyecto | Por qué es equivalente |
|---|---|---|
| Amazon EKS | Clúster Kubernetes **K3s** sobre EC2 (Ubuntu) | K3s implementa la misma API estándar de Kubernetes (Deployments, Services, Probes, rollout/rollback). Todo manifiesto o comando `kubectl` funciona igual. |
| Amazon ECR | **Docker Hub** (`valochoa09/mi-api`) | Ambos son registries de imágenes Docker; el pipeline solo cambia el destino del `docker push`. |

No fue necesario aprovisionar EKS ni ECR reales: la lógica de plantillas reutilizables, Blue-Green y rollback automático es idéntica sobre cualquier clúster Kubernetes conformante.

---

## 2. Arquitectura general

```
                     ┌─────────────────────────────┐
                     │   GitHub Actions (CI/CD)     │
                     │                              │
  git push tag ──────▶  1. deps-and-test            │
  workflow_dispatch     2. build-and-push (Docker)  │
                     │  3. deploy-to-k8s (SSH)      │
                     └──────────────┬───────────────┘
                                    │ SSH
                                    ▼
                     ┌─────────────────────────────┐
                     │   EC2 (Ubuntu) + K3s          │
                     │                              │
                     │  ┌────────────┐ ┌───────────┐│
                     │  │ demo-api-  │ │ demo-api- ││
                     │  │   blue     │ │   green   ││
                     │  │ (estable)  │ │  (nueva)  ││
                     │  └─────┬──────┘ └─────┬─────┘│
                     │        │              │      │
                     │   demo-api-service     │      │
                     │   (selector dinámico) │      │
                     │        │       demo-api-green-│
                     │        │       preview (interno)│
                     └────────┼──────────────┼───────┘
                              ▼              ▼
                        Tráfico real    Validación de
                        de usuarios     salud (pre-switch)
```

**Repositorios:**
- `AUY1104-K8s-Lab` → plantilla reutilizable (`deploy-api.yaml`), invocada vía `workflow_call`.
- `-MiProyecto-App` → código fuente de la API, manifiestos Kubernetes (`k8s/`), y el workflow "caller" (`client.yaml`) que dispara la plantilla con los parámetros del proyecto.

---

## 3. Estrategias de despliegue: comparación

| Estrategia | Downtime | Rollback | Costo infraestructura | Complejidad | Uso recomendado |
|---|---|---|---|---|---|
| **All-in-once** | Alto (corte total durante el despliegue) | Manual, lento | Bajo (1 set de recursos) | Muy baja | Entornos de desarrollo, sin SLA |
| **Rolling Update** | Bajo, pero con mezcla de versiones simultáneas | Posible pero gradual, revierte réplica por réplica | Bajo | Media | Servicios tolerantes a versiones mixtas temporales |
| **Canary** | Muy bajo | Rápido si se detecta pronto (solo % pequeño afectado) | Medio (requiere control fino de tráfico, ideal con Ingress/Service Mesh) | Alta | Cambios riesgosos, se quiere validar con tráfico real parcial |
| **Blue-Green** (elegida) | Ninguno (corte instantáneo de selector, sin mezcla) | **Inmediato**: revertir selector + `rollout undo` | Medio-alto (doble set de pods corriendo en paralelo) | Media | Servicios críticos donde se necesita rollback instantáneo y 100% de certeza sobre qué versión sirve tráfico |

### Justificación de la elección (Blue-Green)

Para el caso "TechMarket Orders" —un microservicio de órdenes de un e-commerce— el riesgo de un despliegue fallido es alto (pérdida de ventas, órdenes corruptas). Blue-Green se eligió porque:

1. **Aislamiento total**: `demo-api-green` recibe la nueva versión y se valida en `demo-api-green-preview` (Service interno, sin tráfico real) antes de exponerse. Nunca hay usuarios sirviéndose de una versión a medio probar.
2. **Rollback atómico**: revertir es un cambio de selector de Service (operación de milisegundos), no un proceso gradual como en Canary o Rolling Update.
3. **Cero mezcla de versiones**: en Rolling Update o Canary, dos versiones del microservicio sirven tráfico simultáneamente — riesgo real para un servicio de órdenes si hay cambios de esquema o de lógica de negocio entre versiones.

El costo (mantener el doble de réplicas temporalmente) se consideró aceptable frente al riesgo evitado.

---

## 4. Cómo funciona el pipeline

**Repo `AUY1104-K8s-Lab` → `deploy-api.yaml` (plantilla reutilizable):**

1. **`deps-and-test`**: `npm install` + `npm test` (Jest/Supertest) antes de construir nada.
2. **`build-and-push`**: construye la imagen Docker y la sube a Docker Hub con el tag correspondiente.
3. **`deploy-to-k8s`**:
   - Conecta por SSH a la instancia EC2/K3s.
   - Aplica los manifiestos base (`kubectl apply -f k8s/`).
   - Actualiza **solo** la imagen de `demo-api-green` (`kubectl set image`) — blue nunca se toca en este paso.
   - Inyecta variables de entorno dinámicas si se especifican (`kubectl set env`).
   - Espera el rollout de green (`kubectl rollout status --timeout=120s`).
   - Ejecuta la **Validación de Salud**: un pod temporal de `curl` consulta `demo-api-green-preview:3000/health` y valida código HTTP 200 y latencia < 2s.
   - Si la validación pasa → cambia el selector de `demo-api-service` a `green` (tráfico real).
   - Si falla cualquier paso anterior → se activa el **Rollback Automático**.

**Repo `-MiProyecto-App` → `client.yaml` (caller):**

Invoca la plantilla anterior pasándole los parámetros específicos del proyecto (nombre de imagen, IP del servidor, namespace, variables de entorno). Se dispara por `push` de tags `v*.*.*` o manualmente vía `workflow_dispatch`.

---

## 5. Mecanismo de Remediación Automática (Rollback)

### Flujo de detección y remediación

```
Detección (Health Check)
        │
        ▼
¿Código HTTP == 200 y latencia < 2s?
        │
   ┌────┴─────┐
   │           │
  Sí          No
   │           │
   ▼           ▼
Cambiar    ROLLBACK AUTOMÁTICO:
tráfico a  1. Revertir selector del Service a "blue"
green      2. kubectl rollout undo deployment/demo-api-green
   │           │
   ▼           ▼
Notificación implícita vía estado del job en GitHub Actions
(log "FALLA: codigo HTTP X" / "Salud OK" + conclusion del run)
```

### Implementación (código real, `deploy-api.yaml`)

```yaml
- name: Validación de Salud
  run: |
    RESULT=$(ssh ... "kubectl run healthcheck-tmp-$RANDOM --rm -i --restart=Never \
      --image=curlimages/curl -- curl -s -o /dev/null -w '%{http_code} %{time_total}' \
      http://demo-api-green-preview:3000/health")
    CODE=$(echo "$RESULT" | awk '{print $1}')
    if [ "$CODE" != "200" ]; then exit 1; fi

- name: Cambiar tráfico a green
  if: success()
  run: kubectl patch svc demo-api-service -p '{"spec":{"selector":{"version":"green"}}}'

- name: Rollback automático
  if: failure()
  run: |
    kubectl patch svc demo-api-service -p '{"spec":{"selector":{"version":"blue"}}}'
    kubectl rollout undo deployment/demo-api-green
```

La condición `if: failure()` de GitHub Actions se dispara si **cualquier** paso previo del job falla — no depende de reconocer un error específico. Esto significa que el mecanismo reacciona ante **cualquier tipo de fallo** que impida que `demo-api-green` responda sano en `/health`: imagen inexistente (`ImagePullBackOff`), timeout de rollout, error 500 de la aplicación, alta latencia, o un crash del proceso (`CrashLoopBackOff`). No es necesario anticipar el error exacto — el sistema está preparado para un error "desconocido" porque el gate de decisión es el resultado del health check, no una lista de errores predefinidos.

### Escenarios de error identificados y cubiertos

| Escenario | Cómo se manifiesta | Cómo lo detecta el pipeline |
|---|---|---|
| Imagen inexistente / mal escrita | `ImagePullBackOff` | Timeout en "Esperar Rollout de green" (120s) |
| Fallo de `readinessProbe` | Pod nunca queda `Ready` | Mismo timeout de rollout |
| Aplicación responde error 500 | `/health` retorna código ≠ 200 | Step "Validación de Salud" detecta código HTTP |
| Latencia degradada | `/health` responde lento | Comparación de `time_total` > 2.0s |
| Crash loop del contenedor | `CrashLoopBackOff` | Rollout nunca completa, timeout |

### Impacto de la remediación en el negocio

- **MTTR (Mean Time to Recovery)**: con este mecanismo, el tiempo de recuperación ante un despliegue fallido baja de "minutos/horas de intervención manual" a **segundos** (el tiempo que toma el `patch` del Service + `rollout undo`), ya que no depende de que un humano note el problema y actúe.
- **Disponibilidad del servicio**: `demo-api-blue` nunca se detiene durante el proceso, por lo que el usuario final no percibe corte alguno, incluso cuando el despliegue de green falla completamente.
- **Costo operativo en AWS**: el costo adicional es mantener temporalmente el doble de réplicas (blue + green) durante la ventana de despliegue — marginal frente al costo de una caída de servicio no controlada en un microservicio de órdenes.

---

## 6. Evidencia de prueba (Prueba de Fuego simulada)

Se implementó un mecanismo de **inyección de fallo controlada** para validar el rollback antes de la defensa en vivo: un toggle por variable de entorno (`FORCE_HEALTH_FAIL`) en el endpoint `/health` de la API (`src/index.js`), activable únicamente al disparar el pipeline manualmente vía `workflow_dispatch` con el input `env_vars`.

**Resultado de la prueba (GitHub Actions):**
- Con `FORCE_HEALTH_FAIL=true`, el `readinessProbe` del pod nuevo nunca pasó → `kubectl rollout status` agotó el timeout de 120s → step "Esperar Rollout de green" falló (`Error: Process completed with exit code 1`).
- El step **"Rollback automático"** se ejecutó automáticamente (`if: failure()`), revirtiendo el Service y ejecutando `rollout undo` sin intervención manual.
- Verificación posterior en el clúster confirmó `demo-api-service` con selector `version: blue` y `demo-api-green` corriendo la versión anterior con 0 reinicios.

Esto demuestra el circuito completo: **fallo → detección → remediación automática**, sin depender de que el fallo sea uno anticipado de antemano.

---

## 7. Beneficios para el negocio

La arquitectura implementada aporta a TechMarket:

- **Reducción de errores manuales**: el despliegue, la validación y el rollback ya no dependen de que un operador ejecute pasos correctamente bajo presión — están codificados y son repetibles.
- **Aceleración de despliegues**: nuevas versiones de "Orders" pueden salir a producción con confianza, sabiendo que un fallo se revierte solo en segundos.
- **Alineación con prácticas ágiles**: al eliminar el miedo a desplegar (por el rollback automático), el equipo puede iterar con mayor frecuencia, acortando ciclos de entrega.
- **Trazabilidad**: cada despliegue queda documentado en el historial de GitHub Actions, con logs de la validación de salud y, si aplica, del rollback ejecutado.

---

## Aspectos formales

Se utilizó Claude (Anthropic) como asistente durante el desarrollo de este encargo para: diagnóstico de errores en pipelines de GitHub Actions, redacción de esta documentación técnica, y orientación en la configuración de pruebas de fallo controladas. Todo el código fue revisado, ejecutado y validado por el/los estudiante(s) en el entorno real (AWS Academy Learner Lab + K3s).
