# Self-Hosting Iroh

Para un despliegue propio completo necesitas correr dos servicios: **relay** y **DNS/pkarr**.

---

## Arquitectura

```
┌──────────────────────────────────────────────────┐
│                  Tu infraestructura               │
│                                                   │
│  ┌──────────────┐       ┌──────────────────┐     │
│  │ iroh-relay   │       │ iroh-dns-server   │     │
│  │ :443 (HTTPS) │       │ :443 (HTTPS)      │     │
│  │ :7842 (QUIC) │       │ :53  (DNS UDP)    │     │
│  │ :9090 (metr) │       │ :9090 (metrics)   │     │
│  └──────┬───────┘       └────────┬─────────┘     │
│         │                        │                │
│         │ hole-punch + relay     │ publish/resolve│
│         │ tráfico de datos       │ EndpointInfo   │
│         │                        │                │
└─────────┼────────────────────────┼────────────────┘
          │                        │
    ┌─────┴─────┐           ┌─────┴─────┐
    │  Peer A   │           │  Peer B   │
    │ (cliente) │           │ (cliente) │
    └───────────┘           └───────────┘
```

- **Relay**: coordina hole-punching y retransmite datos cuando no hay conexión directa.
- **DNS/pkarr**: permite que los peers publiquen su `EndpointInfo` (RelayUrl, IPs) y que otros lo resuelvan con solo saber el `EndpointId`.

---

## 1. Relay Server (`iroh-relay`)

### Build

```bash
cargo build --release -p iroh-relay
# o con Docker:
docker buildx build -f docker/Dockerfile --target iroh-relay -t iroh-relay .
```

### Configuración mínima (`relay.toml`)

```toml
# Para dev local sin TLS
http_bind_addr = "127.0.0.1:3340"

# Para producción con Let's Encrypt
[tls]
cert_mode = "LetsEncrypt"
hostname = "relay.ejemplo.com"
cert_dir = "/srv/iroh-relay/certs"
contact = "admin@ejemplo.com"
prod_tls = true

# Rate limiting (recomendado)
[limits.client.rx]
bytes_per_second = 1048576    # 1 MB/s por cliente
max_burst_bytes = 2097152     # 2 MB burst
```

### Run

```bash
iroh-relay --config-path relay.toml
```

Puertos expuestos: `80` (HTTP), `443` (HTTPS con TLS), `7842` (QUIC addr discovery), `9090` (métricas).

### Control de acceso

| Modo | Config |
|------|--------|
| Público | `access = "everyone"` |
| Token compartido | `access.shared_token = ["mi-token"]` |
| Allowlist | `access.allowlist = ["abc123..."]` |
| Denylist | `access.denylist = ["abc123..."]` |

### Docker

```bash
docker run -v ./relay.toml:/config/relay.toml \
  -p 80:80 -p 443:443 -p 7842:7842/udp -p 9090:9090 \
  iroh-relay --config-path /config/relay.toml
```

### Selección automática de relay por latencia

El endpoint **no** usa un relay fijo ni un round-robin. Al iniciar, hace **latency probing** contra **todos** los relays del `RelayMap` y elige automáticamente el de menor latencia como **home relay**.

```
RelayMap = [relay-us, relay-eu, relay-asia]

Inicio → probe HTTPS a los 3 → latencias: us=20ms, eu=90ms, asia=180ms
                               → home relay = relay-us

Si relay-us cae o su latencia sube → re-sondeo → home relay = relay-eu
```

Detalles del mecanismo (`iroh/src/net_report.rs`):

- Se envían **3 probes HTTPS** a cada relay del mapa al iniciar.
- Se mide la latencia mínima de cada relay (IPv4, IPv6, HTTPS).
- Se selecciona el relay con **menor latencia promedio en los últimos 5 minutos**.
- **Hysteresis**: no cambia de relay a menos que el nuevo sea **≥33% mejor** que el actual (evita flapping).
- La conexión al home relay **nunca se cierra por inactividad**. Las conexiones a otros relays se eliminan tras 60s sin uso.
- Al enviar datos a un peer remoto, se usa **el home relay del peer remoto** (descubierto vía DNS/pkarr), no el tuyo. Si no tienes conexión activa a ese relay, se crea una bajo demanda.

**Conclusión práctica**: agregas todos tus relays al `RelayMap` y el sistema elige el mejor automáticamente. Si despliegas relays en varias regiones (ej. `relay-us.ejemplo.com`, `relay-eu.ejemplo.com`), cada cliente usará el más cercano.

---

## 2. DNS Server (`iroh-dns-server`)

### Build

```bash
cargo build --release -p iroh-dns-server
# o con Docker:
docker buildx build -f docker/Dockerfile --target iroh-dns-server -t iroh-dns-server .
```

### Configuración (`dns.toml`)

Hay dos ejemplos en el repo: `iroh-dns-server/config.dev.toml` (desarrollo) y `config.prod.toml` (producción).

```toml
# Dev local
[http]
port = 8080
bind_addr = "127.0.0.1"

[https]
port = 8443
bind_addr = "127.0.0.1"
domains = ["localhost"]
cert_mode = "self_signed"

[dns]
port = 5300
bind_addr = "127.0.0.1"
origins = ["irohdns.example.", "."]
```

```toml
# Producción
[https]
port = 443
domains = ["irohdns.ejemplo.org"]
cert_mode = "lets_encrypt"
letsencrypt_prod = true
letsencrypt_contact = "admin@ejemplo.org"

[dns]
port = 53
origins = ["irohdns.ejemplo.org", "."]
rr_ns = "ns1.irohdns.ejemplo.org."
```

### Run

```bash
iroh-dns-server --config dns.toml
```

Puertos: `53` (DNS UDP), `443` (HTTPS con endpoint `/pkarr`), `9090` (métricas).

---

## 3. Cliente: apuntar a tu infraestructura

Una vez corriendo ambos servidores, configuras los clientes para que usen tus URLs en vez de las de n0:

```rust
use iroh::{Endpoint, RelayMode, RelayMap};
use iroh::endpoint::presets::Minimal;
use iroh::address_lookup::{PkarrPublisher, DnsAddressLookup, PkarrRelayClient};
use iroh_relay::relay_map::RelayUrl;

// 1. Relays custom (múltiples = elige el más rápido automáticamente)
let relay_map = RelayMap::from_iter([
    "https://relay-us.ejemplo.com".parse::<RelayUrl>().unwrap(),
    "https://relay-eu.ejemplo.com".parse::<RelayUrl>().unwrap(),
    "https://relay-asia.ejemplo.com".parse::<RelayUrl>().unwrap(),
]);
// Si usas SharedToken:
let relay_map = relay_map.with_auth_token("mi-token");

// 2. DNS/pkarr custom
let pkarr_url = "https://irohdns.ejemplo.org/pkarr".parse().unwrap();
let dns_origin = "irohdns.ejemplo.org".to_string();

let endpoint = Endpoint::builder(Minimal)
    .secret_key(secret_key)
    .alpns(vec![b"mi-app/0".to_vec()])
    .relay_mode(RelayMode::Custom(relay_map))
    // Publicar en tu pkarr relay (lado que acepta)
    .address_lookup(PkarrPublisher::new(pkarr_url.clone()))
    // Resolver vía DNS (lado que conecta)
    .address_lookup(DnsAddressLookup::new(dns_origin))
    // Resolver vía HTTP pkarr (navegadores)
    .address_lookup(PkarrRelayClient::new(pkarr_url))
    .bind()
    .await?;
```

**Sin DNS server propio**: si solo quieres correr tu relay y seguir usando `dns.iroh.link` para descubrimiento, omite las líneas de `PkarrPublisher` y `DnsAddressLookup` y usa `presets::N0` con `RelayMode::Custom`:

```rust
let endpoint = Endpoint::builder(presets::N0)
    .relay_mode(RelayMode::Custom(relay_map))
    .bind()
    .await?;
```

---

## 4. Despliegue mínimo viable

| Componente | ¿Obligatorio? | Notas |
|------------|:---:|-------|
| Relay propio | No | Usar relays públicos de n0 funciona, pero sin garantías de escala/rate limits |
| DNS server propio | No | Usar `dns.iroh.link` funciona. Solo necesario si quieres independencia total |
| Ambos | No | Combinar relay propio + DNS público, o viceversa, es válido |

Para producción con clientes WASM (navegador), se recomienda relay propio porque todo el tráfico pasa por relay.
