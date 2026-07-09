# all2pod

Self-hosted app that turns your [TorBox](https://torbox.app) content into **podcast RSS feeds** — video or audio — that you can subscribe to in any podcast app (Apple Podcasts, Overcast, Pocket Casts, …).

Create feeds, add movies/shows/audiobooks to them, and all2pod publishes a per-feed RSS feed. Video is transcoded to an Apple-compatible MP4; streams are fetched fresh from TorBox at play time, so links never expire.

> This repository contains only the deployment files. all2pod ships as a prebuilt image — `ghcr.io/jdmry/all2pod` — and you run it with your own TorBox account.

## Quick start (Docker Compose)

```bash
git clone https://github.com/jdmry/all2pod.git
cd all2pod
cp .env.example .env
# edit .env — set TORBOX_API_KEY, ADMIN_USER, ADMIN_PASSWORD, PUBLIC_URL
docker compose up -d
```

Then open your `PUBLIC_URL` and log in with `ADMIN_USER` / `ADMIN_PASSWORD`.

To update:

```bash
docker compose pull && docker compose up -d
```

## Plain `docker run`

```bash
docker run -d --name all2pod --restart unless-stopped \
  -p 8080:8080 \
  -e TORBOX_API_KEY=... \
  -e ADMIN_USER=admin -e ADMIN_PASSWORD=change-me \
  -e PUBLIC_URL=https://podcast.example.com \
  -e TMDB_API_KEY=... \
  -e DATA_PATH=/config/feeds.json -e MEDIA_DIR=/data \
  -v "$PWD/config:/config" -v "$PWD/data:/data" \
  ghcr.io/jdmry/all2pod:latest
```

## Configuration

| Variable         | Required | Description                                                                                 |
|------------------|----------|---------------------------------------------------------------------------------------------|
| `TORBOX_API_KEY` | yes      | Your TorBox API key (TorBox dashboard → Settings → API).                                     |
| `ADMIN_USER`     | yes      | Username for the admin UI (HTTP Basic Auth).                                                 |
| `ADMIN_PASSWORD` | yes      | Password for the admin UI.                                                                   |
| `PUBLIC_URL`     | yes      | Full public URL of the app, e.g. `https://podcast.example.com`.                             |
| `TMDB_API_KEY`   | no       | Free TMDB key for posters and search. Get one at https://www.themoviedb.org/settings/api    |
| `COMET_URL`      | no       | Base URL of a [Comet](https://github.com/g0ldyy/comet) instance for the search-first flow.  |
| `DATA_PATH`      | no       | JSON store path (default `/data/feeds.json`). Its directory also holds ffmpeg scratch + the image cache — keep it on **local disk**. |
| `MEDIA_DIR`      | no       | Directory for converted videos + uploads. Defaults to the `DATA_PATH` directory. May be a network mount. |
| `PORT`           | no       | Internal listen port (default `8080`).                                                       |

### Storage

all2pod separates hot local state from bulk media, so the media directory can be a network mount:

- **`/config`** (local disk) — `feeds.json`, ffmpeg scratch, image cache. Small, frequent, atomic writes.
- **`/data`** (local or a mount) — finished converted videos + uploads. Large, sequential, write-once.

The compose file maps these to `./config` and `./data`. Point `./data` at a mounted volume if you want cheap bulk storage; keep `./config` local.

## Deploy on CapRover

This repo ships a `captain-definition` that references the published image, so you can create a CapRover app from it directly. Enable **Has Persistent Data**, mount a **local** volume for `/config` and a volume for `/data`, then set the environment variables above (use `DATA_PATH=/config/feeds.json` and `MEDIA_DIR=/data`).

## Usage

1. Create a feed (movies / show / audio).
2. Add titles — by search (needs `TMDB_API_KEY` + `COMET_URL`), from your TorBox library, or by uploading a file.
3. Video items convert to MP4 in the background; audio is ready immediately.
4. Copy the feed's RSS URL and add it in your podcast app (Apple Podcasts → Library → Add a Show by URL).

Each feed has its own token, so you can share one feed without exposing the others.

## Notes & limits

- The image bundles **ffmpeg** (conversion, probing, artwork). Conversions run one at a time.
- **Episode artwork**: Apple Podcasts only shows per-episode artwork for public shows in its catalog, and for video it renders a video frame rather than the RSS image — so per-episode artwork on private/URL-added feeds is not shown by Apple Podcasts. Channel artwork works everywhere; other apps handle per-episode audio artwork.
- You bring your own TorBox account and (optionally) your own Comet instance. all2pod does not proxy or host any content.

## Attribution

This product uses the TMDB API but is not endorsed or certified by TMDB.

## License

The files in this repository are MIT-licensed (see `LICENSE`). The all2pod application image itself is distributed as a prebuilt binary and its source is not published.
