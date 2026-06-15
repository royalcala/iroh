# Iroh — Documentación

## Índice

- **[Overview](overview.md)** — ¿Qué es Iroh? Identidad, conexión punto a punto, descubrimiento, relays, presets.

### Conceptos

- [Modelo de Identidad](identity.md) — `SecretKey`, `PublicKey` / `EndpointId`, `Signature`.
- [Arquitectura de Transportes](transports.md) — UDP, Relay (DERP), Custom Transports.
- [Modelo de Conexión](connection.md) — `Endpoint`, `Connection`, `Connecting`, `Accepting`.
- [ALPN y Protocolos](protocols.md) — `Router`, `ProtocolHandler`.

### Crates

- [`iroh-base`](iroh-base.md) — Tipos base: `EndpointId`, `RelayUrl`, `TransportAddr`, `CustomAddr`.
- [`iroh` (core)](iroh.md) — Biblioteca principal: `Endpoint`, transportes, TLS, métricas.
- [`iroh-relay`](iroh-relay.md) — Cliente y servidor del protocolo relay.
- [`iroh-dns`](iroh-dns.md) — Descubrimiento de endpoints vía DNS/pkarr.
- [`iroh-dns-server`](iroh-dns-server.md) — Servidor DNS + pkarr relay (`dns.iroh.link`).

### Flujos

- [Establecimiento de Conexión](connection-setup.md) — Hole-punching, relay fallback, handshake TLS.
- [Protocolo Relay](relay-protocol.md) — Frames, handshake, streams.
- [Descubrimiento](discovery.md) — DNS, pkarr, DHT. Publicación y resolución de direcciones.
- [Path Selection](path-selection.md) — `BiasedRttPathSelector`, múltiples caminos.

### Servidores

- [Self-Hosting](self-hosting.md) — Cómo montar tu propia infraestructura (relay + DNS).
- [Relay Server](relay-server.md) — Control de acceso, rate limiting, TLS.
- [DNS Server](dns-server.md) — DNS-over-HTTPS, pkarr relay, persistencia `redb`.

### Seguridad

- [TLS en Iroh](tls.md) — TLS 1.3, certificados efímeros, verificación por `EndpointId`.
- [Modelo de Confianza](trust.md) — Autenticación directa peer-to-peer.

### Plataformas

- [Soporte de Plataformas](platforms.md) — Linux, macOS, Windows, WASM, Android.
- [Feature Flags](features.md) — `portmapper`, `metrics`, `tls-ring`, `unstable-custom-transports`.

### Observabilidad

- [Métricas](metrics.md) — Prometheus en `Endpoint` y relay server.
- [NetReport](net-report.md) — Análisis de alcanzabilidad de red y NAT.
- [Port Mapping](portmapper.md) — UPnP/PCP.

### Desarrollo

- [Estructura del Proyecto](project-structure.md) — Workspace, directorios, build.
- [Testing](testing.md) — Integración, `patchbay`, proptest, traced tests.
- [CI/CD](ci-cd.md) — GitHub Actions, builds multi-plataforma.

### Ecosistema

- [Crates externos](ecosystem.md) — `iroh-blobs`, `iroh-gossip`, `iroh-docs`, `iroh-ffi`, `iroh-doctor`, `iroh-tor`.
