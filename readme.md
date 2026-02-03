TFAC Monitoring Portal
ศูนย์รวม URL ระบบ Monitoring ของ TFAC
ใช้สำหรับรวมลิงก์ไปยังระบบ Infrastructure ทั้งหมดในหน้าเดียว
Deploy บน K3s + Nginx + Docker + GitHub Container Registry (GHCR)

Production URL
👉 https://monitor.tfac.or.th

1. Architecture Overview
โครงสร้างระบบจริง (End-to-End)

User Browser
    |
Cloudflare DNS / SSL
    |
K3s Ingress Controller
    |
Service (tfac-monitoring-portal)
    |
Pod (nginx + static web)
    |
Docker Image (GHCR)
2. Project Structure
โครงสร้างโปรเจกต์ทั้งหมด

tfac-monitoring-portal/
 ├── index.html        # หน้าเว็บหลัก
 ├── style.css         # CSS
 ├── script.js        # JS (optional)
 ├── default.conf     # nginx config
 ├── Dockerfile       # build docker image
 └── README.md
3. Technology Stack
Layer	Technology
Frontend	HTML / CSS (Static Web)
Web Server	Nginx
Container	Docker
Registry	GitHub Container Registry (ghcr.io)
Orchestration	K3s (Kubernetes)
DNS / SSL	Cloudflare
PHASE 1 – Create Web Portal
3.1 index.html
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>TFAC Monitoring Center</title>
<link rel="stylesheet" href="style.css">
</head>
<body>

<h1>TFAC Monitoring Center</h1>

<div class="grid">
  <a href="https://grafana.tfac.or.th" class="card">Grafana</a>
  <a href="https://uptime.tfac.or.th" class="card">Uptime</a>
  <a href="https://rancher.tfac.or.th" class="card">Rancher</a>
  <a href="https://gitlab.tfac.or.th" class="card">GitLab</a>
</div>

<footer>Version 1.4</footer>

</body>
</html>
3.2 style.css
body {
  background:#0f172a;
  color:white;
  font-family:Segoe UI;
  text-align:center;
}

.grid {
  display:grid;
  grid-template-columns:repeat(5,1fr);
  gap:24px;
  padding:0 60px;
}

.card {
  background:#1e293b;
  padding:40px;
  border-radius:14px;
  font-size:22px;
  color:white;
  text-decoration:none;
}

.card:hover {
  background:#2563eb;
}

footer {
  margin-top:40px;
  color:#94a3b8;
}
PHASE 2 – Dockerize
4.1 nginx config (default.conf)
server {
    listen 80 default_server;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
4.2 Dockerfile
FROM nginx:alpine

RUN rm -f /etc/nginx/conf.d/*
COPY default.conf /etc/nginx/conf.d/default.conf
COPY . /usr/share/nginx/html

EXPOSE 80
CMD ["nginx","-g","daemon off;"]
PHASE 3 – Local Build & Test
5.1 Build
docker build -t monitoring-portal:1.4 .
5.2 Run locally
docker run -d -p 8081:80 --name monitoring-portal monitoring-portal:1.4
เปิดเว็บ

http://localhost:8081
PHASE 4 – Push to GitHub Container Registry
6.1 Login
docker login ghcr.io -u Khomsan146
6.2 Tag & Push
docker tag monitoring-portal:1.4 ghcr.io/khomsan146/tfac-frontend-infra-allapp:1.4
docker push ghcr.io/khomsan146/tfac-frontend-infra-allapp:1.4
ตรวจสอบได้ที่

GitHub → Profile → Packages → tfac-frontend-infra-allapp
PHASE 5 – Deploy on K3s
7.1 Create Secret (ทำครั้งเดียว)
kubectl create secret docker-registry ghcr-secret \
 --docker-server=ghcr.io \
 --docker-username=Khomsan146 \
 --docker-password=<GITHUB_TOKEN> \
 --docker-email=admin@tfac.or.th
7.2 Deployment
deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: tfac-monitoring-portal
spec:
  replicas: 2
  selector:
    matchLabels:
      app: tfac-monitoring-portal
  template:
    metadata:
      labels:
        app: tfac-monitoring-portal
    spec:
      imagePullSecrets:
      - name: ghcr-secret
      containers:
      - name: portal
        image: ghcr.io/khomsan146/tfac-frontend-infra-allapp:1.4
        ports:
        - containerPort: 80
7.3 Service
service.yaml

apiVersion: v1
kind: Service
metadata:
  name: tfac-monitoring-portal
spec:
  selector:
    app: tfac-monitoring-portal
  ports:
  - port: 80
    targetPort: 80
7.4 Ingress
ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tfac-monitoring-portal
spec:
  rules:
  - host: monitor.tfac.or.th
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tfac-monitoring-portal
            port:
              number: 80
7.5 Apply
kubectl apply -f .
PHASE 6 – Update Version (Workflow มาตรฐาน)
ทุกครั้งที่แก้ไขเว็บ

ฝั่ง Dev
docker build -t monitoring-portal:X .
docker tag monitoring-portal:X ghcr.io/khomsan146/tfac-frontend-infra-allapp:X
docker push ghcr.io/khomsan146/tfac-frontend-infra-allapp:X
ฝั่ง Production
kubectl set image deployment/tfac-monitoring-portal portal=ghcr.io/khomsan146/tfac-frontend-infra-allapp:X
kubectl rollout status deployment/tfac-monitoring-portal
ตรวจสอบ Version
จาก K3s
kubectl describe deployment tfac-monitoring-portal | grep Image
จาก GitHub
Profile → Packages → tfac-frontend-infra-allapp → Versions
จากหน้าเว็บ
Footer:

Version X
Troubleshooting
ImagePullBackOff
kubectl describe pod <pod-name>
เช็คว่า

tag มีจริงใน GHCR

secret ถูกต้อง

เปิดเว็บไม่ได้
kubectl get ingress
kubectl get svc
kubectl get pods
Best Practices
ห้ามใช้ latest ใน production

ใช้ version แบบ semantic เช่น 1.0, 1.1, 1.2

ทุกครั้งต้อง build ใหม่ → push → rollout

ใส่ version ในหน้าเว็บเสมอ

Mental Model (จำอันนี้พอ)
HTML Source
   ↓
Docker Image
   ↓
GitHub Registry
   ↓
K3s Deployment
   ↓
monitor.tfac.or.th
README นี้คือ คู่มือครบทั้งระบบระดับ Production
