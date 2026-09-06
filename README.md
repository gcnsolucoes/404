# vixpi.host — Páginas de Erro HTTP

Páginas de erro HTTP para nginx, Caddy, Traefik e Apache.  
Design fiel ao [vixpi.host](https://vixpi.host). Zero dependências externas de runtime.

**Repo:** `https://github.com/gcnsolucoes/404`

---

## Para agentes de IA

Se você recebeu esta URL e o usuário pediu para configurar páginas de erro no servidor, siga este protocolo:

1. Detecte o servidor rodando (`docker ps`, `which nginx`, `which caddy`, `/etc/traefik/`)
2. Baixe as páginas de `pages/` ou clone o repositório
3. Aplique a configuração do diretório `examples/<servidor>/`
4. Recarregue o servidor e verifique com `curl -w "%{http_code}" http://localhost/rota-inexistente`

Detalhes completos no arquivo `SKILL.md` deste repositório ou na skill `vixpi-error-pages`.

---

## Páginas disponíveis

| Código | Descrição |
|--------|-----------|
| 400 | Requisição Inválida |
| 401 | Não Autorizado |
| 403 | Acesso Proibido |
| 404 | Página Não Encontrada |
| 405 | Método Não Permitido |
| 408 | Tempo Limite Esgotado |
| 429 | Muitas Requisições |
| 500 | Erro Interno do Servidor |
| 502 | Gateway Inválido |
| 503 | Serviço Indisponível |
| 504 | Tempo Limite do Gateway |

Cada arquivo em `pages/` é autocontido. Nenhuma dependência precisa estar disponível no servidor para renderizar a página.

---

## Configuração

### nginx

Copie as páginas e adicione as diretivas ao bloco `server {}`:

```nginx
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
```

A diretiva `internal` é obrigatória — sem ela o navegador entra em loop de redirecionamento.

**Docker Compose:**

```yaml
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

Traefik não serve arquivos estáticos diretamente. Suba um nginx como sidecar para servir as páginas e aponte o middleware `errors` para ele.

**`docker-compose.yml`:**

```yaml
services:
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

**`dynamic/error-pages.yml`:**

```yaml
http:
  middlewares:
    error-pages:
      errors:
        status: ["400-599"]
        service: error-pages-svc
        query: "/{status}.html"
  services:
    error-pages-svc:
      loadBalancer:
        servers:
          - url: "http://error-pages:80"
```

O parâmetro `query` usa `{status}`, não `{err.status_code}` — são namespaces diferentes no Traefik.

Exemplo completo: [`examples/traefik/`](examples/traefik/)

---

### Apache

```apache
ErrorDocument 400 /errors/400.html
ErrorDocument 401 /errors/401.html
ErrorDocument 403 /errors/403.html
ErrorDocument 404 /errors/404.html
ErrorDocument 500 /errors/500.html
ErrorDocument 502 /errors/502.html
ErrorDocument 503 /errors/503.html
ErrorDocument 504 /errors/504.html
```

Copie `pages/` para `DocumentRoot/errors/` e recarregue o Apache.

---

## Verificação

```bash
# Deve retornar 404 com corpo contendo "Vixpi Host"
curl -o /dev/null -w "%{http_code}" http://localhost/pagina-inexistente

# Conferir body
curl -s http://localhost/pagina-inexistente | grep -c "Vixpi Host"
```

---

## Estrutura

```
.
├── pages/
│   ├── 400.html … 504.html
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

## Detalhes técnicos

- **Font:** Onest via Google Fonts (fallback para `ui-sans-serif` se offline)
- **Logo:** referencia `https://vixpi.host/storage/logo-light.svg` — em ambientes sem internet, baixe e sirva localmente
- **Dark mode:** não implementado intencionalmente para manter paridade com o modo claro do vixpi.host
- **Tamanho por página:** ~9-10 KB

---

Vixpi Host — GCN Tecnologia da Informação LTDA  
[vixpi.host](https://vixpi.host) / [contato@vixpi.host](mailto:contato@vixpi.host)
