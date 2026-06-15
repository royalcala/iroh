# Iroh — Overview

Iroh es una biblioteca de networking peer-to-peer en Rust (v1.0.0-rc.1) construida sobre QUIC. Su principio central es **"dial by public key"**: le dices a Iroh "conéctate a ese peer" y él encuentra y mantiene la conexión más rápida posible, haciendo hole-punching automático y usando relay como fallback.

---

## ¿Cómo funciona?

### Identidad

Cada dispositivo genera un `SecretKey` (Ed25519) y deriva su `EndpointId` (clave pública). Esta clave es tu identidad criptográfica: se usa tanto para que otros te encuentren como para autenticar las conexiones TLS. No hay PKI tradicional — los peers se autentican directamente por clave pública.

```rust
let secret_key = SecretKey::generate();
let endpoint_id: EndpointId = secret_key.public();
```

### Conexión punto a punto

Para conectar dos peers se necesita:

1. **Ambos lados** generan o cargan su `SecretKey` y comparten un mismo ALPN (protocolo de aplicación).
2. **El lado que acepta** imprime o comparte su `EndpointId` y su `RelayUrl`.
3. **El lado que conecta** usa esos datos para llamar a `endpoint.connect(addr, ALPN)`.

```rust
// Lado que acepta
let endpoint = Endpoint::builder(presets::N0)
    .secret_key(secret_key)
    .alpns(vec![b"mi-app/0".to_vec()])
    .relay_mode(RelayMode::Default)
    .bind().await?;

endpoint.online().await;
let addr_info = endpoint.addr();
// addr_info.id() -> EndpointId
// addr_info.relay_urls() -> RelayUrl

while let Some(incoming) = endpoint.accept().await {
    let conn = incoming.accept()?.await?;
    // ...
}
```

```rust
// Lado que conecta
let endpoint = Endpoint::builder(presets::N0)
    .secret_key(secret_key)
    .alpns(vec![b"mi-app/0".to_vec()])
    .relay_mode(RelayMode::Default)
    .bind().await?;

endpoint.online().await;

let addr = EndpointAddr::from_parts(remote_id, addrs);
let conn = endpoint.connect(addr, b"mi-app/0").await?;
```

### Descubrimiento automático (DNS / pkarr)

Con `presets::N0`, el lado que conecta puede usar **solo el `EndpointId`** sin necesidad de saber el relay URL ni IPs. Iroh automáticamente:

1. Consulta `dns.iroh.link` por registros TXT firmados (formato pkarr).
2. Extrae el `RelayUrl` y direcciones directas del peer remoto.
3. Intenta conexión directa (hole-punching vía relay).
4. Si falla, hace fallback a relay.

El lado que acepta publica automáticamente su `RelayUrl` en `dns.iroh.link/pkarr`.

### Relays

Existen **relays públicos operados por Number 0** en 4 regiones:

| Región | Hostname |
|--------|----------|
| NA Este | `use1-1.relay.n0.iroh.link` |
| NA Oeste | `usw1-1.relay.n0.iroh.link` |
| Europa | `euc1-1.relay.n0.iroh.link` |
| Asia-Pacífico | `aps1-1.relay.n0.iroh.link` |

No necesitas correr tu propio relay. Iroh se conecta automáticamente al relay geográficamente más cercano. Si quieres correr el tuyo, el binario es `iroh-relay`.

> **¿Cómo elige el relay?** El `RelayMap` puede contener múltiples URLs. Al iniciar, el endpoint hace latency probing (probes HTTPS) contra **todos** los relays del mapa y selecciona automáticamente el de menor latencia como **home relay**. Si la latencia de otro relay mejora ≥33%, cambia. La conexión al home relay se mantiene siempre viva; las demás se crean bajo demanda cuando un peer remoto tiene ese relay como su home.

#### ¿Los relays públicos tienen límites?

Sí. El servidor relay aplica **rate limiting por cliente** (token-bucket) sobre el tráfico entrante (`bytes_per_second` + `max_burst_bytes`). El control de acceso por defecto es `Everyone` — cualquiera puede conectarse sin token.

Sin embargo, hay una limitación importante: el límite de **conexiones simultáneas** (`accept_conn_limit`) **no está implementado aún** (el campo existe en el struct `Limits` pero tiene el comentario `"Not currently implemented, setting this has no effect."` en `iroh-relay/src/server.rs:491`). Esto significa que no hay un hard cap de cuántos clientes pueden estar conectados a la vez al relay — la contención real depende de los recursos del servidor.

El key cache del servidor está dimensionado para **1 millón de clientes concurrentes** (~56 MB en 64-bit, `iroh-relay/src/defaults.rs:19`).

#### ¿Qué pasa en WASM / navegador?

En navegadores no hay UDP ni hole-punching posible. La conexión **siempre va por relay**, usando **WebSocket** en lugar de QUIC:

- El cliente WASM convierte `https://` → `wss://` y se conecta vía `ws_stream_wasm` (`iroh-relay/src/client.rs:387-443`).
- El auth token (si se requiere) se envía como `?token=` query param porque los navegadores no permiten headers custom en WebSocket.
- No hay handshake TLS exportable desde WASM, ni `DnsResolver`, ni proxy support.

Para apps en navegador, la carga sobre los relays públicos es mayor porque todo el tráfico pasa por ellos. Para producción con muchos clientes WASM se recomienda correr tu propio relay.

### Grupos / Mallas

El crate `iroh` solo maneja conexiones **punto a punto**. Para funcionalidad de grupos existen crates externos:

- **`iroh-gossip`** — Redes overlay pub-sub (broadcast a múltiples peers).
- **`iroh-blobs`** — Transferencia de contenido direccionado por hash (BLAKE3).
- **`iroh-docs`** — Key-value store eventualmente consistente.

### Presets

`presets::N0` configura automáticamente:

- Crypto provider (`ring` o `aws-lc-rs`)
- Publicación pkarr a `dns.iroh.link`
- Resolución DNS y pkarr desde `dns.iroh.link`
- Relays de producción
