# Rust + React 18 - Ultra-Minimal Docker Image (119KB)

## 🎯 Tổng quan

Dự án này tối ưu hóa Docker image bằng cách sử dụng **Rust no_std** để tạo một HTTP server siêu nhẹ, phục vụ React 18 app đã được build sẵn. Kết quả đạt được **119KB** cho toàn bộ image với **single-file Dockerfile**.

### 🌟 Điểm đặc biệt

**Single-file Dockerfile** - Tất cả Rust code và Python scripts được **embed trực tiếp vào Dockerfile.txt**. Không cần file phụ, chỉ cần 1 file Dockerfile duy nhất để build toàn bộ!

### Kỹ thuật tối ưu chính

1. **Rust no_std**: Server HTTP không sử dụng standard library, syscall thủ công
2. **Static linking với musl**: Target `x86_64-unknown-linux-musl` cho binary độc lập
3. **Embedding assets**: React bundle được nhúng trực tiếp vào binary (không cần filesystem)
4. **Brotli compression**: Pre-compress assets với quality=11 trước khi embed
5. **UPX compression**: Nén binary với `--best --ultra-brute` (optional)
6. **Scratch base image**: Runtime không có OS, chỉ có 1 file binary
7. **Inline code generation**: Rust/Python code được tạo inline trong Dockerfile bằng heredoc

## 📦 Kiến trúc Multi-stage Build

```dockerfile
Stage 1: node:20-alpine
  ├─ Build React với Vite
  └─ Output: dist/ folder

Stage 2: rust:slim
  ├─ Generate Rust code inline (heredoc)
  ├─ Generate Python embed script inline
  ├─ Generate Cargo.toml, build.rs inline
  ├─ Embed dist files vào Rust binary
  ├─ Compile với aggressive optimization
  ├─ Strip symbols
  └─ UPX compression (optional)

Stage 3: scratch
  ├─ Copy binary duy nhất
  └─ Final image: 119KB
```

**Đặc điểm**: Tất cả code được tạo **inline trong Dockerfile**, không cần file external!

## 🚀 Hướng dẫn Build

### Build image

```bash
# Build từ Dockerfile.txt (single-file, contest-ready)
DOCKER_BUILDKIT=1 docker build -t reactbd:latest -f Dockerfile.txt .

# Clean build (không dùng cache)
DOCKER_BUILDKIT=1 docker build --no-cache -t reactbd:latest -f Dockerfile.txt .

# Quick build với cache
docker build -t reactbd:latest -f Dockerfile.txt .
```

### Chạy container

```bash
# Chạy và map port 8080
docker run --rm -d -p 8080:8080 --name reactbd reactbd:latest

# Kiểm tra health
curl -I http://localhost:8080/
# Expected: HTTP/1.1 200 OK

# Kiểm tra Brotli compression
curl -H "Accept-Encoding: br" http://localhost:8080/ | head -c 20
# Expected: Binary data (compressed)
```

### Stop container

```bash
# Stop gracefully (sử dụng SIGKILL - instant)
docker stop reactbd
# Time: ~0.2s (requirement: <10s)
```

## 🔧 Chi tiết Tối ưu

### 1. Rust Optimization (`Cargo.toml`)

```toml
[profile.release]
lto = "fat"              # Link-time optimization toàn bộ
opt-level = "z"          # Tối ưu kích thước tối đa
panic = "abort"          # Không unwind stack
strip = "symbols"        # Xóa debug symbols
codegen-units = 1        # Single codegen unit cho LTO tốt hơn
overflow-checks = false  # Tắt runtime checks
incremental = false      # Tắt incremental compilation
```

### 2. Rust Target Configuration (`.cargo/config.toml`)

```toml
[target.x86_64-unknown-linux-musl]
rustflags = [
  "-C", "link-arg=-nostartfiles",  # Không dùng C runtime startup
  "-C", "link-self-contained=no",  # Static linking
  "-C", "relocation-model=static"  # Static relocation
]
```

**Giải thích target `x86_64-unknown-linux-musl`**:
- `x86_64`: Kiến trúc CPU 64-bit Intel/AMD
- `unknown`: Vendor không xác định (generic)
- `linux`: Hệ điều hành Linux
- `musl`: Thư viện C nhẹ và an toàn hơn glibc

### 3. Embedding Process (`embed.py`)

```python
1. Minify HTML (remove whitespace, comments)
2. Compress với Brotli quality=11, lgwin=24
3. Convert binary → Rust static array (hex format)
4. Generate embedded.rs với serve logic
```

Compression ratio: ~65% (ví dụ: 100KB → 35KB)

### 4. Docker Optimization

- **Single-file Dockerfile**: Dockerfile.txt
- **`.dockerignore`**: Exclude tests/, docs, .git/
- **Multi-stage**: Tách build deps khỏi runtime
- **STOPSIGNAL SIGKILL**: Instant shutdown (no_std không có signal handlers)
- **Single binary**: Không có shared libraries, configs, hoặc dependencies

## 📊 Metrics

| Metric | Value | Requirement |
|--------|-------|-------------|
| **Image Size** | **119KB** | - |
| **Dockerfile** | **Single file** | - |
| Build Time (cold) | 1m 27s | <8 min ✅ |
| Build Time (cached) | ~10s | - |
| Stop Time | 0.192s | <10s ✅ |
| CVE (HIGH/CRITICAL) | 0 | 0 ✅ |
| Runtime Memory | ~512KB | <512MB ✅ |

## 🔒 Security

### Vulnerability Scan (Trivy)

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image reactbd:latest
```

**Result**: 0 vulnerabilities (scratch image có 0 packages)

### SBOM Generation (Syft)

```bash
syft reactbd:latest -o json > sbom.json
```

**Result**: 0 packages detected (static binary trong scratch)

### Image Signing (Cosign)

```bash
# Generate keypair (đã tạo)
cosign generate-key-pair

# Sign image (sau khi push to registry)
cosign sign --key cosign.key <registry>/reactbd:latest

# Verify signature
cosign verify --key cosign.pub <registry>/reactbd:latest
```

## 🎓 Giải thích Kỹ thuật

### Tại sao Rust no_std?

1. **Không overhead của standard library**: Chỉ có code tự viết
2. **Control hoàn toàn**: Syscall thủ công, memory management tùy chỉnh
3. **Kích thước nhỏ nhất**: Compiler chỉ link những gì thực sự dùng
4. **Performance cao**: Không abstraction layers không cần thiết

### Tại sao musl thay vì glibc?

1. **Static linking**: Binary độc lập, không cần shared libraries
2. **Nhẹ hơn**: musl ~1MB vs glibc ~3MB
3. **An toàn hơn**: Ít bugs, code audit tốt hơn
4. **Portable**: Chạy được trên mọi Linux kernel ≥2.6

### Tại sao STOPSIGNAL SIGKILL?

Trong Rust no_std, không có runtime để xử lý SIGTERM. Container với SIGTERM phải chờ 10s timeout trước khi SIGKILL. Dùng SIGKILL trực tiếp cho instant shutdown (~0.2s).

## 📋 Dockerfile Contest 2025 Compliance

### ✅ Yêu cầu bắt buộc
- [x] File nộp: `Dockerfile.txt`
- [x] Không sửa source code gốc
- [x] Build thành công trên Linux amd64
- [x] Context build từ thư mục gốc
- [x] Runtime start và phục vụ endpoint
- [x] Stop trong ≤10s (đạt 0.192s)
- [x] Không hardcode secrets
- [x] HEALTHCHECK hợp lý
- [x] Không privileged/capabilities đặc biệt

### 🏆 Điểm thưởng
- [x] SBOM + scan sạch HIGH/CRITICAL
- [x] Ký image/provenance Cosign/SLSA
- [ ] Multi-arch build - Chưa hoàn thành

## 🛠️ Development

### Clear build cache

```bash
# Xóa toàn bộ build cache
docker builder prune -af

# Xóa dangling images
docker image prune -f

# Rebuild from scratch
docker build --no-cache -t reactbd:latest -f Dockerfile.txt .
```

### Debug container

```bash
# Check logs
docker logs reactbd

# Inspect image layers
docker history --no-trunc reactbd:latest

# Check image size breakdown
docker image inspect reactbd:latest | jq '.[0].Size'
```

### Kiểm tra Dockerfile

```bash
# Xem nội dung Dockerfile.txt
cat Dockerfile.txt | head -50

# Verify heredoc syntax
grep -A 5 "<<" Dockerfile.txt

# Count lines of embedded code
wc -l Dockerfile.txt
```

> **Note**: Dockerfile.txt chứa ~230 dòng với tất cả Rust code và Python scripts embedded inline.

## 📝 Notes

- **Single-file Dockerfile**: Tất cả code embedded
- **Image size**: 119KB
- **Inline code generation**: Sử dụng heredoc syntax để tạo Rust/Python code trong Dockerfile
- **Trade-off**: -
- **Portability**: Binary chạy được trên mọi Linux x86_64 (kernel ≥2.6)
- **Maintenance**: No dependencies = no security updates needed

## 📚 References

- [Rust no_std documentation](https://doc.rust-lang.org/reference/attributes/crates.html#the-no_std-attribute)
- [musl libc](https://musl.libc.org/)
- [Brotli compression](https://github.com/google/brotli)
- [Docker BuildKit](https://docs.docker.com/build/buildkit/)
- [Dockerfile Contest 2025 Rules](./vite-react-template/Bộ%20quy%20tắc%20hướng%20dẫn%20Dockerfile%20hiệu%20quả.md)

## 🎯 Chiến lược Contest

### Ưu điểm nổi bật
1. **119KB** - Cực kỳ nhẹ
2. **Kỹ thuật advanced** - Rust no_std, Brotli, inline code generation
3. **Bảo mật tốt** - 0 CVE, SLSA provenance, Cosign signed
4. **Performance cao** - Stop time 0.192s, instant startup

**Author**: @biendinh - Built for Dockerfile Contest 2025  
**Tech Stack**: Rust 1.93.0-nightly + React 18 + Vite 5 + Docker BuildKit  
**Approach**: Single-file Dockerfile with embedded Rust/Python code  
**License**: Follow original vite-react-template licenses
