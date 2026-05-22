# waveshare-12in48-gfx

ESP32 firmware that drives a [Waveshare 12.48" tri-color e-paper module (B)](https://www.waveshare.com/12.48inch-e-paper-module-b.htm) (1304 × 984, black/white/red) using Adafruit GFX as the drawing layer.

Waveshare provides an SDK with primitive drawing (`GUI_Paint`). Adafruit GFX gives you the wider ecosystem — FreeFonts, custom converted TTFs, primitives, a familiar API — but doesn't out of the box work with this display on a stock no-PSRAM ESP32. This repo is what made that work.

## Why it's not just plug-and-play

| Constraint | What it forces |
|---|---|
| Adafruit GFX uses `1=white, 0=black`; Waveshare's `GUI_Paint` uses the opposite | Need a bridging buffer with the right convention before handing bytes to the EPD driver |
| Full 1304 × 984 1-bit framebuffer is ~160 KB; no-PSRAM ESP32 max contiguous alloc is ~114 KB | Can't hold the full framebuffer. Must split into halves |
| The panel is four sub-panels driven via two SPI pairs (M2/S2 top, M1/S1 bottom) with separate DC/RST lines | The render path has to slice the buffer per panel and clock it out via the right CS pins |
| No partial refresh (tri-color hardware) | Every update is a full ~12s refresh — render strategy matters |
| Byte-by-byte SPI is too slow | Need bulk scanline sends |
| VSPI default pins (18/19/23/5) overlap with display GPIOs | Must use HSPI |

## The bridge: `EinkCanvas`

[`lib/EPD12in48/EinkCanvas.h`](lib/EPD12in48/EinkCanvas.h) is an `Adafruit_GFX` subclass that exposes full 1304 × 984 coordinates but allocates only a 1304 × 492 buffer. You call `setHalf(TOP)`, draw the full scene, render that buffer to the top two panels, then `setHalf(BOTTOM)` and repeat. Pixels outside the active half are silently discarded by `drawPixel`, so text and shapes that cross the y=492 line just need to be drawn twice — no per-object clipping logic in the calling code.

```cpp
class EinkCanvas : public Adafruit_GFX {
public:
    enum Half { TOP = 0, BOTTOM = 492 };
    void setHalf(Half h) { _yOffset = (int16_t)h; }

    void drawPixel(int16_t x, int16_t y, uint16_t color) override {
        if (x < 0 || x >= FULL_W || y < 0 || y >= FULL_H) return;
        int16_t localY = y - _yOffset;
        if (localY < 0 || localY >= HALF_H) return;
        // ...write into _buffer
    }
};
```

A refresh runs four passes: top-black, bottom-black, top-red, bottom-red. Same calling code each time; only the colour filter and the active half differ. Data transfer is ~344 ms total on HSPI @ 10 MHz with bulk scanline sends.

## Build + flash

PlatformIO. Pulls Adafruit GFX, ArduinoJson, PubSubClient, ArduinoOTA.

```bash
cp src/config.example.h src/config.h   # WiFi + MQTT credentials
pio run -t upload
```

One thing not in any of the libraries: the EPD driver needs `int Version = 1;` declared globally to select the V1 vs V2 hardware variant. See `src/main.cpp`.

## Display data via MQTT

The firmware subscribes to a JSON topic, parses it, and renders a fixed layout (sections, weather, a quote at the bottom). A red-text convention uses `\x01` as a byte marker inside strings: text before the marker renders black, text after renders red. The `server/` directory contains a Python service that produces this payload — see below.

## Server (background)

`server/` is a Python service that assembles the daily payload (weather from Meteoblue, upcoming dates from a YAML file, a rotating quote) and publishes it via MQTT. It's working code, not a library — included so the firmware has something real to render against.

```bash
cd server
cp .env.example .env                            # MQTT + Meteoblue creds
cp data/dates.example.yml data/dates.yml        # personal dates
docker compose up -d
```

## What comes after this

The same daily-brief idea now runs on different hardware (Seeed reTerminal E1003, IT8951-driven 10.3" grayscale e-paper) under ESPHome instead of PlatformIO. That's a separate repo: [`esphome-eink-dashboard`](https://github.com/codeincontext/esphome-eink-dashboard). Different driver, different framework, different code; the data sources are similar.

This repo stays around as a reference for the Waveshare 12.48" + Adafruit GFX combination on stock ESP32.
