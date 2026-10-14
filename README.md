# Roteiro do Seu Jeito — site

Landing page estática de **roteirodoseujeito.com.br**, servida por [Caddy](https://caddyserver.com)
em Docker Compose, com HTTPS automático (Let's Encrypt).

```
site/            # o site (é só isso que você edita no dia a dia)
  index.html
  404.html
  favicon.svg
  robots.txt
  sitemap.xml
  assets/        # imagens das páginas do roteiro
Caddyfile              # configuração do servidor (modo padrão, com HTTPS)
Caddyfile.proxy        # configuração para rodar atrás de outro proxy
docker-compose.yml     # stack padrão (portas 80 e 443)
docker-compose.proxy.yml  # override para rodar atrás de outro proxy
.env.example           # modelo de variáveis; copie para .env
```

## Antes de publicar

1. **DNS**: no painel do seu registrador, aponte o domínio para o IP do servidor:

   | Tipo | Nome  | Valor            |
   |------|-------|------------------|
   | A    | `@`   | IP do servidor   |
   | A    | `www` | IP do servidor   |

   O `www` é redirecionado para o domínio raiz. Espere o DNS propagar antes de subir
   a stack — o Let's Encrypt só emite o certificado depois que o domínio resolve.

2. **Firewall**: as portas **80** e **443** (TCP, e 443/UDP para HTTP/3) precisam estar
   abertas e livres. Se já houver um nginx/Apache ocupando a 80, veja
   "Atrás de um proxy existente" mais abaixo.

## Publicar

No servidor, com Docker e o plugin Compose instalados:

```bash
git clone <url-do-repositorio> roteirodoseujeito
cd roteirodoseujeito
cp .env.example .env     # confira DOMAIN e ACME_EMAIL
docker compose up -d
```

Pronto. O Caddy pede o certificado sozinho e já redireciona HTTP → HTTPS.
Acompanhe a emissão com:

```bash
docker compose logs -f
```

## Atualizar o site

O conteúdo é um bind mount: não precisa rebuildar imagem.

```bash
git pull                 # ou edite site/index.html direto no servidor
docker compose restart   # opcional, só para limpar cache interno
```

O HTML é servido com `max-age=300`, então uma alteração aparece em até 5 minutos
para quem já visitou. As imagens em `assets/` têm hash no nome e cache de 1 ano —
ao trocar uma imagem, use um nome novo.

## Atrás de um proxy existente

Se o servidor já tem nginx, Traefik ou Cloudflare Tunnel cuidando do TLS:

```bash
docker compose -f docker-compose.yml -f docker-compose.proxy.yml up -d
```

O site passa a responder só em `http://127.0.0.1:8080` (ajuste com `HTTP_PORT` no `.env`),
sem tentar emitir certificado. Aponte o seu proxy para essa porta. Exemplo de nginx:

```nginx
location / {
    proxy_pass http://127.0.0.1:8080;
    proxy_set_header Host              $host;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

> O override usa a tag `!override` do Compose, que precisa do Compose v2.24 ou mais novo
> (`docker compose version`).

## Comandos úteis

```bash
docker compose ps                  # estado e healthcheck
docker compose logs -f             # logs de acesso e do ACME
docker compose restart             # reiniciar
docker compose down                # parar (os certificados ficam no volume)
docker compose pull && docker compose up -d   # atualizar o Caddy
```

## Detalhes

- **Certificados** ficam no volume `caddy_data`. Não apague o volume: recriar
  certificado com frequência esbarra no rate limit do Let's Encrypt.
- **Cabeçalhos de segurança** (HSTS, CSP, nosniff, Referrer-Policy) estão no `Caddyfile`.
  A CSP libera o Google Fonts e os estilos/script inline da página — se adicionar
  um script externo (pixel, analytics), inclua o domínio dele na diretiva `script-src`.
- **Compressão** gzip/zstd está ligada: a home sai com ~10 KB.
- **Pagamento** é externo (Kiwify); o site não processa dados de cartão nem tem formulário.
- A imagem está presa na major `caddy:2-alpine`. Para builds idênticos no tempo,
  troque por uma tag exata (ex.: `caddy:2.8.4-alpine`).
