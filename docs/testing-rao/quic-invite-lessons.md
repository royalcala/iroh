# QUIC Stream Invite — Lessons Learned

Lo que aprendimos implementando notificaciones P2P vía `endpoint.connect()` + `ProtocolHandler`.

## Bugs que tuvimos (y cómo evitarlos)

### 1. Un Router, muchos handlers → el último es invisible

**Síntoma:** `connect()` funciona pero el handler nunca recibe la conexión.

**Causa:** Registrar 4+ protocol handlers en el mismo `Router::builder` hace que iroh ignore los últimos. Probablemente un límite interno de `accept()` loop.

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

`connect(peer, alpn)` — sin direcciones explícitas — depende de DNS TXT records (pkarr). En redes donde `dns.iroh.link` no resuelve o tiene latencia alta, falla con "No addressing information available".

**Solución:** Pasar `EndpointAddr::from_parts(peer, addrs)` con direcciones explícitas (localhost IPs + relay URLs).
