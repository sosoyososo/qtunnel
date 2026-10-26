# qTunnel Protocol Documentation

## Overview

qTunnel is a lightweight encrypted tunnel program working as a wrapper between clients and servers. It's designed as a simpler and faster alternative to Stunnel/stud, without SSL certificate management.

## Project Structure

```
qtunnel/
├── src/
│   ├── main.go           # Entry point, CLI arguments parsing
│   └── tunnel/
│       ├── tunnel.go     # Core tunnel logic
│       ├── conn.go       # Connection wrapper with encryption
│       ├── cipher.go     # Encryption implementation (RC4/AES256-CFB)
│       ├── recycler.go   # Memory pool management
│       └── *_test.go     # Unit tests
├── README.md
└── Makefile
```

## Protocol Details

### Connection Flow

1. **Server Mode**: Listens on a port, receives encrypted data, decrypts and forwards to backend
2. **Client Mode**: Listens locally, encrypts data and forwards to server

```
# Server (host-a)
qtunnel -listen=:6378 -backend=127.1:6379 -secret=secret -crypto=rc4

# Client (host-b)
qtunnel -listen=127.1:6379 -backend=host-a:6378 -clientmode=true -secret=secret -crypto=rc4
```

### Data Flow

```
Client App <---> qTunnel Client (encrypt) <---> qTunnel Server (decrypt) <---> Backend Service
```

### No Handshake Protocol

Unlike SSL/TLS, qTunnel has **no handshake**. It directly:
- Client encrypts all outgoing data
- Server decrypts all incoming data
- Pure stream-based encryption (no message boundaries)

## Encryption Methods

### 1. RC4 (Default)

- **Key Length**: 16 bytes (128-bit)
- **Implementation**: Go's `crypto/rc4`

### 2. AES256-CFB

- **Key Length**: 32 bytes (256-bit)
- **Block Cipher**: AES
- **Mode**: CFB (Cipher Feedback)
- **IV**: First 16 bytes of the derived key

## Key Derivation

```go
func secretToKey(secret []byte, size int) []byte {
    // Uses MD5 to expand secret to required key size
    h := md5.New()
    buf := make([]byte, size)
    count := size / md5.Size  // 16 bytes per MD5 hash
    for i := 0; i < count; i++ {
        h.Write(secret)
        copy(buf[md5.Size*i:md5.Size*(i+1)-1], h.Sum(nil))
    }
    return buf
}
```

- **RC4**: key size = 16 bytes (1 iteration of MD5)
- **AES256**: key size = 32 bytes (2 iterations of MD5)
- For AES-CFB, IV = first `block.BlockSize()` bytes of key (16 bytes)

## Encryption/Decryption Interface

```go
type Cipher struct {
    enc cipher.Stream  // encryption
    dec cipher.Stream  // decryption
}

func (c *Cipher) encrypt(dst, src []byte) {
    c.enc.XORKeyStream(dst, src)
}

func (c *Cipher) decrypt(dst, src []byte) {
    c.dec.XORKeyStream(dst, src)
}
```

## Connection Management

### Read Operation (conn.go:22-32)

```go
func (c *Conn) Read(b []byte) (int, error) {
    c.conn.SetReadDeadline(time.Now().Add(30 * time.Minute))
    if c.cipher == nil {
        return c.conn.Read(b)
    }
    n, err := c.conn.Read(b)
    if n > 0 {
        c.cipher.decrypt(b[0:n], b[0:n])
    }
    return n, err
}
```

- 30-minute read timeout
- In-place decryption
- Stream-based (no message framing)

### Write Operation (conn.go:34-40)

```go
func (c *Conn) Write(b []byte) (int, error) {
    if c.cipher == nil {
        return c.conn.Write(b)
    }
    c.cipher.encrypt(b, b)
    return c.conn.Write(b)
}
```

- In-place encryption
- Direct passthrough to underlying connection

## CLI Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `-listen` | `:9001` | Frontend listening address (host:port) |
| `-backend` | `127.0.0.1:6400` | Backend address (host:port) |
| `-crypto` | `rc4` | Encryption method (`rc4` or `aes256cfb`) |
| `-secret` | `secret` | Shared password for encryption |
| `-clientmode` | `false` | Run as client (encrypt) vs server (decrypt) |
| `-logto` | `stdout` | Logging output (`stdout` or `syslog`) |

## iOS Implementation Notes

### Dependencies Needed

1. **CommonCrypto** (iOS native) for AES and MD5
2. **Security** framework for cryptographic operations
3. No RC4 support in iOS - need custom implementation or use alternative

### iOS Crypto Mapping

| qTunnel | iOS Equivalent |
|---------|---------------|
| RC4 | Custom RC4 implementation or use `CCCryptorCreate` with `kCCAlgorithmRC4` |
| AES256-CFB | `CCCryptorCreate` with `kCCAlgorithmAES128`, `kCCModeCFB` |
| MD5 | `CC_MD5` functions |

### Swift Implementation Skeleton

```swift
class QTCipher {
    private var encContext: CCCryptorRef?
    private var decContext: CCCryptorRef?

    init(secret: [UInt8], method: String) {
        let key = secretToKey(secret: secret, size: method == "rc4" ? 16 : 32)
        // Initialize encryption/decryption contexts
    }

    func encrypt(_ data: [UInt8]) -> [UInt8] {
        // In-place encryption using CCCrypt
    }

    func decrypt(_ data: [UInt8]) -> [UInt8] {
        // In-place decryption using CCCrypt
    }
}
```

### Important Considerations

1. **No Protocol Headers**: qTunnel is a pure stream wrapper
2. **30-min Timeout**: Server expects periodic data or keepalive
3. **Same Secret Required**: Both client and server must use identical `-secret`
4. **Same Crypto Method**: Both must use same `-crypto` setting
5. **TCP Only**: Works over TCP connections

### Connection Setup

```swift
// Client side (clientmode=true)
let clientSocket = TCPClient()
clientSocket.connect(host: "server", port: 9001)
let cipher = QTCipher(secret: "secret", method: "rc4")

// Wrap socket writes with encryption
let encryptedData = cipher.encrypt(originalData)
clientSocket.write(encryptedData)

// Wrap socket reads with decryption
let encryptedResponse = clientSocket.read()
let response = cipher.decrypt(encryptedResponse)
```

## Security Considerations

- **No Authentication**: Only password-based encryption
- **No Forward Secrecy**: Same key used for all sessions
- **RC4 Weakness**: RC4 has known biases; prefer AES256-CFB for production
- **MD5 for Key Derivation**: MD5 is broken for collision resistance but still used here for key expansion only

## Testing

Run tests:
```bash
make test
```

## References

- Original project: https://github.com/sosoyososo/qtunnel
- Go crypto package: https://pkg.go.dev/crypto
- iOS CommonCrypto: https://developer.apple.com/documentation/security/1399296-common_cryptor
