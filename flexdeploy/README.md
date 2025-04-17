## Install

Sample JAMLz

First create the DB secret as below:

```
kubectl create secret generic \
  -n flexdeploy flexdeploy-db-credentials \
  --from-literal=POSTGRES_USER="fd_admin" \
  --from-literal=POSTGRES_PASSWORD="password" \
  --from-literal=POSTGRES_DB=flexdeploy_db \
  --from-literal=CONNECTION_STRING="postgresql://fd_admin:password@chris-postgres.cpiersfclgnd.us-west-2.rds.amazonaws.com/flexdeploy_db" \
  --dry-run=client \
  -o yaml | \
kubectl annotate \
  --local=true \
  -f - sealedsecrets.bitnami.com/managed=true \
  --overwrite -o yaml | kubeseal \
  --controller-namespace=sealed-secrets \
  --format yaml
```

once that reconciles and you see the secret and the DB is good you can do the rest of this. 

```
apiVersion: v1
kind: Namespace
metadata:
  name: flexdeploy
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: flexdeploy
  namespace: flux-system
spec:
  chart:
    spec:
      chart: flexdeploy
      sourceRef:
        kind: HelmRepository
        name: intralox-helm
      version: 1.1.0
  interval: 5m0s
  releaseName: flexdeploy
  targetNamespace: flexdeploy
  values:
    volumeClaim: flexdeploy
    fd:
      timezone: America/Chicago
      db:
        url: jdbc:postgresql://chris-postgres.cpiersfclgnd.us-west-2.rds.amazonaws.com:5432/flexdeploy_db
        dbSecretName: flexdeploy-db-password
        dbSecretKey: PASSWORD
        type: postgres
    extraEnv:
      - name: MAX_ACTIVE_CONNECTIONS
        value: "10"
      - name: MAX_IDLE_CONNECTIONS
        value: "5"
      - name: MAX_WAIT_TIME_MILLIS
        value: "10000"
--- 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: flexdeploy
  namespace: flexdeploy
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: kinda.rocks/v1beta1
kind: Database
metadata:
  name: flexdeploy-db
  namespace: flexdeploy
spec:
  secretName: flexdeploy-db-credentials
  instance: rds
  deletionProtected: true
  backup:
    enable: false
    cron: "0 0 * * *"
---
kind: Service
apiVersion: v1
metadata:
  name: flexdeploy-service 
  namespace: flexdeploy
spec:
  selector:
    app: flexdeploy
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: flexdeploy-ingress
  namespace: flexdeploy
  annotations: 
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/tls-acme: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: flex.chris.dev-k8s.intralox.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: flexdeploy-service
            port:
              number: 8080
  tls:
  - hosts: 
    - flex.chris.dev-k8s.intralox.com
    secretName: flexdeploy-ingress-tls
---
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: flexdeploy-db-credentials
  namespace: flexdeploy
spec:
  encryptedData:
    CONNECTION_STRING: AgBJ60LwPGEHXYAcKRtriV55KQudZ1QblIqfqaFLrfMqXj9sFyRk3aSv/7qQHLoY2QmSAFSWVT1BrVSzhOm6ZyuYvVzMLTYmOjJh/SuLJDgo1h8Wir5GKfz+AW6y0aZ3ykrotRJMmRSO0FOHlIp6kKd4KcqPcmUX1/zGeev+ri/MHaCQMi2+wV5IVBRS/Eqp9nYbxgkV/F7yvFqVGPUm6VEl2ZUJIc0NAtEWcqT5UKQtN+DFFjG+KmlOHhWoAavPm1jj1YeA3eigbOotVA8e3YL+lFbYCunAEkCeWf0K01lruIOPxaha3wAqNprZfhQH69pxFUdh6v2RB8f578cfNqQYGfdzlmOczfZlaNQtH+hJPPQ9PN9oDp0I8Tj8r/XxTh3XAr0Y9QWZ2vH3PjRqqwvGb4NBTN8P+6Xe3ue/mbI6mPSCD9OSwd2GmPyEjNZvZW0P8OjFX7in7qujdrAYFap44AdE238aXmOVxBH4rrq/jiBxybhFr3RvPFM7lFOgM4UvUN2JWSpwISS6RNjuTVdcy3HdWgLByNjFRIZy5Q07dKcBEyDf5rmt6KDM/+gHY7rayv5KvSuJu/oems9uDapJ239GwLiMYwxA/UKqahljmwnzzDEl4wwpgHkQlJbLbHmt1EGOkAjf1ag8Dqu0wN5Aj57jUL83lV6AXrXHZ82WX6UCaJSzbmYfduWp/T+PZLc7CgkYWzlsO0CJTLboFpt+47ZqNOOm4knE0xcksxt5ItmpeZsOkbrI++22Nhd3tqLATjnX9FGSel9MVJcArpKZ57puNIfBQapek08homiobM3nH8GgLfVsZYTdUfhy/wx2N4G0QlVM3e4LB3pzC+RR4ZIu27TlKu0mOA8=
    POSTGRES_DB: AgDOQ/xe8XDYxjmUrhV1Jic8GkrlJ6Gxe1U8A/ZAb/D+cnfILNXqZ6HQT6+6mZCSDet2ii71R8nBpgesqPinXG40GrrMpqHPuiIKgbiEtHQN4c7w1GYOxMiiNdDxp4SwZV8p9lJH3CjiIolBdc1JdLS+jo1aydJhepkCi5oTybNg0jUwgpbRy2vf/6fAXxDOFxpYoYnQXojcs0E4dbEjURucSOvRd5h4CFShCJq7x8CvgKTgjWb7gCNzLxKIqoaqM7mtcR2CUxgrlyH6F1xHClkmTPwYU3BpjdGSD2KK9li2MtEKJ4qyud7dp3a3MlJtiaOQXtqs5mk7x0AxTLxRbAn0nj1/L6B2m8NVGwJc1Wwt+BifO9S+6q3joEKCiB2C+2ONZeAYX6teWNESD2JrUoHDe3Vs5mBSdwoUX66qaDarAUOP8CWiuQ3EcTzraVyCrPmtuS8gE5HGFZ/mhzYS8CDUkoK9KV7p33Q3Is1oAgzHWBwn3oTU8mGzt15t8H03MiJ6d8O7y3j55TvMSQnd/40HbEO4Myr4XQFqiuqo8/NDxDnhUqMmMtMScr52dvFLdPYa3+Irh5Eh4mlE2KNIB16BQcVCqzX47OxvA7JlHi9vEQ8URCxU7OQc7QPtWydGei7vGinw1oPOtrNAYIQ1t+6B9hcKbOM9qlz9iNLpMlEavY6CjormhtWTiLXBqUgVNHm77LokTgvE+o4wL7Qy
    POSTGRES_PASSWORD: AgCFQDvjUxnWACINjPPBfdAvWZKWImq+Mki4Q4AK5BSbrxMiOsICDtYEVEjh65WhjBco4mFHtNwsZiUa8tbt9O0tEslbVphOCRSlQcCbmXH7tV5LCOoQ55F3Z5ZDcfvakMW3cyXaDx4+TZ4Xu+SLnfJA+SI4fyDeGCvV7fyI8HI+eJexwFiiVayV0/h+TWSAAC7+K6IeoibcB2zbLm/0AXG/ljZLYXIjqvyXtidDwirDxrjYC96mSVHmZ0aVrFu4mZcGeCjJNiCnSV+ziw7QGmioXK1XZEkq29a4f8ayZxt2zey+kwDlUyMbYMyNb2bfdx2YFQpnpQSavraEtP29QW8HshUJh6f3pxY+pB5wGkPY/vooYf6lGxiYUJGRxc5WzIwLNaaUbmM5YURsNDSlP9WFuZ7F8jmp8GlroYQuvavIEj/8MIsA7Px6tk8F2VCBUmv/LGAylgLRbkww0TRfz5mo4lRBUu8+//d1qhj9Mrn+ahZEMrRR3foOlcxMxLOJgfqOpMZ1Wf1kACJ0cA9s5KYJqIblhp9CNyfmuAhcfusb52h8gWUnIWIiQNJHOUpx4W48rO+MA2j9GF2nNOquGUHiRoN7ltO8DebxOTb78qowTefMmcBT5zHSnLgeLBK0zUoSDCj38K5u1jKFMiw88SaV7Lrstn6rFVUOrmex6wpiaQ89asxS0gM7QjVvM/0WcqjM0tqcaCifK7d5pPFd6WdN7/1AdrX0TT4acUV12cjE
    POSTGRES_USER: AgDMGnOkJYqHHMW25958DtNrxgKHniKCJ2NPOprlYOfwma2wHoRy4Q+GbVmYJ5kmX2W6sCWo6oq3jxHixkCaO1y45oMmWNQeJE5PrYBoURTqANGNvuQ9IHOB7/HcKjASgbU7I4qnFbOFnw0cyzOT4VLvktMFP+YIp/8fWFG1KC80b81pDknyRdHQJ1T1dGi4X70gJWjRL3vTsCPH6SdMQ3ApD9LRWkzdy0HX1SgSCPjc9uCKXcgdUoJGGbLs9y11LGtbGeQkpq5U/Zsf2rD4AEpAU2FgVWTw59jyPhqwuaf9HGcJ9TODLZ5cj8zoyTl5KefZxH8AZxvK2QShEsX5m0lin9Gepd7EL2FJ8Kc6uwR7uOIV6f5md3QSSR1+Ws5LHek5nOc8fRHuBrOkY/G6XEHyPBKGWS9NiSYfTzWArUnsUvKpAvXkuqJ7kK7MsRipj3WZpkOUjy1WddjtC/bzuKTFE5F/tcjNaP3dj/9X7BP6vA2Xt87rjhGogg0xm2OsfaDLYDdoHmEcdkMKs5YLXg/Pun5A0e/4P4uXvjYEULF2SlHQ+4jRGQhcgMsVXNF9Tv0fyxbDXzcEwaxAiaaCk2KKiH3xBWPA/GKyNxrOv8SDnLGWAvdYwnSfSHU6/q3UEYh6tnDQXTObtLte9mpal9y/JLx/HjwOovccPilOs1Fg30tjsyTeiizeDExch6TvGYktrjoxnZFhhA==
  template:
    metadata:
      annotations:
        sealedsecrets.bitnami.com/managed: "true"
      creationTimestamp: null
      name: flexdeploy-db-credentials
      namespace: flexdeploy
---
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: flexdeploy-db-password
  namespace: flexdeploy
spec:
  encryptedData:
    PASSWORD: AgBwwqwPaOrcubReeRnyLWNU3rLpc/sSbqLMAI/ld+i6+BUsfr2C/8pom9wHQurd+HR2dwXSUm5m8wTmzatgluwoASQIiRpg8W4zYXbK6FmMbNAhag4BzdtMUVV/saOgfmATb2LE+YRc4Mv+hOAqWmNguEf4hj3Xf5OINQN8/wdUxsvm6wAui6WKeuHfO5wO70JannZ0Gn2yjcmjWavVzDq7YqzLGDTJFx9aVz2VhaFZtkQdV7LCD2trZm6C6N4asxVSPXs8VUy7E4hXUKjA2m65GGbUw3HST2FaB05UchxYsb2h80j63IGhTIKigoDQjqQg3VtJi+cNYd3k579xSBuCo72nfVmeQ35GfXJHAnQaKQqk1VXS+6i37MldC6m55F1PxcTUOVB0LeeRr7h4aboIDja0hyQFgf844CB2QW0txMpaTrfTPoagcm0EW7FR7EL6ziOAks+anUlKdtapf9EdI9i8IAU8JsyZfziE/zgiM5QtDzlRUONcTpHj+ZpM2Vb8WVR0nOTMdlLns0GsckhWAGX6guJfBz96wTMBvX0QfPFuHNVNqTNgl/4Nu1/Dlo2aScy3ed+8cMHc2uDNRcs1wozrzW/B8OHCw5f5ahkVCbr8H6vmGxrSFvO0ZO4c0D+FHfVPJIAC4YpsmtZvjOklsHG3jATFKxeR5PDJaftp2IEDO2SBcKU+EbcxHgMjZs2a4Sl+IjQzXqSV2GHC3uZ0dqzwfPYYGJ66QZHytDbB
  template:
    metadata:
      annotations:
        sealedsecrets.bitnami.com/managed: "true"
      creationTimestamp: null
      name: flexdeploy-db-password
      namespace: flexdeploy
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-utils
  namespace: flexdeploy
  labels:
    app: k8s-utils
spec:
  replicas: 1
  selector:
    matchLabels:
      app: k8s-utils
  template:
    metadata:
      labels:
        app: k8s-utils
    spec:
      containers:
        - name: k8s-utils
          # This tag contains the pgloader app
          # https://pgloader.readthedocs.io/en/latest/
          image: npdsoftwaredev/k8s-utils:pgloader-v3.6.9
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

Once thats running you need to load the db. You can do that from the k8s-utils pod. more directions and the file you will need is [here](../db-scripts/readme.md)