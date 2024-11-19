# Instalación PostgreSQL en Kubernetes

- Archivo de deploy:

  ```yaml
  apiVersion: v1
    kind: List
    items:
    # psql-configmap.yaml
    - apiVersion: v1
        kind: ConfigMap
        metadata:
        name: postgres-secret
        labels:
            app: postgres
        data:
        POSTGRES_DB: postgres
        POSTGRES_USER: SecureAdmin
        POSTGRES_PASSWORD: ..Salo2022..
        PGPORT: "6600"
    # psql-pv.yaml
    - apiVersion: v1
        kind: PersistentVolume
        metadata:
        name: postgres-volume
        labels:
            type: local
            app: postgres
        spec:
        storageClassName: manual
        capacity:
            storage: 150Gi
        accessModes:
            - ReadWriteMany
        hostPath:
            path: /data/postgresql
    # psql-claim.yaml
    - apiVersion: v1
        kind: PersistentVolumeClaim
        metadata:
        name: postgres-volume-claim
        labels:
            app: postgres
        spec:
        storageClassName: manual
        accessModes:
            - ReadWriteMany
        resources:
            requests:
            storage: 10Gi
    # psql-deployment.yaml
    - apiVersion: apps/v1
        kind: Deployment
        metadata:
        name: postgres
        spec:
        replicas: 1
        selector:
            matchLabels:
            app: postgres
        template:
            metadata:
            labels:
                app: postgres
            spec:
            containers:
                - name: postgres
                image: 'postgres:17.1'
                imagePullPolicy: IfNotPresent
                ports:
                    - containerPort: 6600
                env:
                    - name: PGPORT
                    valueFrom:
                        configMapKeyRef:
                        name: postgres-secret
                        key: PGPORT  # Utiliza la variable PGPORT del ConfigMap
                    - name: POSTGRES_DB
                    valueFrom:
                        configMapKeyRef:
                        name: postgres-secret
                        key: POSTGRES_DB
                    - name: POSTGRES_USER
                    valueFrom:
                        configMapKeyRef:
                        name: postgres-secret
                        key: POSTGRES_USER
                    - name: POSTGRES_PASSWORD
                    valueFrom:
                        configMapKeyRef:
                        name: postgres-secret
                        key: POSTGRES_PASSWORD
                #envFrom:
                #  - configMapRef:
                #      name: postgres-secret
                volumeMounts:
                    - mountPath: /var/lib/postgresql/data
                    name: postgresdata
            volumes:
                - name: postgresdata
                persistentVolumeClaim:
                    claimName: postgres-volume-claim

    - apiVersion: v1
        kind: Service
        metadata:
        name: postgres
        labels:
            app: postgres
        spec:
        type: LoadBalancer
        selector:
            app: postgres
        ports:
            - port: 6600
            targetPort: 6600

    - apiVersion: autoscaling/v2
        kind: HorizontalPodAutoscaler
        metadata:
        name: postgres-hpa
        spec:
        behavior:
            scaleDown:
            stabilizationWindowSeconds: 300  # Espera 5 minutos antes de escalar hacia abajo
            scaleUp:
            stabilizationWindowSeconds: 120
        scaleTargetRef:
            apiVersion: apps/v1
            kind: Deployment
            name: postgres
        minReplicas: 2
        maxReplicas: 10
        metrics:
            - type: Resource
            resource:
                name: cpu
                target:
                type: Utilization
                averageUtilization: 60
            - type: Resource
            resource:
                name: memory
                target:
                type: Utilization
                averageUtilization: 60
  ```
