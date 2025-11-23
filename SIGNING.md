# Image Signing & Provenance

## Cosign Keys

Cosign keypair đã được tạo:
- Private key: `cosign.key`
- Public key: `cosign.pub`

## SLSA Provenance

SLSA provenance document: `slsa-provenance.json`

### Image Details
- **Image**: reactbd:latest
- **Digest**: sha256:9b5a84dc865e974d258c514e0517a838ab746c112ade87dda1049dfbde35b5ae
- **Size**: 119KB (embedded single-file Dockerfile)
- **Base**: scratch (empty image)

### Build Materials
1. **node:20-alpine** - Vite build stage
2. **rust:slim** - Rust compilation stage  
3. **scratch** - Runtime base

## Verification (sau khi push lên registry)

```bash
# Push image to registry
docker tag reactbd:latest <registry>/reactbd:latest
docker push <registry>/reactbd:latest

# Sign with Cosign
cosign sign --key cosign.key <registry>/reactbd:latest

# Verify signature
cosign verify --key cosign.pub <registry>/reactbd:latest

# Attach SLSA provenance
cosign attach attestation --type slsaprovenance --attestation slsa-provenance.json <registry>/reactbd:latest
```

## Security Features

✅ **Cosign keypair generated** - Ready for signing when pushed to registry  
✅ **SLSA provenance** - Build reproducibility & supply chain security  
✅ **Trivy scan clean** - No HIGH/CRITICAL vulnerabilities  
✅ **SBOM available** - Software bill of materials documented  
✅ **Minimal attack surface** - Scratch base with single static binary

---
**Note**: Image signing với Cosign yêu cầu push lên container registry (Docker Hub, ghcr.io, etc.)
Keypair và provenance đã sẵn sàng để sử dụng khi deploy.
