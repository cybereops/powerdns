# PowerDNS Recursor

Deploy do [PowerDNS Recursor](https://www.powerdns.com/powerdns-recursor) via Docker Compose com configuração em YAML nativo (formato exigido a partir da versão 5.x).

## Estrutura

```
recursor/
├── docker-compose.yml
└── recursor.yml
```

---

## docker-compose.yml

```yaml
services:
  pdns-recursor:
    image: powerdns/pdns-recursor-53:5.3.5
    container_name: pdns-recursor
    ports:
      - "5351:53/udp"
      - "5351:53/tcp"
    volumes:
      - ./recursor.yml:/etc/powerdns/recursor.yml:ro
    restart: unless-stopped

  dns-tools:
    image: busybox:latest
    container_name: dns-tools
    entrypoint: ["sh"]
    stdin_open: true
    tty: true
    depends_on:
      - pdns-recursor
```

| Campo | Valor | Descrição |
|---|---|---|
| `image` | `powerdns/pdns-recursor-53:5.3.5` | Imagem oficial com suporte a porta 53 |
| `ports` | `5351:53` | Porta do host mapeada para a 53 interna do container |
| `volumes` | `./recursor.yml` montado em `/etc/powerdns/recursor.yml` | Configuração injetada como somente leitura |
| `dns-tools` | `busybox:latest` | Container auxiliar para diagnósticos com `nslookup` |

> A porta `5351` é usada no host para evitar conflito com o `systemd-resolved`, que ocupa a `53` por padrão no Ubuntu/Debian. Dentro da rede Docker a comunicação entre containers continua na porta `53`.

---

## recursor.yml

```yaml
dnssec:
  validation: off

incoming:
  listen:
    - 0.0.0.0:53
  allow_from:
    - 0.0.0.0/0

logging:
  loglevel: 6
  common_errors: true
  trace: true

recursor:
  forward_zones_recurse:
    - zone: "."
      forwarders:
        - 8.8.8.8
        - 8.8.4.4
```

### Referências da documentação

| Diretiva | Seção na documentação |
|---|---|
| `dnssec.validation` | [DNSSEC settings](https://doc.powerdns.com/recursor/settings.html#dnssec-settings) |
| `incoming.listen` | [Incoming settings](https://doc.powerdns.com/recursor/settings.html#incoming-settings) |
| `incoming.allow_from` | [Incoming settings](https://doc.powerdns.com/recursor/settings.html#incoming-settings) |
| `logging.loglevel` | [Logging settings](https://doc.powerdns.com/recursor/settings.html#logging-settings) |
| `logging.common_errors` | [Logging settings](https://doc.powerdns.com/recursor/yamlsettings.html#setting-yaml-logging-loglevel) |
| `logging.trace` | [Logging settings](https://doc.powerdns.com/recursor/settings.html#logging-settings) |
| `recursor.forward_zones_recurse` | [Recursor settings](https://doc.powerdns.com/recursor/settings.html#recursor-settings) |

### Descrição de cada diretiva

**`dnssec.validation: off`**
Desativa a validação DNSSEC. Adequado para ambientes de laboratório. Em produção o valor recomendado é `validate`.

**`incoming.listen: 0.0.0.0:53`**
Define o endereço e porta onde o recursor aceita consultas. `0.0.0.0` escuta em todas as interfaces do container.

**`incoming.allow_from: 0.0.0.0/0`**
Controla quais origens podem consultar o recursor. `0.0.0.0/0` aceita qualquer endereço — restrinja para produção (ex: `192.168.0.0/16`).

**`logging.loglevel: 6`**
Nível de verbosidade dos logs. Escala de 0 (silencioso) a 9 (máximo). O nível `6` exibe informações operacionais sem excesso de ruído.

**`logging.common_errors: true`**
Loga erros comuns de resolução, como timeouts e respostas malformadas.

**`logging.trace: true`**
Loga cada consulta processada com detalhes da resolução recursiva. Útil para diagnóstico — desative em produção pelo volume gerado.

**`recursor.forward_zones_recurse`**
Define para onde encaminhar consultas que o recursor não resolve localmente. A zona `.` cobre todos os domínios. Os forwarders `8.8.8.8` e `8.8.4.4` são os resolvers públicos do Google.

---

## Deploy

```bash
docker compose up -d
```

Verificar status:

```bash
docker compose ps
```

---

## Testes

### nslookup via container dns-tools

Consulta padrão (registro A):

```bash
docker exec -it dns-tools nslookup google.com pdns-recursor
```

Consulta de registro MX:

```bash
docker exec -it dns-tools nslookup -type=MX gmail.com pdns-recursor
```
Consulta de registro SOA:
```bash
docker exec -it dns-tools nslookup -type=SOA gmail.com pdns-recursor
```

Consulta reversa (PTR):

```bash
docker exec -it dns-tools nslookup 8.8.8.8 pdns-recursor
```

### Logs em tempo real

```bash
docker logs -f pdns-recursor
```

---

## Encerrando

```bash
docker compose down
```
