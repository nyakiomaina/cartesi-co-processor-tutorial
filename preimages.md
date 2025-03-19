# Preimages in Cartesi

Preimages are a fundamental concept in Cartesi's architecture, particularly when dealing with co-processors and input processing. This document explores how preimages work in practice within the Cartesi ecosystem.

## What is a Preimage?

In the context of Cartesi, a preimage refers to the original input data that, when processed, produces a specific output or hash. When running computations inside a Cartesi Machine, you might need to access data that corresponds to a particular hash (either SHA256 or Keccak256).

## Requesting Preimages Inside a Cartesi Machine

The Cartesi Machine provides a powerful mechanism for requesting preimages through the Generic I/O (GIO) framework. This allows your application to request and access preimage data during computation.

### Technical Implementation

The GIO framework uses the following domains and endpoints for preimage operations:

```rust
// GIO domains
const GET_IMAGE_GIO: u32 = 0x2a;  // For retrieving preimages
const PUT_KECCAK256_GIO: u32 = 0x2c;  // For submitting Keccak256 preimages
const PUT_SHA256_GIO: u32 = 0x2d;  // For submitting SHA256 preimages
```

When requesting a preimage, you need to provide:
- Hash Type (1 byte)
  - `1`: SHA256
  - `2`: Keccak256
- Hash (variable length bytes)

Example of requesting a preimage:
```python
def get_preimage(hash_type: int, hash_value: bytes) -> None:
    gio_payload = {
        "domain": 0x2a,  # GET_IMAGE_GIO
        "id": "0x" + (bytes([hash_type]) + hash_value).hex()
    }
    response = requests.post(rollup_server + "/gio", json=gio_payload)
    if response.status_code == 202:
        logger.info(f"GetImage request sent successfully")
    else:
        logger.error(f"Failed to request preimage. Status code: {response.status_code}")
```

When submitting a preimage, you need to provide:
- Reason (2 bytes, uint16)
- Input data (max 256KB)

Example of submitting preimages:
```python
def put_image_keccak256(reason: int, data: bytes) -> bytes:
    if len(data) > 256 * 1024:  # 256KB limit
        raise ValueError("Input data exceeds 256KB limit")

    k = sha3.keccak_256()
    k.update(data)

    gio_payload = {
        "domain": 0x2c,  # PUT_KECCAK256_GIO
        "id": "0x" + data.hex()
    }
    response = requests.post(rollup_server + "/gio", json=gio_payload)
    if response.status_code == 202:
        logger.info(f"PutImageKeccak256 emitted successfully")
        return k.digest()
    else:
        raise RuntimeError(f"Failed to submit preimage. Status code: {response.status_code}")

def put_image_sha256(reason: int, data: bytes) -> bytes:
    if len(data) > 256 * 1024:  # 256KB limit
        raise ValueError("Input data exceeds 256KB limit")

    h = hashlib.sha256()
    h.update(data)

    gio_payload = {
        "domain": 0x2d,  # PUT_SHA256_GIO
        "id": "0x" + data.hex()
    }
    response = requests.post(rollup_server + "/gio", json=gio_payload)
    if response.status_code == 202:
        logger.info(f"PutImageSHA256 emitted successfully")
        return h.digest()
    else:
        raise RuntimeError(f"Failed to submit preimage. Status code: {response.status_code}")
```

Important notes:
- All preimage operations use the same `/gio` endpoint with different domains
- Getting preimages uses domain `0x2a`
- Submitting Keccak256 preimages uses domain `0x2c`
- Submitting SHA256 preimages uses domain `0x2d`
- A successful operation returns status code 202
- If a requested preimage is not found, the computation will be terminated
- Input data size is limited to 256KB (262,144 bytes)

## Operator Requirements

For preimages to be available during a Coprocessor execution:

1. The preimages must be uploaded to the operator(s) running the execution
2. Operators need to have access to the data before it can be served to the Cartesi Machine
3. The data must be available before the computation starts

### Operator API Endpoints

Operators provide several endpoints for managing preimages:

1. `GET /get_preimage/<hash_type>/<hash>`
   - Direct HTTP GET endpoint to retrieve preimage data
   - `hash_type`: 1 for SHA256, 2 for Keccak256
   - `hash`: The hash value without '0x' prefix
   - Returns the raw preimage data
   - **Important Security Notes:**
     - Operators and solvers will charge for this service
     - It's recommended to front this with a CDN for production use
     - Clients MUST verify the hash of received data matches the expected hash
     - Example: `/get_preimage/2/1234abcd...`

2. `/upload_preimages`
   - Accepts a CBOR-encoded array of tuples: `Vec<(u8, Vec<u8>, Vec<u8>)>`
   - Each tuple contains:
     - Hash type (u8)
     - Hash (bytes)
     - Preimage data (bytes)
   - Data size must not exceed 256KB

3. `/check_preimages_status`
   - Accepts a CBOR-encoded array of pairs: `Vec<(u8, Vec<u8>)>`
   - Each pair contains:
     - Hash type (u8)
     - Hash (bytes)
   - Returns availability status for each requested preimage

### Storage Implementation

Preimages need to be stored persistently by operators. The specific storage implementation may vary by operator, but all implementations must:
- Maintain data integrity
- Support quick retrieval by hash
- Handle concurrent access
- Implement appropriate cleanup policies
- Enforce size limits

### CDN and Data Distribution

When working with preimages in a production environment:

1. Use a CDN to cache and distribute preimage data
2. Implement client-side hash verification
3. Consider rate limiting and access controls
4. Monitor CDN costs and usage patterns

## Preimage Hints

Cartesi provides a "hinting" mechanism for certain common types of preimages:

### Ethereum Block Hashes
- You can indicate that your application might need access to specific block hashes
- This allows operators to prepare the necessary data beforehand
- Uses Keccak256 (hash type 2)

### Code Hashes
- Similar to block hashes, you can hint that you'll need access to specific contract code
- This is particularly useful when interacting with smart contracts
- Uses Keccak256 (hash type 2)

## Storage Costs and Economics

When working with preimages in production, be aware of the associated costs:

### Storage Rent
- Preimages stored by operators may incur ongoing storage rent fees
- Longer storage duration typically means higher cumulative costs
- Consider implementing time-based cleanup strategies for rarely accessed preimages

### Retrieval Costs
- Operators and solvers charge for preimage retrieval operations
- Costs may vary based on:
  - Data size
  - Retrieval frequency
  - Network conditions
  - Operator-specific pricing models

### Cost Optimization Strategies
1. Use CDN caching to minimize direct operator retrievals
2. Implement TTL (Time-To-Live) for stored preimages
3. Batch related preimages when possible
4. Consider local caching for frequently accessed data
5. Monitor and analyze usage patterns to optimize storage duration

## Best Practices

When working with preimages in your Cartesi applications:

1. Plan ahead which preimages your application will need
2. Use the hinting mechanism when possible for common data types
3. Ensure all necessary preimages are available to operators
4. Implement proper error handling for cases where preimages might not be available
5. Document your preimage requirements clearly
6. Check preimage availability before starting computation
7. Be aware that missing preimages will terminate the computation
8. Keep preimage sizes under the 256KB limit
9. Consider implementing cleanup strategies for old preimages

## Further Reading

- [Cartesi Documentation](https://docs.cartesi.io)
- [Technical Specifications](https://docs.cartesi.io/machine/host/cmdline/)
- [GIO Framework Documentation](https://docs.cartesi.io)