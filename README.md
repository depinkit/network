# Network: Decentralized P2P Networking for Distributed Systems

`network` is a comprehensive networking package for building decentralized peer-to-peer systems. It provides libp2p-based communication, DHT discovery, gossipsub pub/sub messaging, and IP-over-libp2p virtual networking capabilities. This package forms the networking foundation for distributed applications requiring secure, NAT-traversing communication in trustless environments.

**Origin:** This package was extracted from and is actively used by [NuNet Device Management Service (DMS)](https://gitlab.com/nunet/device-management-service), where it serves as the core networking layer for coordinating compute resources across a decentralized network.

## Overview

The `network` package provides:
- **libp2p Integration**: Full-featured libp2p host with DHT, gossipsub, and custom protocols
- **P2P Discovery**: Automatic peer discovery via DHT and rendezvous points
- **Virtual Networking**: IP-over-libp2p tunneling for transparent network connectivity between peers
- **NAT Traversal**: Automatic relay and hole-punching for connectivity behind NATs
- **Pub/Sub Messaging**: Topic-based broadcasting using gossipsub
- **Background Tasks**: Scheduled tasks for peer discovery and network maintenance
- **Configuration Management**: Flexible configuration with validation and defaults

## Installation

```bash
go get github.com/depinkit/network
```

## Quick Start

### Basic P2P Node

```go
package main

import (
    "context"
    "time"
    
    "github.com/depinkit/network"
    "github.com/depinkit/network/config"
    "github.com/depinkit/network/libp2p"
    "github.com/spf13/afero"
    "gitlab.com/nunet/device-management-service/types"
)

func main() {
    // Create configuration
    cfg := &config.Config{
        P2P: config.P2P{
            ListenAddress: []string{
                "/ip4/0.0.0.0/tcp/9000",
                "/ip4/0.0.0.0/udp/9001/quic-v1",
            },
            BootstrapPeers: []string{
                // Bootstrap peer addresses
            },
        },
    }
    
    // Create libp2p configuration
    libp2pCfg := &types.Libp2pConfig{
        PrivateKey:     generatePrivKey(), // Your key generation
        Rendezvous:     "my-network",
        ListenAddress:  cfg.P2P.ListenAddress,
        BootstrapPeers: parseBootstrapPeers(cfg.P2P.BootstrapPeers),
    }
    
    // Create libp2p instance
    fs := afero.NewOsFs()
    l, err := libp2p.NewLibp2p(libp2pCfg, fs)
    if err != nil {
        panic(err)
    }
    
    // Initialize and start
    if err := l.Init(cfg); err != nil {
        panic(err)
    }
    if err := l.Start(); err != nil {
        panic(err)
    }
    
    defer l.Stop()
    
    // Your application logic here
    select {}
}
```

### Custom Protocol Handler

```go
// Register a custom protocol handler
handler := func(stream network.Stream) {
    defer stream.Close()
    
    // Read from stream
    buf := make([]byte, 1024)
    n, err := stream.Read(buf)
    if err != nil {
        return
    }
    
    // Process and respond
    response := processMessage(buf[:n])
    stream.Write(response)
}

l.SetStreamHandler("/my-protocol/1.0.0", handler)
```

### Pub/Sub Messaging

```go
import (
    "github.com/depinkit/network/libp2p"
)

// Subscribe to a topic
topic := "/my-app/messages"
if err := l.Subscribe(context.Background(), topic); err != nil {
    panic(err)
}

// Add handler for messages on this topic
validator := func(msg *libp2p.Message) bool {
    // Validate message
    return true
}

l.RegisterTopicValidator(topic, 0, validator, libp2p.ValidatorOptions{})

// Publish a message
message := []byte("Hello, network!")
if err := l.Publish(context.Background(), topic, message); err != nil {
    log.Error("Failed to publish")
}
```

### Virtual Network (IP-over-libp2p)

```go
// Create a virtual subnet
subnetCfg := &libp2p.SubnetConfig{
    CIDR:      "10.0.0.0/24",
    Routes:    map[string]peer.ID{
        "10.0.0.2": peerID,
    },
    DNSZones: map[string]string{
        "app.local": "10.0.0.2",
    },
}

subnet, err := l.CreateSubnet(context.Background(), "my-subnet", subnetCfg)
if err != nil {
    panic(err)
}
defer subnet.Close()

// Now you can use standard networking to connect to peers
// via their virtual IPs (e.g., dial tcp://10.0.0.2:8080)
```

## Package Structure

```
github.com/depinkit/network/
├── config/              - Configuration management
│   ├── config.go        - Config structures
│   ├── load.go          - Viper-based config loading
│   └── config_test.go
├── background_tasks/    - Task scheduling
│   ├── scheduler.go     - Background task scheduler
│   ├── task.go          - Task definitions
│   ├── trigger.go       - Periodic, cron, event triggers
│   └── *_test.go
├── libp2p/              - libp2p implementation
│   ├── libp2p.go        - Main libp2p host
│   ├── host.go          - Host creation and setup
│   ├── dht.go           - DHT operations
│   ├── discover.go      - Peer discovery
│   ├── subnet.go        - Virtual networking
│   ├── dns.go           - DNS for virtual networks
│   └── *_test.go
├── utils/               - Utility functions
│   └── utils.go         - Port allocation, network utils
├── network.go           - Network interface definitions
├── vnet.go              - Virtual network implementation
└── types.go             - Common types
```

## Core Components

### Network Interface

The `Network` interface defines the contract for network implementations:

```go
type Network interface {
    Init(cfg *config.Config) error
    Start() error
    Stop() error
    Host() host.Host
    DHT() *dht.IpfsDHT
}
```

### Libp2p Implementation

The `libp2p.Libp2p` type provides a full-featured implementation with:
- **DHT**: Distributed hash table for content and peer routing
- **Gossipsub**: Efficient topic-based pub/sub
- **Relay**: Automatic relay for NAT traversal
- **AutoNAT**: NAT detection and configuration
- **Hole Punching**: Direct connections through NATs
- **Custom Protocols**: Register handlers for custom protocols
- **Virtual Networks**: IP tunneling over libp2p connections

### Configuration

Configuration is flexible and supports multiple sources:

```go
cfg := &config.Config{
    General: config.General{
        DataDir: "/var/lib/myapp",
        Debug:   true,
    },
    P2P: config.P2P{
        ListenAddress: []string{
            "/ip4/0.0.0.0/tcp/9000",
            "/ip4/0.0.0.0/udp/9001/quic-v1",
        },
        BootstrapPeers: []string{
            "/ip4/bootstrap.example.com/tcp/4001/p2p/12D3...",
        },
        Memory:          512,  // MB for libp2p
        FileDescriptors: 2048,
    },
}
```

### Background Tasks

The scheduler manages periodic tasks like peer discovery:

```go
import "github.com/depinkit/network/background_tasks"

scheduler := background_tasks.NewScheduler(10, time.Second)

task := &background_tasks.Task{
    Name:        "Heartbeat",
    Description: "Send periodic heartbeat",
    Function: func(_ interface{}) error {
        return sendHeartbeat()
    },
    Triggers: []background_tasks.Trigger{
        &background_tasks.PeriodicTrigger{
            Interval: 30 * time.Second,
        },
    },
}

scheduler.AddTask(task)
scheduler.Start()
defer scheduler.Stop()
```

## Advanced Features

### DHT Operations

```go
// Provide a value in the DHT
key := "/myapp/value"
value := []byte("important data")
if err := l.DHT().PutValue(ctx, key, value); err != nil {
    log.Error("Failed to put value")
}

// Find a value in the DHT
val, err := l.DHT().GetValue(ctx, key)
if err != nil {
    log.Error("Failed to get value")
}
```

### Peer Discovery

```go
// Find peers providing a service
peers, err := l.FindPeers(ctx, "my-service", 10)
if err != nil {
    log.Error("Discovery failed")
}

for _, p := range peers {
    // Connect to peer
    if err := l.Host().Connect(ctx, p); err != nil {
        log.Warnf("Failed to connect to %s", p.ID)
    }
}
```

### Connection Management

```go
// Get connected peers
peers := l.Host().Network().Peers()
log.Infof("Connected to %d peers", len(peers))

// Get connection info
for _, p := range peers {
    conns := l.Host().Network().ConnsToPeer(p)
    for _, conn := range conns {
        log.Infof("Connection to %s via %s", 
            p, conn.RemoteMultiaddr())
    }
}
```

## Integration with DMS

This package was extracted from [NuNet's Device Management Service](https://gitlab.com/nunet/device-management-service) and remains a core dependency. In DMS, it provides:

- **Node Communication**: Enables compute nodes to discover and communicate with each other
- **Job Distribution**: Network substrate for distributing compute jobs across the network
- **Virtual Networking**: Allows containerized workloads to communicate as if on the same network
- **Secure Channels**: Establishes encrypted communication channels between nodes
- **NAT Traversal**: Enables connectivity for nodes behind NATs and firewalls

### Using with DMS Types

Since this package uses some types from DMS, you'll need to import them:

```go
import (
    "github.com/depinkit/network/libp2p"
    "gitlab.com/nunet/device-management-service/types"
)

libp2pCfg := &types.Libp2pConfig{
    // Configuration
}
```

## Dependencies

### Core Dependencies
- **libp2p**: `github.com/libp2p/go-libp2p`
- **DHT**: `github.com/libp2p/go-libp2p-kad-dht`
- **PubSub**: `github.com/libp2p/go-libp2p-pubsub`
- **QUIC**: `github.com/quic-go/quic-go`

### Depinkit Dependencies
- **Crypto**: `github.com/depinkit/crypto` - Cryptographic operations
- **DID**: `github.com/depinkit/did` - Decentralized identifiers

### DMS Dependencies
- **Types**: `gitlab.com/nunet/device-management-service/types`
- **Observability**: `gitlab.com/nunet/device-management-service/observability`
- **System Utils**: `gitlab.com/nunet/device-management-service/utils/sys`

## Testing

Run the full test suite:

```bash
go test ./...
```

Run tests for a specific package:

```bash
go test ./libp2p
go test ./config
```

Run with coverage:

```bash
go test -cover ./...
```

## Architecture

The network package is designed with modularity and extensibility in mind:

1. **Interface-based Design**: Core functionality is exposed through interfaces, allowing for alternative implementations
2. **Pluggable Components**: Different network backends can be swapped (currently libp2p)
3. **Separation of Concerns**: Configuration, discovery, messaging, and virtual networking are independent modules
4. **Background Processing**: Non-blocking operations are handled by the background task scheduler
5. **Resource Management**: Configurable limits for memory, file descriptors, and connections

## License

Apache License 2.0

## Related Projects

- [**Actor**](https://github.com/depinkit/actor) - Secure actor programming framework that uses this network package
- [**DMS**](https://gitlab.com/nunet/device-management-service) - Device Management Service that this package was extracted from
- [**Crypto**](https://github.com/depinkit/crypto) - Cryptographic operations used by this package
- [**DID**](https://github.com/depinkit/did) - Decentralized identifier support

## Contributing

Contributions are welcome! This package is part of the Depinkit organization and follows the same contribution guidelines as other Depinkit projects.

## Support

For issues and questions:
- Open an issue on the repository
- Check the [DMS documentation](https://gitlab.com/nunet/device-management-service)
- Review libp2p documentation at https://github.com/libp2p/go-libp2p

---

**Part of the [Depinkit](https://github.com/depinkit) ecosystem for building decentralized applications.**
