# -------------------------
# Deployment — описывает запуск приложения
# -------------------------
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  # Начинаем с 2 реплик — чтобы приложение работало даже при сбое одной ноды
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      # Анти-аффинити — стараемся распределять поды по разным нодам
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - web-app
              topologyKey: "kubernetes.io/hostname"
      containers:
        - name: web-app
          image: nginx:latest # как пример приложения
          ports:
            - containerPort: 80
          # Проба готовности — проверяем, что приложение успело инициализироваться
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 15
            periodSeconds: 5
          # Проба живости — следим, что контейнер не завис
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 30
            periodSeconds: 10
          resources:
            # Минимальные ресурсы, которые гарантированы приложению
            requests:
              cpu: "200m"
              memory: "128Mi"
            # Лимиты — максимум, сколько под может съесть
            limits:
              cpu: "500m"
              memory: "256Mi"

---
# -------------------------
# Service — делаем доступ к приложению внутри кластера
# -------------------------
apiVersion: v1
kind: Service
metadata:
  name: web-app-svc
spec:
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP # внутри кластера, через ingress

---
# -------------------------
# HPA — автоскейлинг подов
# -------------------------
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2   # ночью минимум 2 пода
  maxReplicas: 6   # днём может вырасти до 6 подов
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
