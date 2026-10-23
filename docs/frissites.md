# Az AI-router frissítése

## A repó új neve (2026. október)

A repó `kzkz22/skipwise-airouter` lett (korábban `dwyt-airouter`). A GitHub a régi címről átirányít, így a régi
frissítőparancs is működik. A telepített rendszeren semmi nem változik: a parancs (`dwyt-airouter`), a mappa
(`/opt/dwyt-airouter`), a szolgáltatások (`dwyt-llama-swap`, `dwyt-litellm`) és a konténernevek maradnak.

## A program

```bash
/opt/dwyt-airouter/venv/bin/pip install --force-reinstall --no-deps git+https://github.com/kzkz22/skipwise-airouter
dwyt-airouter --version        # az új változat száma
dwyt-airouter generate         # új változat új beállításokat írhat
systemctl restart dwyt-litellm # ha a generate szerint a litellm.yaml változott
```

A llama-swap a `--watch-config` miatt magától újratölti a `llama-swap.yaml`-t. Futó munka közben ne frissíts: a konténert leállíthatja.

## Modellek módosítása

1. `nano /etc/dwyt-airouter/models.yaml`, majd `dwyt-airouter generate`.
2. Ha a `models.ini` vagy a `downloads.json` változott: másold át a GPU-gépre, és ott futtasd a `dwyt-gpu download`-ot (dwyt-gpu, frissites.md).
3. Ha a `litellm.yaml` változott: `systemctl restart dwyt-litellm`.

## Frissítés a 0.2-re (modellfájlok repónként külön mappában)

Két Hugging Face-repóban gyakran ugyanolyan nevű fájl van (például `mmproj-F16.gguf`), és a régi elrendezésben a második felülírta volna az elsőt. A 0.2-től minden fájl a GPU-gépen a `gguf/<szervezet>/<repó>/` mappába kerül, a `models.ini` is oda mutat, a `downloads.json` pedig 2-es változatú.

1. **Előbb a GPU-gépen** frissítsd a `dwyt-gpu`-t (a régi letöltő a 2-es listát nem fogadja el): dwyt-gpu, frissites.md.
2. Itt a program frissítése (fent), majd `dwyt-airouter generate`.
3. A `models.ini`-t és a `downloads.json`-t másold a GPU-gépre, és ott `dwyt-gpu download`. A már letöltött fájlokat a régi helyükről nem veszi át: vagy újra letölti őket, vagy előtte kézzel áthelyezed őket az új mappába (`gguf/unsloth/<repó>/`).

A `chat-template-kwargs` értéke egy JSON-objektum szövegként, egyszeres idézőjelben:

```yaml
chat-template-kwargs: '{"preserve_thinking": true, "reasoning_effort": "medium"}'
```

Ha egy stabil név (LiteLLM nélkül) kikapcsolja a gondolkodást, az `enable_thinking` ehhez adódik hozzá, a modell saját kapcsolói megmaradnak.

## A `kerdes` név (dwyt 6. kör)

A dwyt központjában feltett kérdésekhez a futtató a `kerdes` nevet használja (gondolkodással, felhős tartalékkal). Ha a `models.yaml`-od régebbi, a `names:` alá vedd fel:

```yaml
  - name: kerdes
    model: Qwen3.6-35B-A3B-Gnrl
    thinking: true
    fallback: cloud/qwen3.8-flash
```

Utána `dwyt-airouter generate`, `systemctl restart dwyt-litellm`, a futtatón `dwyt-runner config set llm_ask_model kerdes`.

## Modell-leírás a dwyt-nek (0.3)

A 0.3-tól a `generate` a `litellm.yaml` minden nevéhez leírást ír (`model_info.dwyt`): mire való, melyik modell fut mögötte, helyben vagy felhőben, gondolkodással vagy anélkül, és mi válaszol, ha a GPU-gép ki van kapcsolva. A dwyt-futtató (0.7-től) ezt küldi a központnak, a Futtatók oldal ebből mutatja a modelleket.

1. A program frissítése (fent), majd `dwyt-airouter generate` és `systemctl restart dwyt-litellm`.
2. Ha egy név leírását magad akarod megadni, a `models.yaml`-ban a névhez írd: `description: "Kérdések a videókról, alapos válasz."`, majd újra `generate` és újraindítás.

Ellenőrzés:

```bash
KEY=$(sed -n 's/^LITELLM_MASTER_KEY=//p' /etc/dwyt-airouter/litellm.env)
curl -s -H "Authorization: Bearer $KEY" http://127.0.0.1:4000/v1/model/info | python3 -m json.tool | grep -A4 '"dwyt"'
```

## Mielőtt a llama-swapot vagy a LiteLLM-et frissíted

A beállításaid nem a programokban vannak, hanem az `/etc/dwyt-airouter` mappában (`models.yaml`, `airouter.env`,
`litellm.env`, `generated/`). A lenti lépések ehhez nem nyúlnak: csak a llama-swap programfájlját és a LiteLLM
Python-csomagjait cserélik. A biztonság kedvéért mégis:

1. Futó munka ne legyen (a dwyt Munkák oldalán semmi „fut” állapotban): az újraindítás megszakítaná.
2. Pillanatkép a Proxmoxon (a gazdagépen; a `CT` az AI-router konténerének száma):

   ```bash
   pct snapshot CT frissites-elott
   ```

   Ha bármi elromlik: `pct rollback CT frissites-elott`, és minden pontosan olyan, mint előtte.
   Ha jó lett, pár nap múlva törölheted: `pct delsnapshot CT frissites-elott`.
3. A konténerben mentsd el a beállításokat és a mostani állapotot (ebből a pillanatkép nélkül is visszaállsz):

   ```bash
   D=/root/airouter-mentes-$(date +%Y%m%d)
   mkdir -p $D
   tar czf $D/etc-dwyt-airouter.tgz /etc/dwyt-airouter /etc/systemd/system/dwyt-*.service
   cp -p /opt/dwyt-airouter/bin/llama-swap $D/llama-swap.regi
   /opt/dwyt-airouter/litellm/bin/pip freeze > $D/litellm-csomagok.txt
   /opt/dwyt-airouter/bin/llama-swap --version; /opt/dwyt-airouter/litellm/bin/pip show litellm | grep ^Version
   ```

## llama-swap

A kipróbált változat a v266 (a v262-höz képest a beállítás nem változott; a dwyt-airouter tesztjei és egy
élő próba – modellváltás, aliasnév, 300 MB-os feltöltés a Whisperhez az `/upstream` úton – ugyanúgy futnak).
Frissítés:

```bash
V=266
rm -rf /tmp/llama-swap-uj && mkdir /tmp/llama-swap-uj
curl -sSfL https://github.com/mostlygeek/llama-swap/releases/download/v$V/llama-swap_${V}_linux_amd64.tar.gz \
  | tar xz -C /tmp/llama-swap-uj llama-swap
/tmp/llama-swap-uj/llama-swap --version
/tmp/llama-swap-uj/llama-swap -config /etc/dwyt-airouter/generated/llama-swap.yaml -validate   # hiba = állj meg
install -m 755 /tmp/llama-swap-uj/llama-swap /opt/dwyt-airouter/bin/llama-swap
systemctl restart dwyt-llama-swap
systemctl status dwyt-llama-swap --no-pager | head -5
curl -s http://127.0.0.1:8080/v1/models | head -c 300; echo
```

Visszaállás: `install -m 755 $D/llama-swap.regi /opt/dwyt-airouter/bin/llama-swap && systemctl restart dwyt-llama-swap`
(a `$D` a fenti mentés mappája).

## LiteLLM

A kipróbált változat az 1.106.2 (a generált `litellm.yaml`-lal ugyanúgy működik, mint az 1.104.2: a nevek, a
`model_info.dwyt` leírások, a folyamként kért válasz és a felhős tartalék, ha a helyi szerver nem válaszol).
Frissítés:

```bash
/opt/dwyt-airouter/litellm/bin/pip install 'litellm[proxy]==1.106.2'
systemctl restart dwyt-litellm
journalctl -u dwyt-litellm -n 50 --no-pager     # indult-e hiba nélkül
KEY=$(sed -n 's/^LITELLM_MASTER_KEY=//p' /etc/dwyt-airouter/litellm.env)
curl -s -H "Authorization: Bearer $KEY" http://127.0.0.1:4000/v1/models | python3 -m json.tool | grep '"id"'
curl -s -H "Authorization: Bearer $KEY" http://127.0.0.1:4000/v1/chat/completions -H 'Content-Type: application/json' \
  -d '{"model": "kerdes", "messages": [{"role": "user", "content": "Mondj egy szót!"}]}' | head -c 400; echo
```

A neveknek ugyanazoknak kell lenniük, mint előtte, és a próbakérdésre válasznak kell jönnie. Ha a GPU-gép ki
van kapcsolva, a `kerdes` a felhős tartalékból válaszol – ez is jó jel.

Visszaállás (pontosan a régi csomagokkal):

```bash
/opt/dwyt-airouter/litellm/bin/pip install -r $D/litellm-csomagok.txt
systemctl restart dwyt-litellm
```

## A dwyt futtatója a frissítés után

A futtató óránként újra lekéri a modellek leírását, és a központ Futtatók oldala ebből mutatja őket. Ha azonnal
látni akarod, indítsd újra a futtatót; utána egy kérdés vagy összefoglaló a dwyt-ben a teljes utat kipróbálja.
