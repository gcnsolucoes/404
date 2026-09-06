# gcnsolucoes/404

Páginas de erro HTTP bonitas e prontas para produção — no estilo [vixpi.host](https://vixpi.host).  
Zero dependências externas. Dark mode nativo. Funciona com nginx, Caddy, Traefik e qualquer servidor web.

---

## Páginas disponíveis

| Código | Título | Uso |
|--------|--------|-----|
| `400` | Requisição Inválida | Parâmetros malformados |
| `401` | Não Autorizado | Autenticação necessária |
| `403` | Acesso Proibido | Sem permissão |
| `404` | Página Não Encontrada | Rota inexistente |
| `405` | Método Não Permitido | Verbo HTTP inválido |
| `408` | Tempo Limite Esgotado | Request timeout |
| `429` | Muitas Requisições | Rate limit atingido |
| `500` | Erro Interno do Servidor | Falha genérica do servidor |
| `502` | Gateway Inválido | Upstream inválido |
| `503` | Serviço Indisponível | Manutenção / sobrecarga |
| `504` | Tempo Limite do Gateway | Gateway timeout |

Todas as páginas são arquivos HTML autocontidos — sem CDN, sem fontes externas, sem JavaScript de terceiros.

---

## Configuração rápida

### nginx

Copie a pasta `pages/` para o servidor e adicione ao bloco `server {}`:

```nginx
# /etc/nginx/conf.d/default.conf
server {
    listen 80;
    root /var/www/html;

    error_page 400 /errors/400.html;
    error_page 401 /errors/401.html;
    error_page 403 /errors/403.html;
    error_page 404 /errors/404.html;
    error_page 405 /errors/405.html;
    error_page 408 /errors/408.html;
    error_page 429 /errors/429.html;
    error_page 500 /errors/500.html;
    error_page 502 /errors/502.html;
    error_page 503 /errors/503.html;
    error_page 504 /errors/504.html;

    location ^~ /errors/ {
        internal;
        root /usr/share/nginx;
    }
}
```

Monte as páginas em `/usr/share/nginx/errors/` dentro do container:

```yaml
# docker-compose.yml
services:
  nginx:
    image: nginx:1.27-alpine
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./pages:/usr/share/nginx/errors:ro
```

Exemplo completo: [`examples/nginx/`](examples/nginx/)

---

### Caddy

```caddy
# Caddyfile
example.com {
    root * /var/www/html
    file_server

    handle_errors {
        rewrite * /errors/{err.status_code}.html
        file_server {
            root /srv
        }
    }
}
```

```yaml
# docker-compose.yml
services:
  caddy:
    image: caddy:2.9-alpine
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./pages:/srv/errors:ro
```

Exemplo completo: [`examples/caddy/`](examples/caddy/)

---

### Traefik

Traefik usa um **service dedicado** para servir as páginas de erro via middleware `errors`.

**1. Suba um container nginx servindo as páginas:**

```yaml
# docker-compose.yml
services:
  traefik:
    image: traefik:v3.3
    ports:
      - "80:80"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik.yml:/etc/traefik/traefik.yml:ro
      - ./dynamic:/etc/traefik/dynamic:ro

  error-pages:
    image: nginx:1.27-alpine
    volumes:
      - ./pages:/usr/share/nginx/html:ro

  app:
    image: nginx:1.27-alpine
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app.rule=Host(`example.com`)"
      - "traefik.http.routers.app.middlewares=error-pages@file"
```

**2. Configure o middleware em `dynamic/error-pages.yml`:**

```yaml
http:
  middlewares:
    error-pages:
      errors:
        status:
          - "400-599"
        service: error-pages-svc
        query: "/{status}.html"

  services:
    error-pages-svc:
      loadBalancer:
        servers:
          - url: "http://error-pages:80"
```

Exemplo completo: [`examples/traefik/`](examples/traefik/)

---

### Apache

```apache
# .htaccess ou VirtualHost
ErrorDocument 400 /errors/400.html
ErrorDocument 401 /errors/401.html
ErrorDocument 403 /errors/403.html
ErrorDocument 404 /errors/404.html
ErrorDocument 500 /errors/500.html
ErrorDocument 502 /errors/502.html
ErrorDocument 503 /errors/503.html
ErrorDocument 504 /errors/504.html

<Directory /var/www/html/errors>
    Options -Indexes
    AllowOverride None
</Directory>
```

---

## Estrutura do repositório

```
.
├── pages/
│   ├── 400.html
│   ├── 401.html
│   ├── 403.html
│   ├── 404.html
│   ├── 405.html
│   ├── 408.html
│   ├── 429.html
│   ├── 500.html
│   ├── 502.html
│   ├── 503.html
│   └── 504.html
└── examples/
    ├── nginx/
    │   ├── nginx.conf
    │   └── docker-compose.yml
    ├── caddy/
    │   ├── Caddyfile
    │   └── docker-compose.yml
    └── traefik/
        ├── traefik.yml
        ├── docker-compose.yml
        └── dynamic/
            └── error-pages.yml
```

---

## Design

- Dark mode automático via `prefers-color-scheme`
- Sem dependências externas — funciona offline e em ambientes restritos
- Tipografia do sistema (`ui-sans-serif`, `ui-monospace`)
- Páginas 4xx com acento âmbar, 5xx com acento vermelho
- Fundo com grid de pontos sutil
- Identidade visual [GCN Soluções](https://gcnsolucoes.com.br) / [vixpi.host](https://vixpi.host)

---

## Licença

MIT — livre para uso comercial e pessoal.

---

Desenvolvido por [GCN Soluções](https://gcnsolucoes.com.br) · Powered by [vixpi.host](https://vixpi.host)
