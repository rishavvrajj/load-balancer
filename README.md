# Go Load Balancer

A simple HTTP load balancer in Go using round-robin distribution with health checks.

## What It Does

- Distributes incoming HTTP requests across multiple backend servers
- Uses round-robin algorithm to cycle through available servers
- Health checks backends before forwarding requests (2s timeout)
- Skips unavailable servers automatically

## Project Structure

```
.
├── main.go           # Entry point, configures backends
├── loadbalancer.go   # Load balancer with round-robin logic
├── server.go         # Server interface and health check
└── backend/
    └── main.go       # Test backend server
```

## Quick Start

### 1. Run Multiple Backend Servers

Open 3 terminals and run:

```bash
go run backend/main.go -port 9001
go run backend/main.go -port 9002
go run backend/main.go -port 9003
```

### 2. Update Load Balancer Config

Edit `main.go` to point to your local backends:

```go
addresses := []string{
    "http://localhost:9001",
    "http://localhost:9002",
    "http://localhost:9003",
}
```

### 3. Start Load Balancer

```bash
go run .
```

Load balancer starts on `http://localhost:8000`

### 4. Test It

```bash
curl http://localhost:8000
# Hello from backend on port 9001

curl http://localhost:8000
# Hello from backend on port 9002

curl http://localhost:8000
# Hello from backend on port 9003
```

Requests are distributed evenly across backends.

## How It Works

### Round-Robin Algorithm

```go
func (lb *LoadBalancer) GetAvailableServer() (Server, bool) {
    server := lb.servers[lb.roundRobinCount % len(lb.servers)]
    lb.roundRobinCount++
    
    if server.IsAlive() {
        return server, true
    }
    // Try next server if current is down
}
```

Cycles through servers using modulo arithmetic. Increments counter on each request.

### Health Check

```go
func (s *SimpleServer) IsAlive() bool {
    client := http.Client{Timeout: 2 * time.Second}
    resp, err := client.Get(s.addr)
    if err != nil {
        return false
    }
    return resp.StatusCode < 500
}
```

Pings each server before forwarding. Returns false if timeout or 5xx error.

### Reverse Proxy

Uses `httputil.ReverseProxy` to forward requests:

```go
proxy := httputil.NewSingleHostReverseProxy(serverURL)
proxy.ServeHTTP(rw, req)
```

Handles all HTTP details (headers, body, streaming).

## Build From Scratch

### Step 1: Server Interface

Define what a backend server needs:

```go
type Server interface {
    Address() string
    IsAlive() bool
    Serve(http.ResponseWriter, *http.Request)
}
```

### Step 2: Implement Server

```go
type SimpleServer struct {
    addr  string
    proxy *httputil.ReverseProxy
}

func NewSimpleServer(addr string) (*SimpleServer, error) {
    serverURL, _ := url.Parse(addr)
    proxy := httputil.NewSingleHostReverseProxy(serverURL)
    return &SimpleServer{addr: addr, proxy: proxy}, nil
}
```

### Step 3: Load Balancer Logic

```go
type LoadBalancer struct {
    roundRobinCount int
    servers         []Server
    mu              sync.Mutex
}

func (lb *LoadBalancer) ServeHTTP(rw http.ResponseWriter, req *http.Request) {
    server, ok := lb.GetAvailableServer()
    if !ok {
        http.Error(rw, "No available servers", 503)
        return
    }
    server.Serve(rw, req)
}
```

### Step 4: Wire It Up

```go
func main() {
    servers := []Server{/* create servers */}
    lb := NewLoadBalancer("8000", servers)
    http.ListenAndServe(":8000", lb)
}
```

## Testing Failure Handling

1. Stop one backend (Ctrl+C)
2. Send requests - they skip the dead backend
3. Restart it - automatically included again

## Requirements

- Go 1.20+
- No external dependencies (uses stdlib only)

## Common Issues

**"No available backend servers"**
- Check backends are running (`ps aux | grep backend`)
- Verify addresses in `main.go` match running ports

**Connection refused**
- Backends must start before load balancer
- Use `http://` not `https://` for local backends

## Next Steps

- Add weighted round-robin
- Implement least-connections algorithm
- Add metrics/monitoring
- Support HTTPS backends
- Graceful shutdown handling
