#!/usr/bin/env python3
import socket
import select
from socketserver import ThreadingMixIn, TCPServer, StreamRequestHandler

class ProxyHandler(StreamRequestHandler):
    def handle(self):
        # Read the client request line
        first_line = self.rfile.readline().decode('utf-8', errors='ignore')
        if not first_line:
            return

        method, path, protocol = first_line.strip().split()

        if method.upper() == "CONNECT":
            # HTTPS proxying
            host, port = path.split(":")
            port = int(port)
            try:
                # Connect to target
                remote = socket.create_connection((host, port))
                # Tell client the tunnel is ready
                self.wfile.write(b"HTTP/1.1 200 Connection Established\r\n\r\n")
                # Tunnel data in both directions
                self._tunnel(self.connection, remote)
            except Exception as e:
                self.wfile.write(b"HTTP/1.1 502 Bad Gateway\r\n\r\n")
        else:
            # HTTP proxying (GET, POST, etc.)
            # Read and forward headers
            headers = {}
            while True:
                line = self.rfile.readline().decode('utf-8', errors='ignore')
                if line in ('\r\n', '\n', ''):
                    break
                key, value = line.split(":", 1)
                headers[key.strip().lower()] = value.strip()

            # Determine target host and port
            host_header = headers.get("host", "")
            if ":" in host_header:
                host, port = host_header.split(":", 1)
                port = int(port)
            else:
                host, port = host_header, 80

            try:
                remote = socket.create_connection((host, port))
                # Rewrite request line to use absolute path
                remote.write(f"{method} {path} {protocol}\r\n".encode())

                # Forward all headers
                for h, v in headers.items():
                    remote.write(f"{h}: {v}\r\n".encode())
                remote.write(b"\r\n")

                # If there’s a request body, forward that too
                if 'content-length' in headers:
                    body = self.rfile.read(int(headers['content-length']))
                    remote.write(body)

                # Tunnel the rest
                self._tunnel(self.connection, remote)
            except Exception:
                self.send_error(502, "Bad Gateway")

    def _tunnel(self, client_sock, remote_sock):
        sockets = [client_sock, remote_sock]
        while True:
            r, _, _ = select.select(sockets, [], [])
            for s in r:
                data = s.recv(4096)
                if not data:
                    return
                # Send to the other side
                if s is client_sock:
                    remote_sock.sendall(data)
                else:
                    client_sock.sendall(data)

class ThreadedTCPServer(ThreadingMixIn, TCPServer):
    allow_reuse_address = True

if __name__ == "__main__":
    import sys
    port = int(sys.argv[1]) if len(sys.argv) > 1 else 8888
    server = ThreadedTCPServer(("0.0.0.0", port), ProxyHandler)
    print(f"[*] Open proxy listening on port {port}")
    server.serve_forever()

