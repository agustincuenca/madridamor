# Skill: Networking

## Purpose

Fundamentos de redes que todo desarrollador web necesita conocer para debuggear, configurar y desplegar aplicaciones Rails.

## DNS (Domain Name System)

### Tipos de registros

| Tipo | Uso | Ejemplo |
|------|-----|---------|
| A | IPv4 address | `example.com -> 93.184.216.34` |
| AAAA | IPv6 address | `example.com -> 2606:2800:220:1:...` |
| CNAME | Alias a otro dominio | `www.example.com -> example.com` |
| MX | Mail server | `example.com -> mail.example.com (priority 10)` |
| TXT | Texto arbitrario | SPF, DKIM, verificación |
| NS | Nameservers | `example.com -> ns1.provider.com` |
| SRV | Service locator | `_sip._tcp.example.com` |
| CAA | Certificate Authority | Qué CAs pueden emitir certificados |

### TTL (Time To Live)

```bash
# Ver TTL de un registro
dig example.com +noall +answer

# Resultado:
# example.com.    300    IN    A    93.184.216.34
#                 ^^^
#                 TTL en segundos

# TTLs recomendados:
# - Producción estable: 3600 (1 hora) o más
# - Antes de migración: 300 (5 minutos)
# - Durante migración: 60 (1 minuto)
```

### Configuración DNS para Rails

```
# Registros típicos para una app Rails

# Dominio principal
example.com.           A       IP_DEL_SERVIDOR
example.com.           AAAA    IPv6_DEL_SERVIDOR

# Subdominio www
www.example.com.       CNAME   example.com.

# API subdomain
api.example.com.       A       IP_DEL_API_SERVER

# Email (si usas servicio externo)
example.com.           MX      10 mx1.mailprovider.com.
example.com.           MX      20 mx2.mailprovider.com.

# SPF para email
example.com.           TXT     "v=spf1 include:_spf.mailprovider.com ~all"

# Verificación de dominio
example.com.           TXT     "google-site-verification=xxx"
_dmarc.example.com.    TXT     "v=DMARC1; p=none; rua=mailto:dmarc@example.com"
```

### Debugging DNS

```bash
# Lookup básico
nslookup example.com
dig example.com

# Ver todos los registros
dig example.com ANY

# Query a servidor específico
dig @8.8.8.8 example.com

# Ver cadena completa de resolución
dig +trace example.com

# Reverse lookup
dig -x 93.184.216.34

# Ver registros MX
dig example.com MX

# Ver registros TXT
dig example.com TXT

# Verificar propagación
dig example.com @ns1.google.com
dig example.com @ns1.cloudflare.com
```

## TCP/IP

### Puertos comunes

| Puerto | Servicio |
|--------|----------|
| 22 | SSH |
| 80 | HTTP |
| 443 | HTTPS |
| 3000 | Rails (development) |
| 5432 | PostgreSQL |
| 6379 | Redis |
| 11211 | Memcached |
| 3306 | MySQL |
| 27017 | MongoDB |

### Sockets

```ruby
# En Rails, Puma escucha en socket
# config/puma.rb

# Socket TCP (para desarrollo)
port ENV.fetch("PORT") { 3000 }

# Unix socket (para producción con Nginx)
bind "unix:///var/run/puma.sock"

# O ambos
if ENV["RAILS_ENV"] == "production"
  bind "unix:///var/run/puma.sock"
else
  port 3000
end
```

### TCP Handshake

```
Cliente                    Servidor
   |                          |
   |  -------- SYN -------->  |   1. Cliente inicia
   |  <----- SYN-ACK ------   |   2. Servidor responde
   |  -------- ACK -------->  |   3. Conexión establecida
   |                          |
   |  <==== Datos ===>        |   4. Intercambio de datos
   |                          |
   |  -------- FIN -------->  |   5. Cliente cierra
   |  <----- ACK -------      |   6. Servidor confirma
   |  <----- FIN -------      |   7. Servidor cierra
   |  -------- ACK -------->  |   8. Cliente confirma
```

### Verificar conexiones

```bash
# Ver conexiones activas
netstat -an | grep LISTEN
ss -tlnp

# Ver conexiones a puerto específico
netstat -an | grep :3000
lsof -i :3000

# Ver estado de conexiones
netstat -s

# Verificar si puerto está abierto
nc -zv localhost 3000
telnet localhost 3000
```

## HTTP/HTTPS

### Métodos HTTP

| Método | Uso | Idempotente | Body |
|--------|-----|-------------|------|
| GET | Obtener recurso | Sí | No |
| POST | Crear recurso | No | Sí |
| PUT | Reemplazar recurso | Sí | Sí |
| PATCH | Actualizar parcial | No | Sí |
| DELETE | Eliminar recurso | Sí | No |
| HEAD | Solo headers | Sí | No |
| OPTIONS | Ver opciones | Sí | No |

### Headers comunes

```http
# Request headers
GET /users HTTP/1.1
Host: api.example.com
Accept: application/json
Content-Type: application/json
Authorization: Bearer eyJhbG...
User-Agent: MyApp/1.0
Accept-Language: es-ES,es;q=0.9
Cache-Control: no-cache
X-Request-ID: abc123

# Response headers
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 1234
Cache-Control: max-age=3600
ETag: "abc123"
X-Request-Id: abc123
X-Runtime: 0.045
Set-Cookie: _session_id=xxx; path=/; HttpOnly; Secure; SameSite=Lax
```

### Status codes

```
# 2xx - Éxito
200 OK                    - Éxito general
201 Created               - Recurso creado
204 No Content            - Éxito sin body

# 3xx - Redirección
301 Moved Permanently     - Redirección permanente
302 Found                 - Redirección temporal
304 Not Modified          - Usar cache

# 4xx - Error del cliente
400 Bad Request           - Request malformado
401 Unauthorized          - No autenticado
403 Forbidden             - No autorizado
404 Not Found             - Recurso no existe
422 Unprocessable Entity  - Validación fallida
429 Too Many Requests     - Rate limit

# 5xx - Error del servidor
500 Internal Server Error - Error del servidor
502 Bad Gateway          - Proxy/gateway error
503 Service Unavailable  - Servidor sobrecargado
504 Gateway Timeout      - Timeout del proxy
```

### Cookies

```ruby
# En Rails controller
class SessionsController < ApplicationController
  def create
    # Setear cookie
    cookies[:user_id] = {
      value: user.id,
      expires: 1.week.from_now,
      httponly: true,      # No accesible desde JS
      secure: Rails.env.production?,  # Solo HTTPS
      same_site: :lax      # Protección CSRF
    }

    # Cookie encriptada
    cookies.encrypted[:user_id] = user.id

    # Cookie firmada (no encriptada pero verificable)
    cookies.signed[:user_id] = user.id
  end

  def destroy
    cookies.delete(:user_id)
  end
end

# Session cookie (automática)
# config/initializers/session_store.rb
Rails.application.config.session_store :cookie_store,
  key: "_myapp_session",
  secure: Rails.env.production?,
  same_site: :lax,
  expire_after: 24.hours
```

## SSL/TLS

### Certificados

```bash
# Generar certificado autofirmado (desarrollo)
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout localhost.key \
  -out localhost.crt \
  -subj "/CN=localhost"

# Ver información de certificado
openssl x509 -in certificate.crt -text -noout

# Verificar certificado de sitio
openssl s_client -connect example.com:443 -servername example.com

# Ver fecha de expiración
echo | openssl s_client -servername example.com \
  -connect example.com:443 2>/dev/null | \
  openssl x509 -noout -dates
```

### Let's Encrypt con Certbot

```bash
# Instalar certbot
sudo apt install certbot python3-certbot-nginx

# Obtener certificado
sudo certbot --nginx -d example.com -d www.example.com

# Solo obtener certificado (sin configurar nginx)
sudo certbot certonly --standalone -d example.com

# Renovar certificados
sudo certbot renew

# Renovación automática (cron)
0 0 1 * * certbot renew --quiet
```

### SSL Termination

```nginx
# nginx.conf - SSL termination en Nginx

upstream rails_app {
  server unix:///var/run/puma.sock fail_timeout=0;
}

server {
  listen 80;
  server_name example.com;
  return 301 https://$server_name$request_uri;
}

server {
  listen 443 ssl http2;
  server_name example.com;

  ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

  # SSL configuration
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
  ssl_prefer_server_ciphers off;

  # HSTS
  add_header Strict-Transport-Security "max-age=63072000" always;

  location / {
    proxy_pass http://rails_app;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Host $http_host;
    proxy_redirect off;
  }
}
```

## Proxies

### Forward Proxy

```
Cliente -> Forward Proxy -> Internet -> Servidor

# Uso: anonimato, filtrado, cache
# El cliente sabe que usa proxy
```

### Reverse Proxy (Nginx)

```
Internet -> Nginx (reverse proxy) -> Rails App

# Uso: SSL termination, load balancing, cache
# El cliente no sabe que hay proxy
```

### Configuración Nginx para Rails

```nginx
# /etc/nginx/sites-available/myapp

upstream puma {
  server unix:///var/www/myapp/shared/sockets/puma.sock;
}

server {
  listen 80;
  server_name example.com;

  root /var/www/myapp/current/public;

  # Servir archivos estáticos directamente
  location ^~ /assets/ {
    gzip_static on;
    expires max;
    add_header Cache-Control public;
  }

  # Uploads
  location ^~ /uploads/ {
    expires max;
  }

  # Rails app
  location / {
    try_files $uri @puma;
  }

  location @puma {
    proxy_pass http://puma;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Host $http_host;
    proxy_redirect off;

    # WebSocket support
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
  }

  # Error pages
  error_page 500 502 503 504 /500.html;
  location = /500.html {
    root /var/www/myapp/current/public;
  }
}
```

## Load Balancers

### Algoritmos

| Algoritmo | Descripción |
|-----------|-------------|
| Round Robin | Rotación secuencial |
| Least Connections | Servidor con menos conexiones |
| IP Hash | Mismo cliente siempre al mismo servidor |
| Weighted | Según capacidad del servidor |

### Nginx como Load Balancer

```nginx
upstream rails_cluster {
  # Weighted round robin
  server web1.example.com:3000 weight=3;
  server web2.example.com:3000 weight=2;
  server web3.example.com:3000 weight=1;

  # Least connections
  # least_conn;

  # IP hash (sticky sessions)
  # ip_hash;

  # Keepalive connections
  keepalive 32;
}

server {
  listen 80;

  location / {
    proxy_pass http://rails_cluster;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
  }
}
```

### Sticky Sessions

```nginx
upstream rails_cluster {
  ip_hash;  # Basado en IP

  # O usar cookie
  # sticky cookie srv_id expires=1h domain=.example.com path=/;

  server web1:3000;
  server web2:3000;
}
```

### Health Checks

```nginx
upstream rails_cluster {
  server web1:3000 max_fails=3 fail_timeout=30s;
  server web2:3000 max_fails=3 fail_timeout=30s;
  server web3:3000 backup;  # Solo si otros fallan
}
```

## CDN (Content Delivery Network)

### Cómo funciona

```
Usuario en España
       |
       v
CDN Edge Server (Madrid)
       |
       | (cache miss)
       v
Origin Server (USA)
       |
       | (respuesta)
       v
CDN Edge (cachea)
       |
       v
Usuario (respuesta rápida)

# Siguiente request del mismo recurso:
Usuario -> CDN Edge (cache hit) -> Usuario
```

### Configuración en Rails

```ruby
# config/environments/production.rb
config.action_controller.asset_host = "https://cdn.example.com"

# O con ENV
config.action_controller.asset_host = ENV["CDN_HOST"]

# Conditional por request
config.action_controller.asset_host = proc { |source, request|
  if request.ssl?
    "https://cdn.example.com"
  else
    "http://cdn.example.com"
  end
}
```

### Headers de cache para CDN

```ruby
# app/controllers/static_controller.rb
class StaticController < ApplicationController
  def show
    # Cache por 1 día en CDN, 1 hora en browser
    expires_in 1.day, public: true, stale_while_revalidate: 1.hour

    # Headers explícitos
    response.headers["Cache-Control"] = "public, max-age=86400, s-maxage=604800"
    response.headers["Surrogate-Control"] = "max-age=604800"
  end
end

# Para assets (automático con Rails asset pipeline)
# config/environments/production.rb
config.public_file_server.headers = {
  "Cache-Control" => "public, max-age=31536000"
}
```

### Invalidación de cache

```ruby
# Con Cloudflare
class CloudflareService
  def self.purge_cache(urls)
    conn = Faraday.new(url: "https://api.cloudflare.com")
    conn.post("/client/v4/zones/#{ZONE_ID}/purge_cache") do |req|
      req.headers["Authorization"] = "Bearer #{API_TOKEN}"
      req.headers["Content-Type"] = "application/json"
      req.body = { files: urls }.to_json
    end
  end

  def self.purge_all
    # Cuidado: puede ser lento
    conn.post("/client/v4/zones/#{ZONE_ID}/purge_cache") do |req|
      req.body = { purge_everything: true }.to_json
    end
  end
end

# Uso
CloudflareService.purge_cache([
  "https://example.com/assets/application.css",
  "https://example.com/api/products.json"
])
```

## Firewalls

### iptables básico

```bash
# Ver reglas actuales
sudo iptables -L -n -v

# Permitir SSH
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Permitir HTTP/HTTPS
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Permitir localhost
sudo iptables -A INPUT -i lo -j ACCEPT

# Denegar todo lo demás
sudo iptables -A INPUT -j DROP

# Guardar reglas
sudo iptables-save > /etc/iptables/rules.v4
```

### UFW (Ubuntu)

```bash
# Habilitar
sudo ufw enable

# Reglas básicas
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Permitir desde IP específica
sudo ufw allow from 192.168.1.100 to any port 5432

# Denegar
sudo ufw deny from 23.45.67.89

# Ver estado
sudo ufw status verbose

# Eliminar regla
sudo ufw delete allow 80/tcp
```

### Security Groups (AWS)

```yaml
# Terraform example
resource "aws_security_group" "web" {
  name        = "web-servers"
  description = "Security group for web servers"

  # HTTP
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTPS
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # SSH solo desde oficina
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["203.0.113.0/24"]
  }

  # PostgreSQL solo interno
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.db.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

## Debugging de Red

### curl

```bash
# Request básico
curl https://example.com

# Ver headers de respuesta
curl -I https://example.com

# Ver request y response headers
curl -v https://example.com

# Con datos POST
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -d '{"name": "John", "email": "john@example.com"}'

# Con autenticación
curl -H "Authorization: Bearer TOKEN" https://api.example.com/me

# Seguir redirects
curl -L https://example.com

# Timeout
curl --max-time 10 https://example.com

# Guardar cookies
curl -c cookies.txt https://example.com/login
curl -b cookies.txt https://example.com/dashboard

# Ver tiempos
curl -w "\nDNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n" \
  -o /dev/null -s https://example.com

# Ignorar SSL (solo desarrollo)
curl -k https://localhost:3000
```

### wget

```bash
# Descargar archivo
wget https://example.com/file.zip

# Descargar con nombre específico
wget -O myfile.zip https://example.com/file.zip

# Continuar descarga interrumpida
wget -c https://example.com/large-file.zip

# Spider mode (verificar enlaces)
wget --spider https://example.com

# Mirror sitio
wget --mirror --convert-links --page-requisites https://example.com
```

### netstat/ss

```bash
# Ver puertos en escucha
netstat -tlnp
ss -tlnp

# Ver todas las conexiones
netstat -an
ss -an

# Conexiones establecidas
netstat -an | grep ESTABLISHED

# Conexiones por estado
ss -s

# Conexiones a puerto específico
netstat -an | grep :3000
ss -an | grep :3000
```

### dig/nslookup

```bash
# Lookup básico
dig example.com
nslookup example.com

# Solo respuesta
dig +short example.com

# Registros específicos
dig example.com MX
dig example.com TXT
dig example.com NS

# Reverse lookup
dig -x 93.184.216.34

# Trace completo
dig +trace example.com

# Servidor DNS específico
dig @8.8.8.8 example.com
```

### Otros comandos útiles

```bash
# Ping
ping -c 4 example.com

# Traceroute
traceroute example.com
mtr example.com  # Mejor visualización

# Ver interfaces de red
ifconfig
ip addr

# Ver tabla de rutas
route -n
ip route

# Test de puerto abierto
nc -zv example.com 443
telnet example.com 443

# tcpdump (captura de paquetes)
sudo tcpdump -i eth0 port 80
sudo tcpdump -i eth0 host example.com
```

## CORS en Rails

### Qué es CORS

```
Browser (example.com) -> API (api.example.com)

1. Browser envía preflight request (OPTIONS)
2. API responde con headers CORS permitidos
3. Browser envía request real si está permitido
```

### Configuración

```ruby
# Gemfile
gem "rack-cors"

# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    # Orígenes permitidos
    origins "example.com", "www.example.com", /\Ahttp:\/\/localhost:\d+\z/

    resource "/api/*",
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head],
      credentials: true,  # Permitir cookies
      max_age: 86400      # Cache preflight por 1 día
  end

  # API pública (sin credenciales)
  allow do
    origins "*"

    resource "/public/*",
      headers: :any,
      methods: [:get, :options, :head]
  end
end
```

### Headers CORS

```ruby
# Manual en controller
class ApiController < ApplicationController
  before_action :set_cors_headers

  private

  def set_cors_headers
    response.headers["Access-Control-Allow-Origin"] = request.headers["Origin"] || "*"
    response.headers["Access-Control-Allow-Methods"] = "GET, POST, PUT, DELETE, OPTIONS"
    response.headers["Access-Control-Allow-Headers"] = "Content-Type, Authorization"
    response.headers["Access-Control-Allow-Credentials"] = "true"
    response.headers["Access-Control-Max-Age"] = "86400"
  end

  def options
    head :ok
  end
end
```

### Debugging CORS

```bash
# Simular preflight
curl -X OPTIONS https://api.example.com/resource \
  -H "Origin: https://example.com" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: Content-Type" \
  -v

# Verificar headers de respuesta
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type
Access-Control-Allow-Credentials: true
```

### Errores comunes

```javascript
// Error en browser
// Access to XMLHttpRequest at 'https://api.example.com' from origin
// 'https://example.com' has been blocked by CORS policy

// Causas comunes:
// 1. Origin no está en lista permitida
// 2. Método no permitido
// 3. Header no permitido
// 4. credentials: true pero Allow-Origin es "*"
```

## Checklist de Networking

### Antes de deploy

- [ ] DNS configurado correctamente
- [ ] Certificado SSL válido
- [ ] Firewall permite puertos necesarios
- [ ] CORS configurado si hay API
- [ ] CDN configurado para assets

### Debugging

- [ ] ¿DNS resuelve correctamente? (dig)
- [ ] ¿Puerto está abierto? (nc, telnet)
- [ ] ¿Servidor responde? (curl -v)
- [ ] ¿Headers correctos? (curl -I)
- [ ] ¿SSL válido? (openssl s_client)
- [ ] ¿CORS correcto? (browser console)
