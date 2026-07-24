# ADR-009: Cloudflare R2 as Primary Storage Provider

**Status:** Accepted  
**Date:** 2025-07-23  
**Deciders:** Architecture Team

---

## Context

GLU stores binary assets: book cover images, audio files (text-to-speech chapters), user avatars, uploaded documents (PDFs, EPUBs), and import source files. These files must be stored durably, served quickly via CDN, and cost-effectively as the library grows to hundreds of thousands of books.

---

## Decision

**Use Cloudflare R2 as the primary file storage provider.** Storage is abstracted behind `IStorageProvider` to allow switching providers without changing business logic.

---

## Alternatives Considered

| Solution | Description |
|---|---|
| **Cloudflare R2** | S3-compatible, zero egress fees, CDN via Cloudflare |
| **Amazon S3** | Industry standard, egress fees, CloudFront for CDN |
| **MinIO** | Self-hosted S3-compatible, full control, ops overhead |
| **Azure Blob Storage** | Microsoft ecosystem, CDN via Azure CDN |
| **Google Cloud Storage** | GCP ecosystem, CDN via Cloud CDN |
| **Backblaze B2** | Low cost, S3-compatible, less mature |
| **DigitalOcean Spaces** | S3-compatible, simpler pricing, limited regions |

---

## Reasons for Cloudflare R2

### 1. Zero egress fees — the most important differentiator

Amazon S3 charges $0.09/GB for data transferred out to the internet. At scale, this is the dominant infrastructure cost for media-heavy platforms. A platform serving 1TB of book covers and audio per month pays ~$90/month in S3 egress fees — before any other AWS cost.

**R2 charges zero egress fees.** Data transfer from R2 to Cloudflare's network (and thus to users via CDN) is free. Storage is $0.015/GB/month, competitive with S3's $0.023/GB/month.

For GLU, a media-heavy knowledge platform with book covers, audio, and documents, this is a meaningful long-term cost reduction.

### 2. Native Cloudflare CDN integration

R2 objects are served via Cloudflare's global CDN (200+ PoPs) by enabling a public bucket domain or custom domain. There is no separate CDN configuration required. Files in R2 automatically benefit from Cloudflare's edge cache.

Contrast with S3 + CloudFront: S3 stores the file, CloudFront is a separate service with separate configuration, separate billing, and egress charges from S3 to CloudFront.

### 3. S3-compatible API

R2 implements the S3 API. The AWS SDK (`@aws-sdk/client-s3`) works with R2 by changing the endpoint URL. Migration from R2 to S3, or from S3 to R2, requires changing one config value. No code changes.

### 4. Cloudflare Images for on-the-fly transforms

Cloudflare Images integrates with R2 to resize and optimize images at the edge. Book cover thumbnails (200x300, 400x600, 800x1200) are generated on-the-fly without a Lambda function or Sharp processing step. This eliminates the image-processing queue for covers entirely.

### 5. Worker integration (future)

Cloudflare Workers can intercept requests to R2, add authentication checks, transform content, or log access. This enables per-user access control on private files without a backend round-trip.

---

## Storage Abstraction

The `IStorageProvider` interface makes R2 one of several interchangeable implementations:

```typescript
interface IStorageProvider {
  upload(key: string, file: Buffer, options: UploadOptions): Promise<StorageFile>;
  download(key: string): Promise<Buffer>;
  delete(key: string): Promise<void>;
  getSignedUrl(key: string, expiresIn: number): Promise<string>;
  getPublicUrl(key: string): string;
  exists(key: string): Promise<boolean>;
}
```

`StorageService` delegates to the provider configured in `STORAGE_PROVIDER` env var. Switching from R2 to S3 = change one env var. Zero business code changes.

**Local development** uses MinIO (a self-hosted S3-compatible server). Developers do not need a Cloudflare account to run GLU locally.

---

## Object Key Structure

```
covers/{bookId}/original.{ext}
covers/{bookId}/sm.webp            (200x300)
covers/{bookId}/md.webp            (400x600)
covers/{bookId}/lg.webp            (800x1200)

audio/{bookId}/{chapterId}.mp3
audio/{bookId}/{chapterId}.aac

documents/{userId}/{importJobId}.{ext}   (uploaded PDFs, EPUBs)

avatars/{userId}/original.{ext}
avatars/{userId}/md.webp           (200x200)

exports/{userId}/{exportJobId}.zip

tmp/{userId}/{uploadId}.{ext}      (short-lived upload staging, 1h TTL)
```

---

## Pros

- Zero egress fees (vs $0.09/GB for S3)
- Native Cloudflare CDN (zero additional config)
- S3-compatible API (familiar tooling, easy migration)
- Cloudflare Images for on-the-fly image transforms
- Global durability (Cloudflare infrastructure)
- Simpler billing than AWS (one vendor for CDN + storage)

## Cons

- Cloudflare vendor lock-in for edge features (Workers, Images)
- Fewer regions than AWS S3 (but Cloudflare CDN compensates)
- Less mature managed database/compute options (GLU uses non-Cloudflare DB/compute)
- Multi-cloud strategy requires the abstraction layer (which we have)

---

## Consequences

- `StorageModule` wraps all file operations via `IStorageProvider`
- Local dev uses MinIO (port 9000, console port 9001)
- Staging and production use Cloudflare R2
- All public asset URLs are CDN URLs (`cdn.glu.app/...`), never raw R2 URLs
- Signed URLs used for private content (user documents, draft books before publication)
- `R2Provider` implements `IStorageProvider` using `@aws-sdk/client-s3` with R2 endpoint
- `MinIOProvider` implements `IStorageProvider` for local dev (same SDK, different endpoint)
