# Wardrobe

A local-first digital wardrobe for clothes you own. Upload a photo, confirm its
metadata, browse your closet, compose an outfit on a faceless mannequin, and save
looks to an editorial board.

The default app runs in your browser with local storage, deterministic demo
suggestions, and a **2.5D layered styling preview**. It needs no account, cloud
credentials, or backend. Optional vision/product-reference services and the local
Proxy 3D Lab are separate opt-in paths.

**[Live demo](https://edwardh-jedi.github.io/wardrobe/)** ·
[Architecture](docs/ARCHITECTURE.md) · [Feature scope](docs/PROJECT_SCOPE.md) ·
[CI](https://github.com/EdwardH-jedi/wardrobe/actions/workflows/ci.yml) ·
[Manual QA](docs/QA_CHECKLIST.md)

## Quick start

Use Node.js 22 and the committed npm lockfile:

```bash
git clone https://github.com/EdwardH-jedi/wardrobe.git
cd wardrobe
npm ci
npm run dev
# Open http://localhost:5173
```

Click **Load sample** to explore with demo garments, or **Upload** a photo,
confirm the suggested details, style an outfit, and save it to the board.
No environment configuration is needed for this local flow.

## Features and current scope

| Mode | Implemented behavior | Requirements and limits |
|---|---|---|
| Local archive, default | Clothing photos, editable purchase metadata, closet browsing, outfit composition, saved looks | Browser persistence; metadata and reference suggestions are deterministic mocks |
| Local image preparation | Manual crop and opt-in background cutout | Edge-seeded flood fill, not ML segmentation; difficult backgrounds can fail |
| Optional vision | Sends a thumbnail for draft category, color, and tags | Serverless deployment, provider credentials, explicit configuration and upload consent; user confirms the draft; failures fall back to a labeled mock |
| Optional product references | Product-page metadata lookup and optional eBay candidate search | Configured serverless routes; search needs separate credentials and opt-in; candidates require user approval |
| Proxy 3D Lab | Front/back PNGs to a textured GLB preview, with cutout and alignment controls | Local FastAPI service and lazy-loaded Three.js; extruded silhouette or flat plane |

**Proxy 3D is not virtual try-on, cloth simulation, garment reconstruction, or
accurate body fitting.** Provider code is present, but local checks do not establish
live provider availability or recognition accuracy.

## Architecture and storage

React 18 + TypeScript + Vite handle the UI. A pure domain reducer manages archive
state; the provider layer performs storage and network I/O. CSS design tokens
support the studio, closet, and saved-look views. The experimental 3D viewer is
loaded separately from normal closet startup.

The core flow is:

```text
photo → local image preparation → draft details → user confirmation
      → closet → outfit composition → saved look
```

Metadata uses IndexedDB → localStorage → memory fallback. Memory storage is
non-persistent. Cropped/cutout image blobs use a separate IndexedDB store when
available, with downscaled thumbnails as a fallback. A conservative orphan sweep
protects recently written and referenced blobs.

Full-resolution archival, atomic metadata/blob writes, and complete multi-tab
coordination remain limitations. Browser storage is not a cloud backup.

## Optional services

### Vision and product references

The TypeScript handlers in `api/` are separate from the Vite dev server. See
[.env.example](.env.example) for configuration:

- `VITE_API_BASE` points to the separately served or deployed serverless handlers
- `VITE_ANALYZER=vision` also opts into thumbnail analysis; the upload flow asks
  for consent before sending the image
- `VITE_CANDIDATES=search` also opts into live shopping candidates
- `ANTHROPIC_API_KEY` and optional `ANALYZE_MODEL` configure vision on the server
- `EBAY_CLIENT_ID` and `EBAY_CLIENT_SECRET` configure eBay search on the server

Keep provider credentials server-side, without a `VITE_` prefix, and never commit
them. Provider usage can incur charges. Leaving the frontend opt-ins unset keeps
the local mock experience. An unavailable vision service falls back to a mock
with its provenance preserved.

`npm run dev` does **not** execute the serverless handlers. Product-reference and
vision requests need those routes served separately or deployed; the FastAPI
service below handles proxy 3D, not these routes.

### Local Proxy 3D Lab

In a second terminal, use Python 3.12, matching the backend CI environment:

```bash
cd backend
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python -m uvicorn app.main:app --port 8000
```

Vite forwards `/api` requests to `http://127.0.0.1:8000` for the local lab. The
backend uses FastAPI, Pillow, trimesh, and pygltflib. Windows setup, PNG input
requirements, GLB endpoints, and result storage are documented in
[backend/README.md](backend/README.md).

## Development and checks

```bash
npm run typecheck
npm run lint
npm test
npm run build
# Optional interactive test runner:
npm run test:watch
# From backend/ with its virtual environment active:
python -m pytest -q
```

CI is configured to run the frontend checks and a separate Python backend test
job without production credentials. Frontend unit/component tests cover `src/`;
`api/` handlers are also typechecked. Live vision, shopping, and deployed
serverless behavior need separate integration verification. Check the workflow
run for the exact commit under review.

## Repository map

| Path | Purpose |
|---|---|
| `src/domain/` | Garment/outfit types, pure rules, layout, and provenance |
| `src/app/providers/` | Reducer, archive state, persistence, and I/O coordination |
| `src/components/` | Closet, outfit, studio, UI, and optional avatar lab views |
| `src/lib/` | Storage adapters, image preparation, analyzer and reference integrations |
| `src/styles/` | Shared design tokens and app styles |
| `api/` | Optional serverless vision and product-reference handlers |
| `backend/` | Local FastAPI proxy-3D experiments and Python tests |
| `docs/` | Architecture, scope, image pipeline, avatar track, and QA notes |

## Further reading

- [Image pipeline](docs/AI_IMAGE_PIPELINE.md)
- [Avatar track and limitations](docs/AVATAR_TRACK.md)
- [Implementation plan](docs/planning/PLAN.md)
- [Manual QA checklist](docs/QA_CHECKLIST.md)

The project contains experimental and historical phase notes. Treat future items
in those plans as roadmap work, and use the implementation and checks when
assessing a feature's current behavior.
