# Integración GitHub → Splunk HEC

## Resumen
Configuración de ingesta de eventos de GitHub hacia Splunk utilizando HTTP Event Collector (HEC) a través de GitHub Actions, con nginx como proxy transparente.

## Arquitectura
```
GitHub Repository
    ↓ (GitHub Actions Workflow)
    ↓ HTTP POST
nginx (ingesta.hugo.opentek.cl:8088)
    ↓ (Transparent Proxy)
Splunk HEC (192.168.0.15:8088)
    ↓
Splunk Index (main)
```

## Componentes

### 1. Splunk HTTP Event Collector

**Servidor:** `192.168.0.15:8088`  
**Token:** `d7907368-cc55-4186-82f5-28f66ff95004`  
**Index:** `main`  
**Sourcetype:** `github:actions`

#### Configuración HEC
```ini
# /opt/splunk/etc/apps/splunk_httpinput/local/inputs.conf
[http]
disabled = 0
port = 8088
enableSSL = 0  # Cambiar a 1 en producción

[http://github_webhook]
disabled = 0
token = d7907368-cc55-4186-82f5-28f66ff95004
indexes = main
sourcetype = github:actions
```

### 2. Nginx Proxy (Transparente)

**Servidor:** `ingesta.hugo.opentek.cl`  
**Puerto:** `8088`

#### Configuración nginx
```nginx
# /etc/nginx/conf.d/stream.conf
stream {
    upstream splunk_hec_hugo {
        server 192.168.0.15:8088;
    }
    
    server {
        listen 8088;
        proxy_pass splunk_hec_hugo;
    }
}
```

**Nota:** En producción, habilitar SSL en Splunk y usar `ssl_preread on` en nginx.

### 3. GitHub Actions Workflow

**Archivo:** `.github/workflows/splunk-events.yml`
```yaml
name: Send Events to Splunk
on:
  push:
    branches: [ main, develop ]
  pull_request:
    types: [ opened, closed, reopened, synchronize ]
  issues:
    types: [ opened, closed, reopened ]
  issue_comment:
    types: [ created ]

jobs:
  send-to-splunk:
    runs-on: ubuntu-latest
    steps:
      - name: Send event to Splunk HEC
        run: |
          curl -X POST http://ingesta.hugo.opentek.cl:8088/services/collector/event \
            -H "Authorization: Splunk ${{ secrets.SPLUNK_HEC_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d "{
              \"event\": {
                \"repository\": \"${{ github.repository }}\",
                \"event_name\": \"${{ github.event_name }}\",
                \"action\": \"${{ github.event.action }}\",
                \"actor\": \"${{ github.actor }}\",
                \"ref\": \"${{ github.ref }}\",
                \"sha\": \"${{ github.sha }}\",
                \"workflow\": \"${{ github.workflow }}\",
                \"run_id\": \"${{ github.run_id }}\",
                \"timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"
              },
              \"sourcetype\": \"github:actions\",
              \"source\": \"github\",
              \"host\": \"github.com\"
            }"
```

#### Configuración del Secret

1. Ir a **Settings** → **Secrets and variables** → **Actions**
2. Click en **New repository secret**
3. Name: `SPLUNK_HEC_TOKEN`
4. Value: `d7907368-cc55-4186-82f5-28f66ff95004`

## Queries SPL de Ejemplo

### Búsqueda básica de eventos
```spl
index=main sourcetype=github:actions
| table _time repository event_name actor ref sha
```

### Eventos por tipo
```spl
index=main sourcetype=github:actions
| stats count by event_name
| sort -count
```

### Actividad por usuario
```spl
index=main sourcetype=github:actions
| stats count by actor
| sort -count
```

### Timeline de eventos
```spl
index=main sourcetype=github:actions
| timechart count by event_name
```

### Pull requests cerrados
```spl
index=main sourcetype=github:actions event_name=pull_request action=closed
| table _time repository actor ref
```

### Pushes por rama
```spl
index=main sourcetype=github:actions event_name=push
| rex field=ref "refs/heads/(?<branch>.*)"
| stats count by branch
| sort -count
```

### Dashboard básico
```spl
index=main sourcetype=github:actions
| stats 
    count as "Total Events",
    dc(actor) as "Unique Users",
    dc(repository) as "Repositories"
```

## Testing

### Test directo a Splunk HEC
```bash
curl http://192.168.0.15:8088/services/collector/event \
  -H "Authorization: Splunk d7907368-cc55-4186-82f5-28f66ff95004" \
  -d '{"event": "test direct", "sourcetype": "github:actions"}'
```

### Test a través de nginx
```bash
curl http://ingesta.hugo.opentek.cl:8088/services/collector/event \
  -H "Authorization: Splunk d7907368-cc55-4186-82f5-28f66ff95004" \
  -d '{"event": "test via nginx", "sourcetype": "github:actions"}'
```

### Verificar en Splunk
```spl
index=main sourcetype=github:actions "test"
| table _time event
```

## Migración a Producción (SSL)

### 1. Habilitar SSL en Splunk HEC
```bash
# Generar certificado
cd /opt/splunk/etc/auth/mycerts/
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout hec-key.pem -out hec-cert.pem \
  -subj "/CN=ingesta.hugo.opentek.cl"

cat hec-cert.pem hec-key.pem > hec-combined.pem
chmod 600 hec-combined.pem
chown splunk:splunk hec-combined.pem
```

### 2. Actualizar inputs.conf
```ini
[http]
disabled = 0
port = 8088
enableSSL = 1
serverCert = /opt/splunk/etc/auth/mycerts/hec-combined.pem
sslVersions = tls1.2, tls1.3
```

### 3. Actualizar nginx
```nginx
stream {
    map $ssl_preread_server_name $sni_upstream {
        pmo.kudaw.com pmo_tls;
        splunk.kudaw.com splunk_tls;
        ingesta.hugo.opentek.cl splunk_hec_hugo;
    }
    
    upstream splunk_hec_hugo { server 192.168.0.15:8088; }
    
    server {
        listen 8088;
        proxy_pass $sni_upstream;
        ssl_preread on;
    }
}
```

### 4. Actualizar GitHub Actions workflow
Cambiar `http://` por `https://` en el curl.

## Troubleshooting

### No llegan eventos a Splunk
```bash
# Verificar conectividad
telnet ingesta.hugo.opentek.cl 8088

# Verificar logs de nginx
tail -f /var/log/nginx/error.log

# Verificar logs de Splunk
tail -f /opt/splunk/var/log/splunk/splunkd.log | grep HEC
```

### Error 403 Forbidden
- Verificar que el token HEC esté activo
- Verificar que el índice existe y está habilitado
- Revisar `indexes.conf` y permisos

### Error de conexión SSL
- Verificar certificados en Splunk
- Verificar que `enableSSL = 1` en inputs.conf
- Probar con `curl -k` para debug

### Eventos duplicados
- Verificar que no haya múltiples workflows enviando el mismo evento
- Revisar triggers en el workflow YAML

## Monitoreo

### Alert de eventos fallidos
```spl
index=_internal sourcetype=splunkd component=HttpInputDataHandler
| search "Failed to parse"
| stats count by host
```

### Volumen de eventos por hora
```spl
index=main sourcetype=github:actions
| timechart span=1h count
```

### Health check del HEC
```spl
index=_internal sourcetype=splunkd component=HttpInputDataHandler
| stats count by status
```

## Referencias

- [Splunk HTTP Event Collector Documentation](https://docs.splunk.com/Documentation/Splunk/latest/Data/UsetheHTTPEventCollector)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Nginx Stream Module](https://nginx.org/en/docs/stream/ngx_stream_core_module.html)

---

**Fecha de creación:** 2024-12-04  
**Última actualización:** 2024-12-04  
**Autor:** Hugo Raber  
**Ambiente:** Testing (HTTP) → Producción (HTTPS)


asoidjhasoidjasiodjasiodj
