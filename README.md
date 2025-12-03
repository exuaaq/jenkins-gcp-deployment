# Jenkins Deployment on Google Cloud – Project Documentation

## Overview

Ovaj projekat opisuje implementaciju Jenkins CI/CD servera na Google Cloud Platform (GCP) koristeći: Google Cloud Compute Engine, Docker, NGINX, Namecheap DNS, SSL certifikat preko Certbot-a te reverse proxy konfiguraciju.

Svi koraci dokumentuju proces od kreiranja virtuelne mašine, dodjele domene, konfiguracije firewall-a, instalacija servisa, do finalnog pristupanja Jenkins instanci.

---

## 1. Google Cloud Setup

### 1.1 Kreiranje virtuelne mašine preko Google Cloud Shell-a

```bash
gcloud compute instances create project-jenkins \
--project=compact-scene-426809-k2 \
--zone=europe-west3-c \
--machine-type=n2-standard-2 \
--network-interface=network-tier=PREMIUM,stack-type=IPV4_ONLY,subnet=default \
--maintenance-policy=MIGRATE \
--provisioning-model=STANDARD \
--service-account=394531940262-compute@developer.gserviceaccount.com \
--scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append \
--tags=http-server,https-server \
--create-disk=auto-delete=yes,boot=yes,device-name=project-jenkins,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20240519,mode=rw,size=10,type=projects/fluid-root-423913-m1/zones/europe-west3-c/diskTypes/pd-balanced \
--no-shielded-secure-boot \
--shielded-vtpm \
--shielded-integrity-monitoring \
--labels=goog-ec-src=vm_add-gcloud \
--reservation-affinity=any
```

Glavne stavke ove komande:

* Kreiranje VM instance nazvane `project-jenkins`
* Zona: `europe-west3-c`
* Mašina: `n2-standard-2` (2 vCPU)
* Premium mrežni nivo + IPv4 only
* Mapping diska od 10 GB sa Ubuntu 20.04 ISO
* Tagovi za HTTP/HTTPS
* Shielded VM opcije

---

## 2. Konfiguracija DNS-a na Namecheap

Na Namecheap nalogu:

1. Otvoriti domen koji posjedujete
2. U sekciji **Advanced DNS** dodati novi **A Record**

   * Host: naziv subdomene (npr. `projekat`)
   * Value: vaša **external IP** adresa sa GCP-a

Time subdomena npr. `projekat.exuaaq.me` pokazuje na vašu VM instancu.

---

## 3. Firewall Rule (otvaranje porta)

U Cloud Shell-u kreirano je pravilo za otvaranje porta 8080:

```bash
gcloud compute --project=compact-scene-426809-k2 firewall-rules create allowport8080 \
--direction=INGRESS \
--priority=1000 \
--network=default \
--action=ALLOW \
--rules=tcp:8080 \
--source-ranges=0.0.0.0/0
```

Ovim se omogućava pristup Jenkins serveru koji koristi port 8080.

---

## 4. SSH Pristup preko Google Cloud Shell-a

Pristup VM-u preko Shell-a:

```bash
gcloud compute ssh guest@project-jenkins \
--project=compact-scene-426809-k2 \
--zone=europe-west3-c
```

### Dodavanje `sudo` privilegija korisniku `guest`:

```bash
sudo usermod -aG sudo guest
```

---

## 5. Instalacija Docker-a i NGINX-a

```bash
sudo apt-get update
sudo apt install docker
sudo apt install docker.io
sudo apt install nginx
```

---

## 6. Pokretanje Jenkins-a u Docker Container-u

```bash
docker run -d \
-v jenkins_home:/var/jenkins_home \
-p 8080:8080 \
--restart=on-failure \
jenkins/jenkins:lts-jdk17
```

Opis:

* `-d`: pokretanje u background-u
* volumen za persistentne podatke
* port forwarding 8080 → 8080
* LTS Jenkins sa JDK 17

---

## 7. Instalacija SSL Certifikata

```bash
sudo apt-get update
sudo apt-get install certbot python3-certbot-nginx
sudo certbot --nginx -d projekat.exuaaq.me
```

Pri instalaciji:

* unosi se email
* potvrđuju se uslovi korištenja
* omogućava automatski HTTP → HTTPS redirect

---

## 8. Konfiguracija NGINX Reverse Proxy-a

Editovanje default konfiguracije:

```bash
sudo nano /etc/nginx/sites-available/default
```

U `server` bloku za HTTPS dodati:

```nginx
location / {
    proxy_pass http://localhost:8080;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection keep-alive;
    proxy_set_header Host $http_host;
    proxy_cache_bypass $http_upgrade;
}
```

Restart NGINX-a:

```bash
sudo systemctl restart nginx
```

---

## 9. Pristup Jenkins-u i inicijalno podešavanje

Otvaranjem vaše HTTPS adrese npr.:

```
https://projekat.exuaaq.me
```

Jenkins traži inicijalni admin password.

Dobija se komandom:

```bash
docker exec <container_id> cat /var/jenkins_home/secrets/initialAdminPassword
```

Nakon unosa ključa instaliraju se plugin-i i završava postavka.

---

## Final Notes

Ova dokumentacija pokriva kompletan proces deployanja Jenkins servera na GCP uz sigurnosne i mrežne konfiguracije, DNS mapiranje i produkcijsku NGINX setup proceduru.

Sve komande su testirane i prilagođene Ubuntu 20.04 LTS okruženju.
