+++
date = '2025-09-27T14:25:43-04:00'
title = 'Why You Can’t Terminate TLS at Traefik for PostgreSQL (and What to Do Instead)'
+++

# Context

I had the need to allow **Power BI** to connect to a **PostgreSQL database** running in Kubernetes, fronted by **Traefik** using a **TCP entrypoint**.

At first, I hoped to **terminate TLS at Traefik**, the same way you’d do for HTTPS traffic. But this turned out not to be possible with standard PostgreSQL clients (psql, libpq, psycopg, etc.). Here’s why.

# Why This Happens

Unlike HTTPS, PostgreSQL does **not** start a TLS handshake immediately.  
Instead, a libpq/psql client first sends a **special SSLRequest packet**:

1.  Client → Server: *“Do you support SSL?”* (SSLRequest packet)
    
2.  Server → Client: responds with a single byte `S` (accept) or `N` (reject).
    
3.  If accepted, client → server: *now* sends a proper TLS `ClientHello`.
    

The key point: **Postgres TLS is negotiated in-band, not at connection start.**

Traefik’s TLS termination, on the other hand, expects the client to send a TLS `ClientHello` right away (like a browser or `openssl s_client`).  
When it instead receives a PostgreSQL SSLRequest, it doesn’t know what to do — so it responds incorrectly (with an `H`/garbage), and the connection fails.

# The Two Realistic Options

## **Option A — TLS passthrough (recommended, simple, works with all clients)**

Let **PostgreSQL handle TLS itself**. Traefik just forwards the TCP stream without trying to decrypt it.

That way:

*   The SSLRequest reaches Postgres.
    
*   Postgres replies `S`.
    
*   Client and Postgres negotiate TLS directly.
    

Example config:

```yaml
# New Traefik entry point
traefik:
  ports:
    postgresql:
      port: 15432
      expose:
        default: true
      exposedPort: 15432
      protocol: TCP

# Application ingress route
apiVersion: traefik.io/v1alpha1
kind: IngressRouteTCP
metadata:
  name: ingressroutetcpfoo
spec:
  entryPoints:
    - postgresql
  routes:
  - match: HostSNI(`*`)
    services:
    - name: postgres
      port: 5432
```

Verification:

```bash
docker run -it --rm -e PGPASSWORD=examplepass \
  postgres:14 \
  psql "host=xxx.domain.com port=15432 \
        user=exampleuser dbname=exampledb sslmode=require"
```

Output:

```bash
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
exampledb=# \dt;
```

✅ Works with psql, psycopg, Power BI, etc.

Note: you can find the complete Kubernetes manifests in my GitHub [repo](https://github.com/hoaraujerome/blog-secure-postgres-k8s-traefik).

## **Option B — Terminate TLS at Traefik (not recommended)**

This only works if the client **skips the SSLRequest** and starts with a raw TLS ClientHello.  
Standard Postgres clients don’t do this.

You could theoretically use:

*   a custom client (Python, Go, etc.) that wraps the TCP socket in TLS before speaking the Postgres protocol, or
    
*   an intermediate Postgres-aware proxy (e.g., HAProxy, Envoy) that understands the SSLRequest and handles it correctly.
    

But for most use cases, this is **too complex and brittle**.

Here’s a proof-of-concept Python script that does it: it wraps the socket in TLS first, then speaks Postgres over that channel. That works with Traefik TLS termination, but it’s not compatible with standard drivers.

```python
#!/usr/bin/env python3
"""
Connect to a Postgres server via Traefik TLS termination.

This script:
 - opens a TLS socket to the host:port (sends ClientHello)
 - sends a Postgres StartupMessage (protocol 3.0)
 - handles Authentication (cleartext or MD5)
 - sends a simple SELECT query and prints results

Usage:
  python pg_tls_via_traefik.py \
    --host XXX.domain.com \
    --port 15432 \
    --user exampleuser \
    --password examplepass \
    --dbname exampledb
"""
import ssl
import socket
import struct
import argparse
import hashlib

def recv_n(sock, n):
    buf = b""
    while len(buf) < n:
        chunk = sock.recv(n - len(buf))
        if not chunk:
            raise EOFError("Connection closed unexpectedly")
        buf += chunk
    return buf

def read_message(sock):
    # Postgres message framing (after startup): [type(1)][len(4)][payload(len-4)]
    type_byte = sock.recv(1)
    if not type_byte:
        return None, None
    length_bytes = recv_n(sock, 4)
    length = struct.unpack("!I", length_bytes)[0]
    payload = recv_n(sock, length - 4) if length > 4 else b""
    return type_byte, payload

def read_until_ready_for_query(sock):
    """Read messages until ReadyForQuery 'Z' appears. Returns all collected messages."""
    msgs = []
    while True:
        t, payload = read_message(sock)
        if t is None:
            break
        msgs.append((t, payload))
        if t == b'Z':  # ReadyForQuery
            break
    return msgs

def send_startup(conn_sock, user, dbname):
    # StartupMessage: length(4) + protocol(4) + params (key\0value\0 ... \0)
    params = []
    params.append(b"user")
    params.append(user.encode())
    if dbname:
        params.append(b"database")
        params.append(dbname.encode())
    # join with nulls and final null
    body = struct.pack("!I", 196608)  # protocol 3.0
    for i in range(0, len(params), 2):
        body += params[i] + b'\x00' + params[i+1] + b'\x00'
    body += b'\x00'  # terminator
    total_len = 4 + len(body)
    packet = struct.pack("!I", total_len) + body
    conn_sock.sendall(packet)

def handle_auth(sock, user, password):
    """
    Read an Authentication request, handle cleartext or MD5.
    Returns True if authenticated OK; raises on failure.
    """
    # read messages until AuthenticationOk or error
    while True:
        t, payload = read_message(sock)
        if t is None:
            raise RuntimeError("server closed connection while authenticating")
        if t == b'R':  # Authentication request
            # payload first 4 bytes = auth code
            auth_code = struct.unpack("!I", payload[:4])[0]
            if auth_code == 0:
                # AuthenticationOk
                return True
            elif auth_code == 3:
                # Cleartext password requested
                pwd_bytes = password.encode() + b'\x00'
                plen = 4 + len(pwd_bytes)
                msg = b'p' + struct.pack("!I", plen) + pwd_bytes
                sock.sendall(msg)
                # continue loop to get AuthenticationOk
            elif auth_code == 5:
                # MD5: next 4 bytes = salt
                salt = payload[4:8]
                # MD5 algorithm: pwd_md5 = md5(password + user).hexdigest()
                # then response = 'md5' + md5(pwd_md5 + salt).hexdigest()
                inner = hashlib.md5(password.encode() + user.encode()).hexdigest().encode()
                outer = hashlib.md5(inner + salt).hexdigest().encode()
                resp = b"md5" + outer
                resp_payload = resp + b'\x00'
                plen = 4 + len(resp_payload)
                msg = b'p' + struct.pack("!I", plen) + resp_payload
                sock.sendall(msg)
            else:
                raise RuntimeError(f"Unsupported auth method: {auth_code}")
        elif t == b'E':
            # ErrorResponse
            # payload is a sequence of fields ended by 0; extract the message field 'M'
            print("ErrorResponse from server:", payload)
            raise RuntimeError("Server returned ErrorResponse during auth")
        elif t == b'S':
            # ParameterStatus: ignore
            continue
        elif t == b'K':
            # BackendKeyData
            continue
        elif t == b'Z':
            # ReadyForQuery without explicit AuthenticationOk? treat as ok
            return True
        else:
            # other messages, ignore for now
            continue

def run_query(sock, query):
    # send simple Query message: 'Q' + int32 len + query\0
    payload = query.encode() + b'\x00'
    plen = 4 + len(payload)
    msg = b'Q' + struct.pack("!I", plen) + payload
    sock.sendall(msg)

    # read responses until ReadyForQuery:
    rows = []
    columns = []
    while True:
        t, payload = read_message(sock)
        if t is None:
            break
        if t == b'T':  # RowDescription
            # payload: int16 field count, then per-field: name \0, tableOID(4), columnAttr(2), typeOID(4), typeSize(2), typeModifier(4), format(2)
            idx = 0
            field_count = struct.unpack("!H", payload[idx:idx+2])[0]; idx += 2
            columns = []
            for _ in range(field_count):
                # read null-terminated name
                j = payload.find(b'\x00', idx)
                name = payload[idx:j].decode(); idx = j+1
                # skip the rest of fixed-size fields (18 bytes)
                idx += 18
                columns.append(name)
        elif t == b'D':  # DataRow
            # payload: int16 column count, then per column: int32 length, data
            idx = 0
            col_count = struct.unpack("!H", payload[idx:idx+2])[0]; idx += 2
            values = []
            for _ in range(col_count):
                length = struct.unpack("!I", payload[idx:idx+4])[0]; idx += 4
                if length == 0xFFFFFFFF:
                    values.append(None)
                else:
                    val = payload[idx:idx+length].decode(); idx += length
                    values.append(val)
            rows.append(values)
        elif t == b'C':
            # CommandComplete; ignore
            pass
        elif t == b'E':
            # Error
            print("Query Error:", payload)
            break
        elif t == b'Z':
            # ReadyForQuery -> done
            break
        else:
            # ignore other message types
            pass
    return columns, rows

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--host", required=True)
    parser.add_argument("--port", type=int, default=15432)
    parser.add_argument("--user", required=True)
    parser.add_argument("--password", required=True)
    parser.add_argument("--dbname", required=True)
    args = parser.parse_args()

    context = ssl.create_default_context()
    # If you want to skip server cert validation (NOT recommended), do:
    # context.check_hostname = False
    # context.verify_mode = ssl.CERT_NONE

    # Connect TCP first
    with socket.create_connection((args.host, args.port), timeout=10) as sock:
        # wrap SSL (this sends a TLS ClientHello immediately)
        with context.wrap_socket(sock, server_hostname=args.host) as ssock:
            print("TLS established with", ssock.version())
            # Now speak Postgres protocol INSIDE TLS
            send_startup(ssock, args.user, args.dbname)
            # handle auth (reads messages until AuthenticationOk or error)
            ok = handle_auth(ssock, args.user, args.password)
            if not ok:
                raise RuntimeError("Authentication failed or not completed")
            print("Authenticated successfully (Postgres believes we're connected).")
            # run a simple query listing public tables
            cols, rows = run_query(ssock, "SELECT table_name FROM information_schema.tables WHERE table_schema='public';")
            print("Columns:", cols)
            for r in rows:
                print(r)
            print("Done.")

if __name__ == "__main__":
    main()
```

# Lessons Learned

*   PostgreSQL’s TLS negotiation is **protocol-specific** and not compatible with generic TLS termination.
    
*   **TLS passthrough is the right way** when putting Postgres behind Traefik.
    
*   Always enforce SSL-only in `pg_hba.conf` using `hostssl` and `sslmode=verify-full` on the client side.
    
*   If you *must* terminate TLS at the edge, use a Postgres-aware proxy — but be prepared for extra complexity.
