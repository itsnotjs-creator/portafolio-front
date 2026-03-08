# Guía de Despliegue en VPS

Esta guía explica cómo deployar tu aplicación Astro en tu propia VPS.

## 📋 Prerequisitos

- VPS con Ubuntu/Debian
- Usuario con permisos sudo
- Node.js 18+ instalado en el servidor
- Nginx instalado

## 🚀 Opción 1: Deploy Manual (Build Local)

### Paso 1: Construir la aplicación localmente

```bash
# En tu máquina local
cd /home/jjcmjavascript/apps/itisnotjs/app
npm run build
```

Esto genera una carpeta `dist/` con los archivos estáticos optimizados.

### Paso 2: Subir archivos al servidor

```bash
# Subir archivos con rsync
rsync -avz dist/ deploy@207.180.221.183:/var/www/portafolio
```

Si obtienes error de permisos, conecta al servidor y ejecuta:

```bash
ssh deploy@207.180.221.183
sudo mkdir -p /var/www/portafolio
sudo chown -R deploy:deploy /var/www/portafolio
exit
```

## 🔧 Configurar Nginx

### Paso 1: Conectarse al servidor

```bash
ssh deploy@207.180.221.183
```

### Paso 2: Crear archivo de configuración

```bash
sudo nano /etc/nginx/sites-available/portafolio
```

### Paso 3: Agregar configuración

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    root /var/www/portafolio;
    index index.html;

    # Logs
    access_log /var/log/nginx/portafolio-access.log;
    error_log /var/log/nginx/portafolio-error.log;

    # Manejo de rutas SPA/MPA
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache para assets estáticos
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Seguridad adicional
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
}
```

**Si tienes dominio**, agrega esta línea después de `listen`:

```nginx
    server_name tu-dominio.com www.tu-dominio.com;
```

### Paso 4: Activar el sitio

```bash
# Eliminar configuración default si existe
sudo rm /etc/nginx/sites-enabled/default

# Activar tu sitio
sudo ln -s /etc/nginx/sites-available/portafolio /etc/nginx/sites-enabled/

# Verificar configuración
sudo nginx -t

# Reiniciar nginx
sudo systemctl restart nginx
```

### Paso 5: Verificar el estado

```bash
sudo systemctl status nginx
```

## 🌐 Acceder a tu sitio

Abre tu navegador y ve a:

```
http://207.180.221.183
```

## 🔄 Script de Deploy Automático

Crea un script en tu proyecto local para automatizar deploys futuros:

```bash
nano deploy.sh
```

Contenido:

```bash
#!/bin/bash

echo "🚀 Iniciando deploy..."

# 1. Construir aplicación
echo "📦 Construyendo aplicación..."
npm run build

# 2. Verificar que dist existe
if [ ! -d "dist" ]; then
    echo "❌ Error: carpeta dist no encontrada"
    exit 1
fi

# 3. Subir archivos
echo "📤 Subiendo archivos al servidor..."
rsync -avz --delete dist/ deploy@207.180.221.183:/var/www/portafolio

# 4. Reiniciar nginx (opcional)
echo "🔄 Reiniciando nginx..."
ssh deploy@207.180.221.183 "sudo systemctl reload nginx"

echo "✅ Deploy completado!"
echo "🌐 Tu sitio está disponible en: http://207.180.221.183"
```

Dar permisos de ejecución:

```bash
chmod +x deploy.sh
```

### Usar el script

```bash
./deploy.sh
```

## 🔐 Configurar SSL/HTTPS (opcional pero recomendado)

### Con Let's Encrypt (gratis)

```bash
# En el servidor
sudo apt update
sudo apt install certbot python3-certbot-nginx

# Si tienes dominio
sudo certbot --nginx -d tu-dominio.com -d www.tu-dominio.com

# Renovación automática ya está configurada
```

## 🚀 Opción 2: Deploy con Git (Automatizado)

### En el servidor:

```bash
# Instalar Node.js si no está instalado
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# Clonar repositorio
cd /var/www/
sudo git clone tu-repositorio.git portafolio-src
sudo chown -R deploy:deploy portafolio-src
cd portafolio-src/app

# Instalar dependencias
npm install

# Construir
npm run build

# Copiar archivos
sudo cp -r dist/* /var/www/portafolio/
```

### Script de actualización:

```bash
# En el servidor
nano /home/deploy/update-portafolio.sh
```

```bash
#!/bin/bash
cd /var/www/portafolio-src/app
git pull
npm install
npm run build
sudo cp -r dist/* /var/www/portafolio/
sudo systemctl reload nginx
echo "✅ Actualización completada"
```

## 🔍 Solución de Problemas

### Error 403 Forbidden

```bash
# Verificar permisos
sudo chown -R www-data:www-data /var/www/portafolio
sudo chmod -R 755 /var/www/portafolio
```

### Error 502 Bad Gateway

```bash
# Verificar logs
sudo tail -f /var/log/nginx/error.log

# Reiniciar nginx
sudo systemctl restart nginx
```

### Página no se actualiza

```bash
# Limpiar caché del navegador
# O usar rsync con --delete
rsync -avz --delete dist/ deploy@207.180.221.183:/var/www/portafolio
```

### Ver logs en tiempo real

```bash
# Access logs
sudo tail -f /var/log/nginx/portafolio-access.log

# Error logs
sudo tail -f /var/log/nginx/portafolio-error.log
```

## 📝 Checklist de Deploy

- [ ] Construir aplicación: `npm run build`
- [ ] Subir archivos: `rsync -avz dist/ deploy@IP:/var/www/portafolio`
- [ ] Configurar Nginx
- [ ] Activar sitio en Nginx
- [ ] Verificar permisos de archivos
- [ ] Verificar que el sitio carga correctamente
- [ ] (Opcional) Configurar SSL con Let's Encrypt
- [ ] (Opcional) Configurar dominio

## 🎯 Deploy Rápido (Resumen)

```bash
# Local
npm run build
rsync -avz dist/ deploy@207.180.221.183:/var/www/portafolio

# Servidor (primera vez)
ssh deploy@207.180.221.183
sudo nano /etc/nginx/sites-available/portafolio
# ... pegar configuración nginx ...
sudo ln -s /etc/nginx/sites-available/portafolio /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

## 📚 Recursos

- [Documentación de Astro](https://docs.astro.build)
- [Documentación de Nginx](https://nginx.org/en/docs/)
- [Let's Encrypt](https://letsencrypt.org/)

---

**IP del servidor:** 207.180.221.183  
**Usuario:** deploy  
**Ruta deploy:** /var/www/portafolio

ssh deploy@207.180.221.183
