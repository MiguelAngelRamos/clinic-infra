# clinic-infra

Repositorio de infraestructura de la **Clinic App** — configuración de Kong y manifiestos K8s.

## Estructura

```
clinic-infra/
├── kong/
│   └── kong.yaml          ← configuración declarativa de Kong (referencia)
└── k8s/
    └── kong.yaml          ← manifiestos K8s: ConfigMap + Deployment + Service + Ingress
```

## Arquitectura del tráfico

```
Navegador
    │
    │ HTTP :8000 (port-forward en el lab)
    ▼
Ingress Nginx
    │
    │ enruta /auth /users /patients /specialties /doctors /appointments
    ▼
Kong Gateway :8000
    │
    ├── Valida JWT (plugin jwt)
    ├── Inyecta X-User-Id, X-User-Role, X-User-Email (plugin request-transformer)
    ├── Rate limiting por ruta (plugin rate-limiting)
    ├── Correlation ID — X-Request-ID (plugin correlation-id)
    └── CORS (plugin cors)
    │
    ├── /auth/*          → auth-service:3001
    ├── /users/*         → user-service:3002
    ├── /patients/*      → patient-service:3003
    ├── /specialties/*   → doctor-service:3004
    ├── /doctors/*       → doctor-service:3004
    └── /appointments/*  → appointment-service:3005
```

## Orden de despliegue completo

### Prerequisitos

El clúster Kind debe estar corriendo con Ingress Nginx y Metrics Server instalados.
El namespace `clinic` debe existir.

### Paso 1 — Crear todos los Secrets

```bash
# auth-service
kubectl create secret generic auth-postgres-secret \
  --namespace clinic \
  --from-literal=POSTGRES_USER=auth_user \
  --from-literal=POSTGRES_PASSWORD=<PASSWORD> \
  --from-literal=POSTGRES_DB=auth_db

kubectl create secret generic auth-valkey-secret \
  --namespace clinic \
  --from-literal=VALKEY_PASSWORD=<PASSWORD>

# JWT_SECRET — se usa también en Kong para validar tokens
JWT_SECRET=$(openssl rand -base64 48)
kubectl create secret generic auth-jwt-secret \
  --namespace clinic \
  --from-literal=JWT_SECRET=$JWT_SECRET \
  --from-literal=JWT_REFRESH_SECRET=$(openssl rand -base64 48) \
  --from-literal=SWAGGER_PASSWORD=<PASSWORD>

# Kong necesita el mismo JWT_SECRET que usa auth-service para firmar
kubectl create secret generic kong-jwt-secret \
  --namespace clinic \
  --from-literal=KONG_JWT_SECRET=$JWT_SECRET

# user-service
kubectl create secret generic user-postgres-secret \
  --namespace clinic \
  --from-literal=POSTGRES_USER=user_svc_user \
  --from-literal=POSTGRES_PASSWORD=<PASSWORD> \
  --from-literal=POSTGRES_DB=user_db

# patient-service
kubectl create secret generic patient-postgres-secret \
  --namespace clinic \
  --from-literal=POSTGRES_USER=patient_svc_user \
  --from-literal=POSTGRES_PASSWORD=<PASSWORD> \
  --from-literal=POSTGRES_DB=patient_db

# doctor-service
kubectl create secret generic doctor-postgres-secret \
  --namespace clinic \
  --from-literal=POSTGRES_USER=doctor_svc_user \
  --from-literal=POSTGRES_PASSWORD=<PASSWORD> \
  --from-literal=POSTGRES_DB=doctor_db

# appointment-service
kubectl create secret generic appt-postgres-secret \
  --namespace clinic \
  --from-literal=POSTGRES_USER=appt_svc_user \
  --from-literal=POSTGRES_PASSWORD=<PASSWORD> \
  --from-literal=POSTGRES_DB=appointment_db
```

### Paso 2 — Desplegar los microservicios (por orden de dependencia)

```bash
# 1. auth-service (sin dependencias)
kubectl apply -f ../clinic-auth-service/k8s/auth-service.yaml

# 2. user-service (sin dependencias)
kubectl apply -f ../clinic-user-service/k8s/user-service.yaml

# 3. patient-service (depende de user-service)
kubectl apply -f ../clinic-patient-service/k8s/patient-service.yaml

# 4. doctor-service (depende de user-service)
kubectl apply -f ../clinic-doctor-service/k8s/doctor-service.yaml

# 5. appointment-service (depende de patient y doctor)
kubectl apply -f ../clinic-appointment-service/k8s/appointment-service.yaml
```

### Paso 3 — Desplegar Kong

```bash
kubectl apply -f k8s/kong.yaml
```

### Paso 4 — Ejecutar migraciones

En el laboratorio, las migraciones se corren manualmente desde cada servicio
antes de que empiece a recibir tráfico:

```bash
# Ejemplo para auth-service — lo mismo para cada servicio
kubectl exec -n clinic deployment/auth-service -- \
  node -e "require('./dist/database/migrations/run')" || \
  echo "Ejecutar migraciones desde el entorno de desarrollo"
```

En un pipeline CI/CD profesional, las migraciones se corren como un
Job de K8s antes del Deployment principal.

### Paso 5 — Verificar todo

```bash
# Ver el estado de todos los Pods
kubectl get pods -n clinic

# Verificar que Kong arrancó correctamente
kubectl logs -n clinic deployment/kong

# Probar el flujo completo
# 1. Registrar usuario
curl -X POST http://192.168.1.28:8000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@clinic.com","password":"Test1234"}'

# 2. Login
curl -X POST http://192.168.1.28:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@clinic.com","password":"Test1234"}'

# 3. Usar el accessToken en rutas protegidas
curl http://192.168.1.28:8000/users/health
curl http://192.168.1.28:8000/patients/health
curl http://192.168.1.28:8000/specialties/health
curl http://192.168.1.28:8000/appointments/health
```

## Actualizar la configuración de Kong

Cualquier cambio en `kong/kong.yaml` requiere:

```bash
# 1. Actualizar el ConfigMap
kubectl create configmap kong-config \
  --from-file=kong/kong.yaml \
  --namespace clinic \
  --dry-run=client -o yaml | kubectl apply -f -

# 2. Reiniciar Kong para recargar la configuración
kubectl rollout restart deployment/kong -n clinic

# 3. Verificar que Kong arrancó correctamente
kubectl rollout status deployment/kong -n clinic
kubectl logs -n clinic deployment/kong --tail=50
```

## Nota importante: Kong y el secreto JWT

Kong necesita el **mismo** `JWT_SECRET` que usa `auth-service` para firmar los tokens.
Si rotas el secreto JWT en `auth-service`, debes actualizar también `kong-jwt-secret`
y reiniciar Kong.

El flujo es:
```
auth-service firma el token con JWT_SECRET
     ↓
Cliente envía el token en Authorization: Bearer <token>
     ↓
Kong verifica la firma con el mismo JWT_SECRET
     ↓
Si la firma es válida → inyecta headers y enruta
Si es inválida → rechaza con 401
```
