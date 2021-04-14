## Jfrog-Artifactory Operator CR installation 

 Make sure you have Postgresql ready

```
TARGET_NAMESPACE=jfrog-artifactory

oc new-project ${TARGET_NAMESPACE}
```

```
oc adm policy add-scc-to-user anyuid -z openshiftartifactoryha-artifactory-ha
```

## Create dummy service to get an internal certificate¶
```
oc apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: jfrog
  annotations:
    service.alpha.openshift.io/serving-cert-secret-name: jfrog-cert
spec:
  ports:
  - name: service-serving-cert
    port: 443
    targetPort: 8443
  selector:
    app: service-serving-cert
EOF
```

```
KEY=$(openssl rand -hex 32)

NAMESPACE_UID=$(oc get namespace/${TARGET_NAMESPACE} -o jsonpath="{.metadata.annotations.openshift\.io/sa\.scc\.uid-range}" | cut -f1 -d'/')
NAMESPACE_GID=$(oc get namespace/${TARGET_NAMESPACE} -o jsonpath="{.metadata.annotations.openshift\.io/sa\.scc\.supplemental-groups}" | cut -f1 -d'/')

oc apply -f - <<EOF
apiVersion: charts.helm.k8s.io/v1alpha1
kind: OpenshiftArtifactoryHa
metadata:
  name: openshiftartifactoryha
spec:
  artifactory-ha:
    artifactory:
      image:
        registry: registry.connect.redhat.com
        repository: jfrog/artifactory-pro
        tag: 7.15.4-1
      joinKey: ${KEY}
      masterKey: ${KEY}
      uid: ${NAMESPACE_UID}
      node:
        replicaCount: 2
        waitForPrimaryStartup:
          enabled: false
    databaseUpgradeReady: true
    database:
      driver: org.postgresql.Driver
      password: postgresql
      type: postgresql
      url: jdbc:postgresql://postgresql:5432/postgresql
      user: postgresql
    initContainerImage: >-
      registry.connect.redhat.com/jfrog/init@sha256:895c2e0f78bdff08102ce1627fb5a2add3d6619a31cf2b9115efeeb4a9dcb519
    nginx:
      uid: ${NAMESPACE_UID}
      gid: ${NAMESPACE_GID}
      http:
        externalPort: 80
        internalPort: 8080
      https:
        externalPort: 443
        internalPort: 8443
      image:
        registry: registry.redhat.io
        repository: rhel8/nginx-116
        tag: latest
      service:
        ssloffload: false
      tlsSecretName: jfrog-cert
    postgresql:
      enabled: false
    waitForDatabase: true
EOF

```

```
oc expose svc/openshiftartifactoryha-nginx
```
