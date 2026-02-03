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
PHASE 2 – Dockerize
4.1 nginx config (default.conf)
4.2 Dockerfile
PHASE 3 – Local Build & Test
5.1 Build test
docker build -t monitoring-portal:1.x
5.2 Run locally
docker run -d -p 8081:80 --name monitoring-portal monitoring-portal:1.4
http://localhost:8081
PHASE 4 – Push to GitHub Container Registry
6.1 Login
docker login ghcr.io -u Khomsan146
6.2 Tag & Push
docker tag monitoring-portal:1.x ghcr.io/khomsan146/tfac-frontend-infra-allapp:1.x
docker push ghcr.io/khomsan146/tfac-frontend-infra-allapp:1.x
ตรวจสอบได้ที่
GitHub → Profile → Packages → tfac-frontend-infra-allapp
PHASE 5 – Deploy on K3s
7.1 Create Secret 
kubectl create secret docker-registry ghcr-secret \
7.2 Deployment
deployment.yaml
7.3 Service
service.yaml
7.4 Ingress
ingress.yaml
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


