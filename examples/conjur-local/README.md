# Probar Conjur CSI localmente

Este laboratorio ejecuta Conjur OSS con Docker Compose y utiliza un clúster
Minikube existente para el Secrets Store CSI Driver, el proveedor CSI de
Conjur y la aplicación de prueba. No instala Conjur en Kubernetes.

> Es solo para desarrollo local. La configuración de Conjur, TLS y PostgreSQL
> no es adecuada para producción.

## Requisitos

- Docker con el plugin Compose, `kubectl`, Helm y Minikube.
- Un Minikube activo. El nombre `host.minikube.internal` debe resolver desde
  sus Pods.

## 1. Iniciar Conjur

Desde la raíz del repositorio:

```sh
cd examples/conjur-local
umask 077
printf 'CONJUR_DATA_KEY=%s\n' "$(docker run --rm cyberark/conjur data-key generate)" > .env
docker compose up -d --wait
```

El proxy HTTPS queda disponible en `https://localhost:8443` desde la máquina y
en `https://host.minikube.internal:8443` desde Minikube.

## 2. Preparar Conjur y guardar un secreto de prueba

Crea la cuenta de laboratorio y captura la API key del administrador sin
escribirla a un archivo:

```sh
account_output="$(docker compose exec -T conjur conjurctl account create default)"
admin_api_key="$(printf '%s\n' "$account_output" | sed -n 's/^API key for admin: //p')"
test -n "$admin_api_key"
docker compose exec -T client conjur init oss -u https://proxy -a default --self-signed
docker compose exec -e ADMIN_API_KEY="$admin_api_key" -T client \
  sh -c 'conjur login -i admin -p "$ADMIN_API_KEY"'
```

Crea la identidad Kubernetes autorizada y la variable que leerá la app:

```sh
docker compose exec -T client conjur policy replace -b root -f /policy/host.yml
docker compose exec -T client conjur policy load -b root -f /policy/authn-jwt.yml
docker compose exec -T client conjur policy load -b root -f /policy/db-credentials.yml
docker compose exec -T client conjur variable set \
  -i db-credentials/password -v 'conjur-csi-local-test-value'
```

Configura Authn JWT con el issuer y las claves públicas de Minikube:

```sh
issuer="$(kubectl get --raw /.well-known/openid-configuration |
  python3 -c 'import json,sys; print(json.load(sys.stdin)["issuer"])')"
jwks="$(kubectl get --raw /openid/v1/jwks |
  python3 -c 'import json,sys; print(json.dumps({"type":"jwks","value":json.load(sys.stdin)},separators=(",",":")))')"

docker compose exec -T client conjur variable set \
  -i conjur/authn-jwt/kube/public-keys -v "$jwks"
docker compose exec -T client conjur variable set \
  -i conjur/authn-jwt/kube/issuer -v "$issuer"
docker compose exec -T client conjur variable set \
  -i conjur/authn-jwt/kube/token-app-property -v sub
```

El audience `conjur` lo configuraremos en el driver CSI en el siguiente paso.

## 3. Instalar el driver CSI y el proveedor de Conjur

```sh
helm repo add secrets-store-csi-driver \
  https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm repo add cyberark https://cyberark.github.io/helm-charts
helm repo update

helm upgrade --install csi-secrets-store \
  secrets-store-csi-driver/secrets-store-csi-driver \
  --namespace kube-system \
  --set linux.providersDir=/var/run/secrets-store-csi-providers \
  --set 'tokenRequests[0].audience=conjur' \
  --wait --timeout 5m

helm upgrade --install conjur-csi-provider cyberark/conjur-k8s-csi-provider \
  --namespace kube-system \
  --wait --timeout 5m
```

## 4. Desplegar la aplicación de prueba

Obtén el certificado TLS que generó Compose. Está ignorado por Git:

```sh
docker compose cp proxy:/etc/nginx/tls/nginx.crt ./conjur-ca.crt
```

Instala el chart local. Los valores de prueba montan
`db-credentials/password` en `/mnt/secrets-store/db-password`:

```sh
helm upgrade --install conjur-csi-test ../../charts/generic-app \
  --namespace conjur-csi-test --create-namespace \
  --values test-values.yaml \
  --set-file conjur.sslCertificate=./conjur-ca.crt \
  --wait --timeout 5m
```

Comprueba que el Pod está listo y que pudo recuperar el valor de Conjur:

```sh
kubectl get pods -n conjur-csi-test
kubectl exec -n conjur-csi-test deployment/conjur-csi-test-generic-app \
  -- cat /mnt/secrets-store/db-password
```

El segundo comando debe imprimir `conjur-csi-local-test-value`. El chart monta
el secreto como archivo; no crea un Secret de Kubernetes.

Si el Pod no inicia, revisa los eventos de montaje CSI:

```sh
kubectl describe pod -n conjur-csi-test \
  -l app.kubernetes.io/instance=conjur-csi-test
kubectl logs -n kube-system -l app=conjur-k8s-csi-provider --all-containers
```

## 5. Detener y limpiar

Elimina primero la app de prueba y su namespace:

```sh
helm uninstall conjur-csi-test -n conjur-csi-test
kubectl delete namespace conjur-csi-test
```

Al terminar de usar el driver/proveedor, desinstálalos solo si ninguna otra
app del clúster los utiliza:

```sh
helm uninstall conjur-csi-provider -n kube-system
helm uninstall csi-secrets-store -n kube-system
```

Detén Conjur conservando la base de datos para la próxima sesión:

```sh
docker compose down
```

Para borrar también los datos de laboratorio y la clave local:

```sh
docker compose down -v
rm -f .env conjur-ca.crt
```
