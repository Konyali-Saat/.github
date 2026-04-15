# Konyali-Saat — Central Reusable Workflows

Bu repo, `Konyali-Saat` GitHub organizasyonundaki tum repolar icin merkezi olarak
bakimi yapilan **reusable GitHub Actions workflow**'larini barindirir. Her repo
ayni standart CI/CD hattini burada tanimli workflow'lari `workflow_call` ile
cagirarak kullanir.

> WIF (Workload Identity Federation) ve GCP servis hesabi bilgileri workflow
> tanimlarinda sabittir — cagiran reponun ek secret tanimlamasina gerek yoktur
> (PR-Agent icin `OPENROUTER_API_KEY` haric).

## Workflow Listesi

| Dosya | Amac |
|-------|------|
| [`.github/workflows/ci-node.yml`](.github/workflows/ci-node.yml) | Node.js / TypeScript CI (lint, type-check, test, build) |
| [`.github/workflows/ci-python.yml`](.github/workflows/ci-python.yml) | Python CI (lint, test) |
| [`.github/workflows/deploy-cloudrun.yml`](.github/workflows/deploy-cloudrun.yml) | Cloud Run deploy (WIF auth + Artifact Registry) |
| [`.github/workflows/pr-agent.yml`](.github/workflows/pr-agent.yml) | PR-Agent otomatik PR review (OpenRouter uzerinden) |

---

## 1. `ci-node.yml` — Node.js / TypeScript CI

4 paralel job calistirir: `lint`, `type-check`, `test`, `build`. Her job npm
cache kullanir. `is-nextjs: true` verilirse build job'unda `.next/cache`
icin ek cache katmani devreye girer.

### Inputs

| Input | Tip | Default | Aciklama |
|-------|-----|---------|----------|
| `node-version` | string | `"22"` | Kullanilacak Node.js surumu |
| `run-lint` | boolean | `true` | Lint job'u calistir |
| `run-type-check` | boolean | `true` | `tsc --noEmit` ile tip kontrolu |
| `run-tests` | boolean | `true` | Test job'u calistir |
| `run-build` | boolean | `true` | Build job'u calistir |
| `is-nextjs` | boolean | `false` | `.next/cache` cache katmanini etkinlestir |
| `lint-command` | string | `npx eslint . --format=stylish` | Ozel lint komutu |
| `test-command` | string | `npm test -- --verbose --coverage` | Ozel test komutu |
| `build-command` | string | `npm run build` | Ozel build komutu |

### Ornek kullanim

```yaml
# Cagiran reponun .github/workflows/ci.yml dosyasi
name: CI
on:
  pull_request:
  push:
    branches: [main]

jobs:
  ci:
    uses: Konyali-Saat/.github/.github/workflows/ci-node.yml@main
    with:
      node-version: "22"
      is-nextjs: true
      lint-command: "npm run lint"
```

---

## 2. `ci-python.yml` — Python CI

2 paralel job: `lint`, `test`. pip cache kullanir.

### Inputs

| Input | Tip | Default | Aciklama |
|-------|-----|---------|----------|
| `python-version` | string | `"3.12"` | Kullanilacak Python surumu |
| `run-lint` | boolean | `true` | Lint job'u calistir |
| `run-tests` | boolean | `true` | Test job'u calistir |
| `lint-command` | string | `ruff check . && ruff format --check .` | Ozel lint komutu |
| `test-command` | string | `pytest -v` | Ozel test komutu |
| `requirements-file` | string | `requirements.txt` | Bagimlilik dosyasi yolu |

### Ornek kullanim

```yaml
jobs:
  ci:
    uses: Konyali-Saat/.github/.github/workflows/ci-python.yml@main
    with:
      python-version: "3.12"
      requirements-file: "requirements-dev.txt"
```

---

## 3. `deploy-cloudrun.yml` — Cloud Run Deploy

WIF ile `konyali-saat-etl` projesine authenticate olur, imaji
`europe-west1-docker.pkg.dev/konyali-saat-etl/images/` Artifact Registry'ye push eder,
ardindan Cloud Run'a deploy eder. Image tag olarak `${{ github.sha }}` kullanilir.
Deploy sonrasi rollback komutu log'da gosterilir.

### Sabit WIF degerleri (workflow icinde)

- Provider: `projects/518226731997/locations/global/workloadIdentityPools/github-pool/providers/github-provider-new`
- Service Account: `github-actions-sa@konyali-saat-etl.iam.gserviceaccount.com`

### Inputs

| Input | Tip | Default | Aciklama |
|-------|-----|---------|----------|
| `service-name` | string | **required** | Cloud Run servis adi |
| `region` | string | `europe-west1` | GCP region |
| `dockerfile` | string | `Dockerfile` | Dockerfile yolu |
| `environment` | string | `production` | GitHub environment etiketi |

### Ornek kullanim

```yaml
# Cagiran reponun .github/workflows/deploy.yml dosyasi
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    uses: Konyali-Saat/.github/.github/workflows/deploy-cloudrun.yml@main
    with:
      service-name: "urun-siparis-api"
      region: "europe-west1"
    permissions:
      contents: read
      id-token: write
```

> **Not:** Cagiran workflow `permissions: id-token: write` tanimlamalidir
> (WIF token uretimi icin gerekli).

---

## 4. `pr-agent.yml` — PR-Agent Review

[Codium-ai/pr-agent](https://github.com/Codium-ai/pr-agent) kullanarak acilan
PR'lara otomatik aciklama, review ve kod onerisi ekler. Model erisimi
OpenRouter uzerinden saglanir.

### Secrets

| Secret | Aciklama |
|--------|----------|
| `OPENROUTER_API_KEY` | OpenRouter API anahtari (PR-Agent icin) |

### Ornek kullanim

```yaml
# Cagiran reponun .github/workflows/pr-review.yml dosyasi
name: PR Review
on:
  pull_request:
    types: [opened, reopened, synchronize]

jobs:
  review:
    uses: Konyali-Saat/.github/.github/workflows/pr-agent.yml@main
    secrets:
      OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
    permissions:
      contents: read
      pull-requests: write
      issues: write
```

---

## Surum Yonetimi

Workflow'larda breaking change yapildiginda:

1. Yeni surum icin branch / tag olustur (orn. `v2`).
2. Cagiran repolari asamali olarak yeni ref'e al (`@v2`).
3. `@main` referansini kullananlar icin CHANGELOG'da degisikligi duyur.

`@main` referansi her zaman **en guncel** hali isaret eder — stabilite gerektiren
repolar sabit commit SHA veya tag referansi kullanmalidir.

---

## Ilgili Dokumanlar

- Global kurallar: `~/.claude/CLAUDE.md`
- GCP Project: `konyali-saat-etl` (Project Number: `518226731997`)
- Container Registry: `europe-west1-docker.pkg.dev/konyali-saat-etl/images/`
