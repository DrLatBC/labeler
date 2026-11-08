# Labeler

Web app that prints labels to a Phomemo M110 over Web Bluetooth. Replaces the Print Master app.

Live: https://drlatbc.github.io/labeler/ (Chrome on Android or desktop; Web Bluetooth does not work on iOS).

## Homebox

Served at https://homebox.drlat.dev/labeler/ so it shares the Homebox login cookie. The Homebox tab only appears when `/api/v1/status` on the same origin answers as Homebox.

- Search: `GET /api/v1/entities?q=&pageSize=25&orderBy=` (`isLocation=true` for locations)
- Labels are drawn by Labeler, not Homebox's labelmaker (its fixed font clips names at 40 x 30 mm): QR to `/item/{id}` or `/location/{id}` (the same links Homebox's labels use), name auto-sized beside it, parent location and asset ID below, tags as black pills along the bottom (one pill is centered). Labels under 30 mm tall, stacked labels, and tags too many for one row print the tags as a text line instead. Tall labels stack the QR above the text. QR from [qrcode-generator](https://github.com/kazuhikoarase/qrcode-generator) 1.4.4 on cdnjs.
- Add photos: after picking an item or location, open a live camera and shoot any number of photos in a row (tap a thumbnail to review the shots full screen, swipe between them, delete any, and star one to upload it as the main photo with `primary=true`), or pick many from the gallery. Homebox's own edit page takes one file at a time. Each goes to `POST /api/v1/entities/{id}/attachments` (`file`, `name`, `type=photo`); photos over 9.5 MB are re-encoded as JPEG first because Homebox and nginx cap a request at 10 MB.
- Main photo: the picked item's photos show as a strip (`GET /api/v1/entities/{id}` attachments of type photo, thumbnails where Homebox has made one). Tap one to view full size; the star makes it the main photo via `PUT /api/v1/entities/{id}/attachments/{attachmentId}` (`type`, `title`, `primary: true`), and Homebox clears the old one.
- Delete photos: the × on a thumbnail or the trash in the full-size view removes it from Homebox after a confirm (`DELETE /api/v1/entities/{id}/attachments/{attachmentId}`). Homebox leaves an item with no main photo when the main one goes, so Labeler stars the next one.
- Share target: sharing a Homebox `/item/{id}`, `/location/{id}` or `/a/{assetId}` link to the installed app opens that label (`GET /api/v1/entities/{id}`, or `q=#<assetId>` across items then locations)

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
