# Modellek: új modell kipróbálása és ajánlás

## Új modell kipróbálása

Egyetlen fájlt szerkesztesz: az AI-routeren az `/etc/dwyt-airouter/models.yaml`-t. Minden mást ebből készít a
`dwyt-airouter generate`.

1. `nano /etc/dwyt-airouter/models.yaml`
   - helyi modell: a `local_models:` alá (név, Hugging Face-fájl, llama-server-kapcsolók, tokenkorlátok);
   - felhős modell: a `cloud: models:` alá;
   - ha egy szerep (`osszefoglalo`, `osszefoglalo-helyi`, `kerdes`, …) is az új modellt használja: a `names:` alatt
     a `model:` értéke.
2. `dwyt-airouter generate` – a végén kiírja, mi változott és mit kell még tenni.
3. Ha a `models.ini` vagy a `downloads.json` változott (új helyi modellnél mindig):
   - mindkettőt másold a GPU-gép modellmappájába;
   - ott: `dwyt-gpu download downloads.json --dir $DWYT_MODELS_DIR`;
   - ha a `dwyt-llama` épp fut: `docker stop dwyt-llama` (a következő kérés már az új presetekkel indítja).
4. Ha a `litellm.yaml` változott: `systemctl restart dwyt-litellm`. A llama-swap magától újratölti a sajátját.
5. Ellenőrzés: a LiteLLM `/v1/models` listájában ott az új név, és egy próbakérdésre válasz jön
   ([frissites.md](frissites.md), LiteLLM rész).
6. Hogy a Skipwise-ban választható legyen, a futtatón:
   - `skipwise-runner config set llm_models "UjModell,MasikModell"` (nyilvános források);
   - `llm_private_models`: saját felvételekhez, csak ha azokhoz is engeded.

   Ugyanez a futtató ablakában vagy a központ Futtatók → Beállítás oldalán is megadható. Utána az elem oldalán az
   „Új összefoglaló” alatt választható, így két modell ugyanarra a videóra egymás mellett összevethető.

## Ajánlás a többnyelvűség első köréhez (2026. október)

Nyelvek: magyar, angol, német, olasz, spanyol, francia. Feltétel: a helyi modell elférjen az RTX 3090 24 GB-jában
legalább kb. 180k kontextussal; a felhős legyen olcsó. Magyarra kis modellnél nincs megbízható mérés, ezért ezek
**próbára** javasolt modellek, a döntést a saját próba hozza meg.

### Helyi: Gemma 4 26B-A4B (MoE) a Qwen3.6-35B-A3B mellett

- Többnyelvűségre tanítva, 140+ nyelv, Apache 2.0, képet is fogad.
- Egy 24 GB-os összevetésben spanyol/francia/kínai ellenőrzéseken a Gemma 4 31B kicsit jobb volt a Qwen3.8-27B-nél
  (14/18 kontra 13/18; a szerző szerint ez nem minőségi rangsor).
- Elfér hosszú kontextussal: Q4-ben 32K-nál kb. 18 GB / 107 tok/s, 256K-nál kb. 23 GB / 64 tok/s.
- A Gemma 4 31B (dense) 64K fölött nem fér el, ezért nem jó választás.
- A Qwen3.8-27B marad a kódolásra és az ügynöki munkára.
- A gondolkodás kapcsolója ugyanaz (`enable_thinking`), a `thinking: true/false` változatlanul működik.

```yaml
local_models:
  - name: Gemma4-26B-A4B-Gnrl
    file: unsloth/gemma-4-26B-A4B-it-GGUF/gemma-4-26B-A4B-it-UD-Q4_K_XL.gguf   # 17 GB
    args: {ctx-size: 196608, cache-type-k: q8_0, cache-type-v: q8_0, temp: 1.0, top-p: 0.95, top-k: 64}
    max_input_tokens: 196608
    max_output_tokens: 32768
```

A mintavétel az Unsloth ajánlása; a 196608-as kontextus becslés, ha nem tölt be, csökkentsd.

### Felhő (OpenRouter, USD / 1M token)

| Modell | Bemenet | Kimenet | Megjegyzés |
|---|---|---|---|
| qwen/qwen3.8-flash (a mostani) | 0,15 | 0,47 | egy szolgáltató (Alibaba) |
| **google/gemma-4-31b-it** | **0,10** | **0,34** | 140+ nyelv, gondolkodás kapcsolható, 13 szolgáltató |
| deepseek/deepseek-v4.1-flash | 0,07 | 0,50 | legolcsóbb bemenet, 1M kontextus; a szolgáltatók ára és minősége szór |
| xiaomi/mimo-v2.6-flash | 0,10–0,14 | 0,28 | |

Javaslat: **Gemma 4 31B**: olcsóbb a mostaninál, és a helyi Gemmával egy családba tartozik, így a helyi → felhős
tartalék hasonló stílusú. Az összefoglalónál a bemenet ára számít; egy egyórás videó összefoglalója mindegyikkel
kb. 0,3–0,4 cent, ezért a minőség döntsön.

```yaml
cloud:
  models:
    - name: cloud/gemma-4-31b
      model: openrouter/google/gemma-4-31b-it
```

### A próba

1. Vedd fel a jelölteket a `models.yaml`-ba és a futtató `llm_models` listájába (fent).
2. 5–6 videóra (magyar, angol, német, olasz, spanyol, francia) kérj új összefoglalót és fordítást mindegyikkel.
3. Ha a Gemma jobb, a `names:` alatt az `osszefoglalo`, `osszefoglalo-helyi` és `kerdes` modelljét írd át rá,
   a `fallback`-et a felhős Gemmára; a dwyt-ben semmi mást nem kell változtatni.

### Források

- [Best Local LLM for 24GB VRAM (openclawdc)](https://openclawdc.com/blog/best-local-llm-24gb-vram/)
- [Qwen3.8 vs Qwen3.6 vs Gemma 4: 24GB GPU Test (kingy.ai)](https://kingy.ai/blog/qwen3-8-27b-vs-qwen3-6-27b-vs-gemma-4-31b/)
- [Gemma 4 31B vs Qwen3.8-27B (benchlm)](https://benchlm.ai/compare/gemma-4-31b-vs-qwen3-8-27b)
- [Hardware for Gemma 4 (hardware-corner)](https://www.hardware-corner.net/hardware-for-gemma-4-llm/)
- [Gemma 4 – How to Run Locally (Unsloth)](https://unsloth.ai/docs/models/gemma-4)
- [unsloth/gemma-4-26B-A4B-it-GGUF](https://huggingface.co/unsloth/gemma-4-26B-A4B-it-GGUF/tree/main)
- [OpenRouter: Gemma 4 31B](https://openrouter.ai/google/gemma-4-31b-it),
  [Qwen3.8 Flash](https://openrouter.ai/qwen/qwen3.8-flash),
  [DeepSeek V4.1 Flash](https://openrouter.ai/deepseek/deepseek-v4.1-flash),
  [MiMo-V2.6-Flash](https://openrouter.ai/xiaomi/mimo-v2.6-flash)
- [OpenRouter API pricing (benchlm)](https://benchlm.ai/hosts/openrouter)
