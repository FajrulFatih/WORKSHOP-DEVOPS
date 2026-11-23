# Praktikum 02: Konsep Container dan Instalasi Docker

> **Mata Kuliah:** Workshop DevOps  
> **Dosen Pengampu:** Dr. Ferry Astika Saputra, S.T., M.Sc.  
> **Kelompok:** 5  
> **Anggota:**
> - Muhammad Fajrul Fatih Abul ‘Ilmi (3124600040)
> - Nur Aini Agusthina (3124600050)
> - Abdullah Falich (3124600059)
> 
> **Program Studi:** D4 Teknik Informatika, Departemen Teknik Informatika dan Komputer  
> **Institusi:** Politeknik Elektronika Negeri Surabaya (PENS) - 2026

---

## PRAKTIKUM BAB 2: KONSEP CONTAINER DAN INSTALASI DOCKER

### Praktikum

#### Arsitektur Laboratorium dan Prasyarat Lingkungan

```bash
mkdir -p ~/docker-lab/bab-2
cd ~/docker-lab/bab-2
sudo apt update
```

![Screenshot 1](uploads/praktikum-02/image-01.jpg)

![Screenshot 2](uploads/praktikum-02/image-02.png)

```bash
sudo usermod -aG docker $USER
newgrp docker
docker version
```

![Screenshot 3](uploads/praktikum-02/image-03.png)

```bash
docker run hello-world 
```

![Screenshot 4](uploads/praktikum-02/image-04.png)

#### Container Nginx dan Ubuntu Interaktif

```bash
docker pull nginx:1.26
docker run -d --name web-public -p 8080:80 nginx:1.26
```

![Screenshot 5](uploads/praktikum-02/image-05.jpg)

```bash
docker logs --tail 20 web-public
curl http://localhost:8080
```

![Screenshot 6](uploads/praktikum-02/image-06.jpg)

#### Dockerfile Custom Web Statis

```dockerfile
mkdir -p ~/docker-lab/custom-web && cd ~/docker-lab/custom-web
cat > index.html << 'EOF'
<h1>Docker Lab PENS</h1>
<p>Container berhasil berjalan.</p>
EOF
cat > Dockerfile << 'EOF'
FROM nginx:1.26-alpine
LABEL maintainer="admin@pens.ac.id"
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF
docker build -t pens-web:1.0 .
docker run -d --name pens-app -p 9090:80 pens-web:1.0
```

![Screenshot 7](uploads/praktikum-02/image-07.png)

```dockerfile
curl http://localhost:9090
```

![Screenshot 8](uploads/praktikum-02/image-08.jpg)

