# Labeler

Web app that prints labels to a Phomemo M110 over Web Bluetooth. Replaces the Print Master app.

Live: https://drlatbc.github.io/labeler/ (Chrome on Android or desktop; Web Bluetooth does not work on iOS).

## Homebox

Served at https://homebox.drlat.dev/labeler/ so it shares the Homebox login cookie. The Homebox tab only appears when `/api/v1/status` on the same origin answers as Homebox.

- Search: `GET /api/v1/entities?q=&pageSize=25&orderBy=` (`isLocation=true` for locations)
- Label image: `GET /api/v1/labelmaker/{item|location|asset}/{id}`, sized by `HBOX_LABEL_MAKER_WIDTH=320` / `HBOX_LABEL_MAKER_HEIGHT=240` in `~/homebox/homebox.env` on the box
- Share target: sharing a Homebox `/item/{id}`, `/location/{id}` or `/a/{assetId}` link to the installed app opens that label

Deploy: `ssh lore_capsule 'cd /var/www/labeler && git pull --ff-only'`. nginx serves it via a `location /labeler/` alias block in `/etc/nginx/sites-available/homebox`.

## Run locally

```
python -m http.server 8765
```

Web Bluetooth needs https or localhost. To test from a phone against the PC, add `http://<pc-ip>:8765` to `chrome://flags/#unsafely-treat-insecure-origin-as-secure` and type the URL with `http://`.

## M110 protocol

Taken from [pyphomemo](https://github.com/mkuhlmann/pyphomemo), [phomymo](https://github.com/transcriptionstream/phomymo) and [phomemo-tools](https://github.com/vivier/phomemo-tools).

- Service `0xff00`, write `0xff02`, notify `0xff03`
- Head 384 dots, 8 dots/mm; raster rows MSB = leftmost pixel, 1 = black
- Job: `1b 4e 0d 05` (speed), `1b 4e 04 <1-15>` (density), `1f 11 <0a gaps | 0b continuous | 26 marks>`, `1d 76 30 00 wL wH hL hH`, bitmap, `1f f0 05 00 1f f0 03 00`
- Pacing: 30 ms after each setup command, bitmap in 128-byte chunks with 20 ms between, 300 ms before the footer, 500 ms after. Sending it unpaced makes the printer flash and drop the job.
