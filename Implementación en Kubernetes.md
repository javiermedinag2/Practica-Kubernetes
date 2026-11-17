# Implementación en Kubernetes

El manifiesto [`k8s/biblioteca.yaml`](./k8s/biblioteca.yaml) crea los recursos de la aplicación Biblioteca en el namespace `practica`: una base MongoDB con almacenamiento persistente y una aplicación web Flask.

## Convenciones del manifiesto

Cada `---` separa un recurso YAML independiente. En cada recurso:

- `apiVersion` indica la versión de la API de Kubernetes que define el recurso.
- `kind` indica el tipo de recurso.
- `metadata.name` es su nombre, y `metadata.namespace` lo sitúa en `practica`.
- Se omiten campos que Kubernetes completa con sus valores predeterminados: por ejemplo, las réplicas de los controladores quedan en `1`, los servicios son `ClusterIP`, la política de descarga para imágenes con etiqueta `1.0` es `IfNotPresent`, y el puerto destino de cada servicio coincide con su puerto publicado.

## Namespace `practica`

El recurso `Namespace` crea el espacio lógico `practica`. Los servicios, controladores y pods de la aplicación quedan aislados allí de los que pertenecen a otros namespaces.

## Servicio `mdb`

El servicio `mdb` ofrece el puerto TCP `27017` y selecciona los pods con la etiqueta `app: mdb`. `clusterIP: None` lo hace headless: no asigna una IP virtual y permite que Kubernetes publique el DNS estable `mdb` para el pod de MongoDB. Tanto la aplicación como el contenedor de espera usan ese nombre.

## StatefulSet `mdb`

El `StatefulSet` administra MongoDB, cuya identidad estable y almacenamiento persistente lo distinguen de un pod efímero:

- `serviceName: mdb` vincula el StatefulSet al servicio headless y a su identidad DNS.
- `selector.matchLabels` y `template.metadata.labels` deben coincidir: determinan qué pods administra.
- El contenedor `mongodb` usa la imagen local `biblioteca-mongo:1.0`, construida desde `bd/Dockerfile`.
- `volumeMounts` monta el volumen `mongo-data` en `/data/db`, la ubicación de datos de MongoDB.
- `volumeClaimTemplates` solicita un volumen de `1Gi` con acceso `ReadWriteOnce`, que permite montarlo en modo lectura/escritura desde un nodo.
- `startupProbe` ejecuta `mongosh` para comprobar que MongoDB responde. `failureThreshold: 30` permite que el arranque tarde; `timeoutSeconds: 10` da tiempo a completar cada comprobación. Mientras la comprobación de inicio no tenga éxito, Kubernetes no ejecuta la comprobación de vida.
- `readinessProbe` usa el mismo `ping` para indicar cuándo MongoDB puede recibir tráfico.
- `livenessProbe` reinicia el contenedor si MongoDB deja de responder. Los periodos de sondeo y el umbral de fallos no se especifican, por lo que Kubernetes usa sus valores predeterminados.

## Deployment `biblioteca-web`

El `Deployment` mantiene el pod de Flask:

- Su selector y las etiquetas de la plantilla usan `app: biblioteca-web` para asociar el pod con el Deployment y con su servicio.
- El contenedor `web` usa la imagen local `biblioteca-web:1.0`, construida desde `web/Dockerfile`, y escucha en el puerto `8080`.
- El `initContainer` `wait-for-mongodb` ejecuta Python antes de iniciar Flask. Intenta abrir una conexión TCP a `mdb:27017` hasta 60 veces, con una pausa de dos segundos entre intentos. Si MongoDB aún no está disponible tras el último intento, el contenedor de inicialización falla y Kubernetes no inicia la aplicación; esto evita que Flask consulte MongoDB antes de que arranque.
- La `readinessProbe` consulta `/` para indicar cuándo Flask puede recibir solicitudes.
- La `livenessProbe` consulta `/` para que Kubernetes reinicie el contenedor si la aplicación deja de responder.

## Servicio `biblioteca-web`

El servicio `biblioteca-web` selecciona los pods con la etiqueta `app: biblioteca-web` y publica el puerto `8080` dentro del clúster. Al omitirse `type`, Kubernetes usa el tipo `ClusterIP`, accesible desde el propio clúster. Para probarlo desde la máquina local:

```powershell
kubectl -n practica port-forward svc/biblioteca-web 8080:8080
```

Después se puede visitar `http://localhost:8080`.

## Aplicar el manifiesto

```powershell
kubectl apply -f .\k8s\biblioteca.yaml
kubectl -n practica get pods,services,pvc
```

Al cambiar el nombre de la base de datos de `milo` a `mdb`, el StatefulSet crea una nueva identidad de pod y un nuevo PVC (`mongo-data-mdb-0`). El PVC anterior (`mongo-data-milo-0`), si existe, no se renombra ni se reutiliza automáticamente; conserva los datos previos hasta que se migren o eliminen por separado.
