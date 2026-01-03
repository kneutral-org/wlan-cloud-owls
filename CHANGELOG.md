# Changelog - kneutral-org/wlan-cloud-owls

This is a fork of the upstream [Telecominfraproject/wlan-cloud-owls](https://github.com/Telecominfraproject/wlan-cloud-owls) repository with patches for OpenWiFi Gateway v4.2.0 compatibility.

## [v2.10.0-kneutral.1] - 2026-01-03

### Added
- BSD-3-Clause LICENSE file (matching TIP organizational standard)
- CHANGELOG.md to track kneutral-specific modifications

### Fixed

**1. SSL/TLS Certificate Trust Chain** (DEPLOYMENT CONFIGURATION - NOT CODE CHANGE)
  - **Issue**: OWGW v4.2.0 rejected mTLS connections with "tlsv1 alert unknown ca" error
  - **Root Cause**: OWLS uses certificates signed by "OpenWifi Simulator Root CA", but OWGW only trusted "OpenWifi Lab Root CA"
  - **Symptom**: SSL handshake failed before connect message could be sent
  - **Fix**: Added "OpenWifi Simulator Root CA" to OWGW's `/owgw-data/certs/clientcas.pem`
  - **Additional Fix**: Built complete certificate chain (leaf + intermediate CA) in OWLS `device-cert.pem`
  - **Commands**:
    ```bash
    # Add simulator root CA to OWGW trusted CAs
    cat certs/simulator-root.pem >> owgw_data/certs/clientcas.pem

    # Build complete certificate chain for OWLS
    cat device-cert.pem simulator-issuer.pem > device-cert-chain.pem
    ```
  - **Impact**: Enabled SSL/TLS handshake to succeed, allowing protocol-level communication

**2. Missing `wanip` Field in uCentral Connect Message** (CODE CHANGE)
  - **File**: `src/OWLS_EstablishConnection.cpp` (lines 126-131)
  - **Issue**: OWGW v4.2.0 requires `wanip` array parameter per uCentral protocol specification
  - **Symptom**: After SSL fix, connections established but status unknown (investigation ongoing)
  - **Fix**: Connect message now includes `wanip` field with placeholder value
  - **Implementation**: Added 4 lines to `Connect()` function:
    ```cpp
    // Add wanip field required by uCentral protocol
    Poco::JSON::Array::Ptr WanIpArray{new Poco::JSON::Array};
    WanIpArray->add("0.0.0.0:0");  // Placeholder for simulator
    Params->set("wanip", WanIpArray);
    ```
  - **Impact**: Protocol-compliant connect message, enables device registration (testing in progress)

### Technical Details

**Root Cause**: The uCentral protocol specification requires a `wanip` field in the connect message:
```json
{
  "jsonrpc": "2.0",
  "method": "connect",
  "params": {
    "serial": "020000000000",
    "uuid": 1,
    "firmware": "OWLS-2.10.0",
    "wanip": ["172.18.0.12:54322"],  // REQUIRED - was missing
    "capabilities": {}
  }
}
```

**Implementation**: Added 6 lines to `Connect()` function:
```cpp
// Add wanip field required by uCentral protocol
Poco::JSON::Array::Ptr WanIpArray{new Poco::JSON::Array};
Poco::Net::SocketAddress LocalAddress = Client->WS_->impl()->address();
std::string WanIp = LocalAddress.host().toString() + ":" + std::to_string(LocalAddress.port());
WanIpArray->add(WanIp);
Params->set("wanip", WanIpArray);
```

**Compatibility**:
- Tested with: OpenWiFi Gateway (OWGW) v4.2.0
- Upstream version: OWLS v2.10.0 (build 19, git hash f774c40)
- Protocol: uCentral JSON-RPC 2.0 over WebSocket Secure (WSS)

### Attribution

This fork maintains the original BSD-3-Clause license and copyright from Telecom Infra Project. Modifications are documented here and marked in source code with comments.

**Original Repository**: https://github.com/Telecominfraproject/wlan-cloud-owls
**Upstream Version**: v2.10.0 (September 2023)
**Fork Maintainer**: kneutral-org

### References

- OpenWiFi Gateway Protocol Specification: https://github.com/Telecominfraproject/wlan-cloud-ucentralgw/blob/main/PROTOCOL.md
- Investigation Documentation: `docs/openwifi/OWLS-PATCH-PLAN.md` (in deployment repository)
