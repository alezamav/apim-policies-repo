# apim-policies-repo

Configuración de **Azure API Management** y del **ingress** del clúster AKS para el reto
**BankX Reactive Ledger** (NTT DATA · Tech Connect 2026).

Complementa a [`infra-azure-apim-webapp`](https://github.com/alezamav/infra-azure-apim-webapp),
que aprovisiona los recursos base (AKS, Cosmos DB y la instancia de APIM). Este repositorio
se encarga de la capa de exposición: instalar `ingress-nginx`, apuntar APIM al ingress y
publicar las operaciones y la policy de enrutamiento.

## Qué hace el pipeline

| Paso | Descripción |
|---|---|
| Autenticación | `azure/login@v2` con el secreto `AZURE_CREDENTIALS` |
| kubeconfig | Obtiene credenciales del AKS vía REST (`listClusterUserCredential`) |
| ingress-nginx | Instala o actualiza con Helm, servicio `LoadBalancer` y health probe en `/healthz` |
| Espera de IP | Reintenta hasta 30 veces (~5 min) hasta que el LoadBalancer recibe IP pública |
| Verificación | Comprueba que el ingress responde `200` en `/healthz` antes de continuar |
| Backend APIM | Crea o actualiza `aks-ingress-backend` apuntando a `http://<INGRESS_IP>` |
| API | Publica `cardops-api` desde `apis/cardops-api.json` |
| Operaciones | `create-transaction`, `list-transactions`, `health`, `ingress-health` |
| Producto | `cardops-product` con suscripción y lo vincula a la API |
| Policy | Aplica `policies/cardops-policy.xml` en formato `rawxml` |
| Smoke tests | `curl` contra el ingress desde el runner |

## Estructura

```
.
├── .github/workflows/
│   └── deploy-apim.yml       # Pipeline completo
├── apis/
│   └── cardops-api.json      # Definición de la API en APIM
├── policies/
│   └── cardops-policy.xml    # rate-limit + set-backend-service + rewrite-uri
└── README.md
```

## La policy

`policies/cardops-policy.xml` aplica en el inbound:

- **`rate-limit`** — 100 llamadas por cada 60 segundos.
- **`set-header Expect`** — elimina la cabecera `Expect: 100-continue`, que causa
  problemas de latencia entre APIM y algunos backends.
- **`set-backend-service`** — enruta al backend `aks-ingress-backend`.
- **`rewrite-uri`** — expresión C# que reescribe la ruta:
  - `/_ingress/health` → `/healthz` (health del propio ingress a través de APIM)
  - `/cardops/...` → quita el prefijo antes de reenviar al backend
  - cualquier otra ruta pasa sin cambios

El backend recibe `forward-request` con timeout de 60 segundos.

## Prerrequisitos

### 1. Infraestructura desplegada

El repositorio `infra-azure-apim-webapp` debe haber corrido correctamente: AKS, Cosmos DB
y APIM en estado `Succeeded`.

### 2. Providers registrados

Se registran **una sola vez**, a nivel de suscripción, con una cuenta con permisos sobre
ella. El Service Principal está acotado al resource group y no puede hacerlo:

```bash
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.ApiManagement
```

### 3. Secreto en GitHub

**Settings → Secrets and variables → Actions → New repository secret**

| Nombre | Valor |
|---|---|
| `AZURE_CREDENTIALS` | JSON del Service Principal |

Es el **único** secreto necesario. Los nombres de recursos de Azure están declarados
en el bloque `env:` del workflow, porque son configuración pública y no credenciales:

```yaml
env:
  AZ_RESOURCE_GROUP: rg-cardops-demo
  AKS_NAME: aks-cardops-demo
  APIM_NAME: apim-cardops-demo
  APIM_API_ID: cardops-api
  APIM_API_PATH: cardops
  INGRESS_NAME: ingress-nginx-controller
  INGRESS_NAMESPACE: ingress-nginx
```

Tratarlos como secretos no aportaría seguridad —son visibles en el portal de Azure y en
el repositorio de infraestructura— y sí restaría observabilidad: GitHub enmascara los
secretos en los logs, de modo que un nombre mal escrito aparecería como `***` y sería
imposible de diagnosticar. Para desplegar en otro entorno basta con editar estos valores
en el workflow.

## Ejecución

Se dispara con `push` a la rama **`develop`**, o manualmente desde
**Actions → deploy-apim → Run workflow**.

## Verificación

```bash
# Health del ingress a través de APIM
curl -i https://apim-cardops-demo.azure-api.net/cardops/_ingress/health

# Health de la aplicación (requiere los pods desplegados)
curl -i https://apim-cardops-demo.azure-api.net/cardops/actuator/health
```

## Consideraciones

- **Idempotencia** — `helm upgrade --install` y los `PUT` contra la API de management
  permiten reejecutar el pipeline sin efectos secundarios.
- **IP del LoadBalancer** — Azure puede tardar varios minutos en asignarla. El pipeline
  espera hasta 5 minutos y luego verifica que responda, en vez de asumir que estar lista
  equivale a estar operativa.
- **`subscriptionRequired`** — la API se publica con `false` mientras el producto
  `cardops-product` se crea con `true`. Al ser la configuración de la API la que decide
  en las llamadas directas, el gateway responde sin necesidad de subscription key.
- **Orden de despliegue** — este pipeline configura la ruta de entrada; los pods de la
  aplicación se despliegan desde el repositorio del servicio. Hasta entonces, las
  operaciones `/api/transactions` y `/actuator/health` devolverán `404` desde el ingress.
