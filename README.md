# SCP: Code Share Protocol

[![Version: 0.1.0](https://img.shields.io/badge/Version-0.1.0-blue.svg)](https://github.com/yourusername/scp-protocol/releases)

## Overview

SCP (Code Share Protocol) enables dynamic code sharing and integration at runtime without compilation to files or application restarts. It addresses the lack of stable ABI in many languages by providing a self-describing binary format with metadata about functions, types, and dependencies.

### Key Features

- **Dynamic Loading**: Load code into memory on-the-fly with relocation offsets
- **Type Safety**: Describe types and endpoints to minimize undefined behavior
- **Cross-Platform**: Binary format supporting multiple architectures (x86, ARM)
- **Hot Reloading**: Update code without downtime
- **Extensible**: Versioned format for future enhancements

### Use Cases

| Scenario | Description |
|----------|-------------|
| **Hot Code Reloading** | Games and servers without downtime |
| **Plugin Systems** | Embedded systems and applications |
| **Microservices** | Dynamic updates within single process |
| **JIT Integration** | Experiments with LLVM-based compilation |

## Architecture

SCP consists of two components:

1. **Header**: Binary metadata block (fixed format for parsing)
2. **Code Blob**: Machine-dependent executable code following the header

### Workflow

```
Generator → [Header + Code Blob] → Loader → Runtime Execution
```

- **Generator**: Creates header + code blob (compiler/JIT output)
- **Loader**: Parses header, allocates memory, relocates offsets, registers functions  
- **Runtime**: Function invocation through pointers with optional type checking

## Binary Format Specification

### Header Structure

The header uses little-endian byte order and consists of fixed fields followed by variable-length tables.

#### Fixed Fields (32 bytes)

| Offset | Field | Type | Size | Description |
|--------|-------|------|------|-------------|
| 0 | `magic_number` | uint32 | 4 | Magic identifier: `0x53435000` ('SCP\0') |
| 4 | `version_major` | uint16 | 2 | Major version (0 for v0.x) |
| 6 | `version_minor` | uint16 | 2 | Minor version (1 for v0.1) |
| 8 | `header_size` | uint32 | 4 | Total header size (excluding code blob) |
| 12 | `code_blob_size` | uint64 | 8 | Size of following code blob |
| 20 | `architecture` | uint8 | 1 | 0=x86_64, 1=ARM64, 2=x86_32 |
| 21 | `calling_convention` | uint8 | 1 | 0=cdecl, 1=stdcall, 2=system |
| 22 | `flags` | uint16 | 2 | bit0=thread-safe, bit1=requires_gc |
| 24 | `checksum` | uint32 | 4 | CRC32 of header + code blob |
| 28 | `reserved` | bytes | 4 | Reserved (zero-filled) |

#### Variable Tables

Each table starts with `count` (uint32) followed by `count` entries.

##### 1. Functions Table

| Field | Type | Size | Description |
|-------|------|------|-------------|
| `name_offset` | uint32 | 4 | Offset to function name in strings table |
| `entry_offset` | uint64 | 8 | Entry point offset in code blob |
| `param_count` | uint8 | 1 | Number of parameters |
| `params` | uint8[] | variable | Parameter type codes |
| `return_type` | uint8 | 1 | Return type code |
| `flags` | uint8 | 1 | bit0=pure function |

**Type Codes**: 0=void, 1=int8, 2=int16, 3=int32, 4=int64, 5=float32, 6=float64, 10=pointer, 11=struct_ref

##### 2. Types Table (Custom Structures)

| Field | Type | Size | Description |
|-------|------|------|-------------|
| `id` | uint16 | 2 | Unique type ID |
| `name_offset` | uint32 | 4 | Offset to type name |
| `size` | uint32 | 4 | Size in bytes |
| `field_count` | uint8 | 1 | Number of fields |
| `fields` | struct[] | variable | Per field: {offset(uint32), type_code(uint8)} |

##### 3. Dependencies Table

| Field | Type | Size | Description |
|-------|------|------|-------------|
| `name_offset` | uint32 | 4 | Offset to dependency name |
| `required_version` | uint32 | 4 | Minimum version (0 if none) |

##### 4. Strings Table

Null-terminated UTF-8 strings referenced by offsets from other tables.

## Usage Example

### Loading SCP Code (Pseudo-C)

```c
void* load_scp(const void* data) {
    SCP_Header* hdr = (SCP_Header*)data;
    
    // Validate magic number
    if (hdr->magic_number != 0x53435000) return NULL;
    
    // Allocate executable memory
    void* mem = mmap(NULL, hdr->code_blob_size, PROT_EXEC | PROT_READ | PROT_WRITE);
    
    // Copy code blob
    memcpy(mem, (char*)data + hdr->header_size, hdr->code_blob_size);
    
    // Parse functions and relocate entry points
    // entry_point = mem + function.entry_offset
    
    return mem;
}
```

### Binary Header Example

```hex
53 43 50 00  // magic: 'SCP\0'
00 00 01 00  // version: 0.1
80 00 00 00  // header_size: 128 bytes
00 10 00 00  // code_blob_size: 4096 bytes
...
```

## Implementation Guidelines

### Generator (Code Producer)
- Integrate with compilers (e.g., clang plugin) for automatic header generation
- Ensure proper symbol relocation and offset calculation
- Validate type consistency across function boundaries

### Loader (Code Consumer)
- Implement runtime type checking for debug builds
- Use memory protection (DEP/NX) for security
- Handle architecture-specific calling conventions

### Security Considerations

⚠️ **Warning**: Not designed for untrusted code. Recommendations:
- Use digital signatures (e.g., Ed25519) for trusted sources
- Implement sandboxing for unknown code
- Validate all offsets and bounds before memory access

## Development Tools

| Tool | Purpose |
|------|---------|
| `scp-gen` | Generate SCP files from object files |
| `scp-validate` | Verify SCP format and checksums |
| `scp-dump` | Human-readable header inspection |

## Roadmap

- [ ] Reference implementation in C/Rust
- [ ] WebAssembly integration
- [ ] Multi-language bindings (Python, JavaScript)
- [ ] Garbage collector hooks
- [ ] Debugging symbol support

## Contributing

1. Fork the repository
2. Create a feature branch
3. Submit a pull request with tests
4. Ensure >80% code coverage for core functionality

## License

MIT License. See [LICENSE](LICENSE) for details.

---

**Status**: Proof of Concept - Use at your own risk in production environments.
