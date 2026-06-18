# QUIC Stream Invite — Lessons Learned

Lo que aprendimos implementando notificaciones P2P vía `endpoint.connect()` + `ProtocolHandler`.

## Bugs que tuvimos (y cómo evitarlos)

### 1. Un Router, 4+ handlers → los últimos son ignorados ⚠️ BUG CONFIRMADO

**Síntoma:** `connect()` se cuelga (deadline elapsed). El handler nunca recibe la conexión.

**Causa:** Confirmado con test `test_invite_in_single_router`. Registrar 4+ ALPNs en el mismo `Router::builder` hace que el último handler nunca sea despachado. El `Router::spawn()` llama a `Endpoint::set_alpns(alpn_list)` que envía todos los ALPNs al TLS server config vía `noq_proto::ServerConfig::set_alpn_protocols()`. 

El `ProtocolMap` es un `BTreeMap` sin límites (`iroh/src/protocol.rs:377`). El `Router::accept()` no tiene límites (`iroh/src/protocol.rs:484`). El `handle_connection()` verifica el ALPN y busca en el mapa sin restricciones (`iroh/src/protocol.rs:625-661`).

La causa raíz probablemente está en la capa TLS/QUIC (`noq`), no en iroh. La negociación de ALPN en la handshake TLS puede tener un límite en el número de protocolos que el servidor anuncia. TLS 1.3 soporta múltiples ALPNs, pero implementaciones específicas podrían truncar la lista.

**NOTA:** Cuando el invite handler está en un Router SEPARADO, el `set_alpns()` de ese Router **sobrescribe** el server config del endpoint (`iroh/src/endpoint.rs:930`). El endpoint solo anuncia los ALPNs del último Router que hizo spawn. Esto significa que si el invite Router hace spawn después que el docs Router, el endpoint SOLO anuncia `/syntrix/invite/1` y las conexiones de docs/gossip/blobs deberían fallar. Sin embargo, en la práctica las conexiones ya establecidas no se ven afectadas porque solo las nuevas handshakes TLS usan el nuevo server config.

**Fix:** Router dedicado para el invite handler, separado del Router de docs/gossip/blobs.

```rust
// ❌ No funciona
Router::builder(ep)
    .accept(blobs_alpn, blobs)
    .accept(gossip_alpn, gossip)
    .accept(docs_alpn, docs)
    .accept(invite_alpn, invite)  // NUNCA recibe
    .spawn();

// ✅ Funciona
Router::builder(ep)
    .accept(blobs_alpn, blobs)
    .accept(gossip_alpn, gossip)
    .accept(docs_alpn, docs)
    .spawn();

Router::builder(ep)           // Router separado
    .accept(invite_alpn, invite)  // AHORA recibe
    .spawn();
```

### 2. Router no almacenado → el accept loop muere

**Síntoma:** La conexión llega pero es rechazada con "aborted by peer during handshake".

**Causa:** `Router::spawn()` retorna un handle. Si no se almacena (ej. en `AppState`), Rust lo droppea al salir del scope y el accept loop se cancela.

**Fix:** Guardar el Router en la struct que vive mientras la app corre.

```rust
// ❌ Muere al salir de new()
async fn new() -> Self {
    let _router = Router::builder(ep).accept(alpn, handler).spawn();
    // _router droppeado aquí
}

// ✅ Vive mientras AppState viva
struct AppState {
    _router: Router,  // guardado
}
```

### 3. TLS cert verification entre procesos

**Síntoma:** "aborted by peer: handshake failed" cuando dos procesos separados intentan conectarse.

**Causa:** Cada proceso genera su propio certificado TLS autofirmado. En el mismo proceso, la pila QUIC comparte el contexto TLS y no valida. En procesos separados, el lado que conecta rechaza el cert del lado que acepta.

**Fix para dev:** `CaRootsConfig::insecure_skip_verify()` + feature `test-utils` en `iroh`.

```toml
# Cargo.toml
iroh = { version = "=1.0.0-rc.1", features = ["test-utils"] }
```

```rust
Endpoint::builder(N0)
    .ca_roots_config(CaRootsConfig::insecure_skip_verify())
    .bind().await?
```

**Fix para prod:** Usar certificados reales vía Let's Encrypt en un relay propio, o mantener `insecure_skip_verify` solo para dev con `#[cfg(debug_assertions)]`.

### 4. Race condition: admin cierra antes de que client lea

**Síntoma:** El admin reporta éxito, el client recibe la conexión pero `read_to_end()` falla con "closed by peer".

**Causa:** `open_uni()` + `write_all()` + `finish()` son asíncronos. Si el admin droppea la conexión inmediatamente después de `finish()`, el paquete puede no haberse entregado antes de que el client llame `accept_uni()`.

**Fix:** El admin espera a que el client cierre la conexión primero.

```rust
// ❌ Race condition
let mut send = conn.open_uni().await?;
send.write_all(&data).await?;
send.finish()?;
// conn droppeado → client pierde datos

// ✅ Sin race
let mut send = conn.open_uni().await?;
send.write_all(&data).await?;
send.finish()?;
let _ = conn.closed().await;  // espera al client
```

## Patrón probado que funciona

```rust
// ── Client (acepta) ──
let ep = Endpoint::builder(N0)
    .secret_key(key)
    .ca_roots_config(CaRootsConfig::insecure_skip_verify())
    .bind_addr("127.0.0.1:0".parse()?)?
    .bind().await?;

let invite_router = Router::builder(ep.clone())  // Router separado
    .accept(b"/syntrix/invite/1", invite_handler)
    .spawn();
// invite_router guardado en AppState

// ── Admin (conecta) ──
let conn = ep.connect(peer, b"/syntrix/invite/1").await
    .or_else(|_| ep.connect(EndpointAddr::from_parts(peer, addrs), alpn)).await?;

let mut send = conn.open_uni().await?;
send.write_all(&payload).await?;
send.finish()?;
let _ = conn.closed().await;  // espera al client
```

## DNS discovery no siempre funciona

`connect(peer, alpn)` — sin direcciones explícitas — depende de DNS TXT records (pkarr) publicados en `dns.iroh.link`. Cuando el cliente llama `ep.online().await`, publica su `EndpointInfo` (relay URL + direct addresses) en el DHT de pkarr. El admin, al llamar `connect(peer, alpn)`, consulta `dns.iroh.link` por los TXT records del peer.

**Problema:** en nuestra red, esta resolución DNS tarda >10 segundos o directamente falla. Testeado con `test_dns_discovery` — `connect(peer, alpn)` timeoutea consistentemente. Para una UI interactiva (el admin hace clic en Send Invite y espera respuesta), >10s es inaceptable.

**Solución:** compartir el `EndpointAddr` completo del cliente como JSON, para que el admin haga `connect(EndpointAddr::from_parts(peer, addrs), alpn)` sin depender de DNS.

### El JSON de EndpointAddr

```json
{
  "addrs": [
    "relay:https://use1-1.relay.n0.iroh-canary.iroh.link./",
    "ip:127.0.0.1:43525",
    "ip:192.168.1.179:43525",
    "ip:[2806:103e:16:c28c::4]:58328"
  ],
  "node_id": "56f322092380e71f78bbbd80ceca7036bc8b8aba515c8c30c663833d86f90788"
}
```

| Campo | Significado | Uso |
|-------|-------------|-----|
| `node_id` | Hash Ed25519 (32 bytes hex) de la llave pública del dispositivo | Identidad del peer. El admin lo usa para construir `PublicKey`. |
| `addrs` | Lista de `TransportAddr`: relay URLs + direcciones IP del dispositivo | El admin construye `EndpointAddr::from_parts(peer, addrs)` y llama `connect()`. |

**`relay:URL`** — la URL del relay donde el cliente está conectado (n0 público, o nuestro relay en producción). Permite conexión vía relay si la directa falla.

**`ip:HOST:PORT`** — direcciones IP directas (localhost, LAN, WAN, IPv6). El admin intenta conexión directa por estas IPs primero.

### Flujo

```
1. Client: ep.addr() → serializa a JSON → muestra al usuario
2. Usuario: copia el JSON → lo pega en el admin
3. Admin: parsea JSON → extrae node_id + addrs
4. Admin: endpoint.connect(EndpointAddr::from_parts(peer, addrs), ALPN)
5. Iroh: intenta directo por IPs locales (127.0.0.1, 192.168.x.x)
6. Si falla: intenta por relay (relay URL)
7. Conexión establecida en <1s
```

**Para producción:** con relay + DNS propios, `connect(peer, alpn)` resolverá en <1s y no será necesario compartir el JSON. El admin solo necesitará el `node_id`.
