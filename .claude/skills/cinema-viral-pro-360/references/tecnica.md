# Protocolos Técnicos — Cinema Viral Pro 360

Referência de execução. Use junto com `../SKILL.md` (protocolo e critérios)
e `direcao.md` (decisões criativas). Comandos testados com FFmpeg 6.x; em
versões antigas, alguns parâmetros mudam (indicados quando relevante).

Sumário:
1. Detecção de ferramentas (sem instalar nada)
2. Estrutura de projeto e preservação dos originais
3. Análise técnica com ffprobe
4. Conformação para 1080x1920, montagem e transições
5. Color grading com FFmpeg
6. Áudio: limpeza, mixagem, ducking e loudness
7. Legendas animadas (ASS)
8. Python e OpenCV: cenas, desfoque de rostos, enquadramento
9. Exportação e controle de qualidade (QC)
10. Solução de problemas

Convenções: `$IN`, `$OUT` e caminhos são exemplos; sempre use aspas em
caminhos com espaços. Nunca use `-y` apontando para a pasta de originais.

---

## 1. Detecção de ferramentas (sem instalar nada)

```bash
for t in ffmpeg ffprobe python3; do
  command -v "$t" >/dev/null && echo "OK  $t: $($t -version 2>&1 | head -1)" || echo "FALTA $t"
done
python3 - <<'PY'
import importlib
for m in ("cv2", "numpy"):
    try:
        mod = importlib.import_module(m); print("OK ", m, getattr(mod, "__version__", ""))
    except ImportError:
        print("FALTA", m)
PY
# Filtros FFmpeg necessários
ffmpeg -hide_banner -filters 2>/dev/null | \
  grep -E ' (zscale|tonemap|lut3d|subtitles|ass|loudnorm|ebur128|xfade|sidechaincompress|huesaturation|drawtext|gblur|afftdn) ' \
  | awk '{print "OK  filtro", $2}'
ffmpeg -hide_banner -encoders 2>/dev/null | grep -E ' (libx264|libx265|aac) ' | awk '{print "OK  encoder", $2}'
```

Matriz de decisão:

| Disponível | Capacidade |
|---|---|
| Só FFmpeg | Tudo de §3–§7 e §9 (cortes, cor, áudio, legendas, QC). Desfoque apenas por região fixa/manual (§8.4). |
| FFmpeg + Python + NumPy | Scripts de sincronização por BPM, relatórios de QC automatizados. |
| FFmpeg + Python + OpenCV | Detecção de cenas, desfoque automático de rostos, enquadramento inteligente. |
| Nada disponível | Entregar roteiro, decupagem, instruções de edição para app (CapCut, Premiere, DaVinci) e checklist de QC manual. |

Se faltar algo: **informe**, ofereça a alternativa da tabela e **peça
autorização** antes de qualquer instalação.

---

## 2. Estrutura de projeto e preservação dos originais

### 2.1 Estrutura

```
AAAA-MM-DD_<tema>/
├── 00_originais/     # cópia ou link dos brutos — SOMENTE LEITURA
├── 01_proxies/       # versões leves para análise (opcional)
├── 02_trabalho/      # trechos conformados, intermediários
├── 03_audio/         # trilha, foley, voz tratada
├── 04_legendas/      # .ass/.srt
├── 05_exports/       # finais: <tema>_v01_corte.mp4, <tema>_v02_final.mp4
├── 06_qc/            # relatórios, folhas de contato, medições
└── relatorio.md
```

Se os originais já estiverem em outra pasta do usuário, **não os mova**:
leia diretamente de lá ou crie links simbólicos em `00_originais/`.

### 2.2 Registro de integridade (início do projeto)

```bash
cd "00_originais"
find . -type f ! -name 'SHA256SUMS' -print0 | sort -z | xargs -0 sha256sum > ../06_qc/SHA256SUMS_originais.txt
chmod -R a-w .        # proteção contra escrita (cópias locais; não altere permissões em pastas que não são suas sem autorização)
cd ..
```

### 2.3 Regras durante a edição
- Toda saída vai para `02_trabalho/` ou `05_exports/`; nunca para `00_originais/`.
- Antes de qualquer comando, confira que o caminho de saída é diferente do
  de entrada. Nunca use edição "no lugar".
- Versione: `_v01`, `_v02`… em vez de sobrescrever finais aprovados.

### 2.4 Verificação final de integridade

```bash
(cd "00_originais" && sha256sum -c ../06_qc/SHA256SUMS_originais.txt) \
  && echo "INTEGRIDADE OK" || echo "FALHA DE INTEGRIDADE — INVESTIGAR"
```

Resultado obrigatório no relatório final.

---

## 3. Análise técnica com ffprobe

### 3.1 Resumo por arquivo

```bash
ffprobe -v error -select_streams v:0 \
  -show_entries stream=codec_name,profile,width,height,pix_fmt,r_frame_rate,avg_frame_rate,color_space,color_transfer,color_primaries,color_range,bit_rate:stream_side_data=rotation:format=duration,size \
  -of default=nw=1 "$IN"
ffprobe -v error -select_streams a:0 \
  -show_entries stream=codec_name,sample_rate,channels,bit_rate -of default=nw=1 "$IN"
```

### 3.2 Inventário em lote (CSV)

```bash
echo "arquivo,largura,altura,rotacao,fps_r,fps_avg,transfer,duracao" > 06_qc/inventario.csv
for f in 00_originais/*; do
  w=$(ffprobe -v error -select_streams v:0 -show_entries stream=width  -of csv=p=0 "$f")
  h=$(ffprobe -v error -select_streams v:0 -show_entries stream=height -of csv=p=0 "$f")
  rot=$(ffprobe -v error -select_streams v:0 -show_entries stream_side_data=rotation -of csv=p=0 "$f")
  r=$(ffprobe -v error -select_streams v:0 -show_entries stream=r_frame_rate   -of csv=p=0 "$f")
  a=$(ffprobe -v error -select_streams v:0 -show_entries stream=avg_frame_rate -of csv=p=0 "$f")
  t=$(ffprobe -v error -select_streams v:0 -show_entries stream=color_transfer -of csv=p=0 "$f")
  d=$(ffprobe -v error -show_entries format=duration -of csv=p=0 "$f")
  echo "\"$f\",$w,$h,${rot:-0},$r,$a,${t:-unknown},$d" >> 06_qc/inventario.csv
done
```

### 3.3 Interpretação e alertas

| Observação | Significado | Ação |
|---|---|---|
| `rotation` = -90/90 | Vídeo vertical gravado com metadado de rotação | O FFmpeg aplica autorrotação por padrão; confira dimensões após conformar. |
| `r_frame_rate` ≠ `avg_frame_rate` | Taxa de quadros variável (VFR, comum em celular) | Converter para CFR na conformação (`fps=30`) antes de editar. |
| `color_transfer` = `arib-std-b67` | HDR HLG (iPhone "HDR/Dolby Vision") | Tonemapping para SDR BT.709 (§5.5). |
| `color_transfer` = `smpte2084` | HDR PQ/HDR10 | Tonemapping para SDR BT.709 (§5.5). |
| `pix_fmt` 10 bits (`yuv420p10le`) | Maior latitude de cor | Gradue antes de converter para 8 bits. |
| fps ≥ 100 | Material para câmera lenta | Reserve para speed ramp (§4.5). |
| Áudio mono ou 44,1 kHz | Comum em celular | Converter para 48 kHz estéreo na mixagem. |

---

## 4. Conformação para 1080x1920, montagem e transições

Todos os trechos devem ter **mesma resolução, SAR 1:1, fps, formato de
pixel e base de tempo** antes de concatenar ou aplicar `xfade`.

### 4.1 Trecho de origem (cortar sem perder qualidade)
Extraia o trecho já conformado (recodificando — cortes com `-c copy` só
caem em keyframes e geram imprecisão):

```bash
ffmpeg -hide_banner -ss 12.40 -i "$IN" -t 3.20 \
  -vf "scale=1080:1920:force_original_aspect_ratio=increase:flags=lanczos,crop=1080:1920,setsar=1,fps=30,format=yuv420p" \
  -c:v libx264 -preset medium -crf 14 -c:a aac -b:a 320k -ar 48000 -ac 2 \
  02_trabalho/c01.mp4
```

`-ss` antes de `-i` é rápido e, com recodificação, preciso no FFmpeg moderno.
Use CRF 12–16 para intermediários (margem para gerações seguintes).

### 4.2 Horizontal → vertical
**a) Recorte central (crop)** — quando a ação está no centro:
`scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920`

**b) Recorte deslocado** — quando a ação está à esquerda/direita
(x em pixels após a escala; para origem 1920x1080 escalada a 3414x1920,
x vai de 0 a 2334):
`scale=-2:1920,crop=1080:1920:x=900:y=0`

**c) Recorte com movimento (pan)** — acompanha a ação de x=400 a x=1600 em 3 s:
`scale=-2:1920,crop=1080:1920:x='400+(1600-400)*min(t/3\,1)':y=0`

**d) Fundo desfocado** — quando a ação ocupa todo o quadro horizontal:

```bash
ffmpeg -hide_banner -i "$IN" -filter_complex \
"[0:v]split=2[bg][fg]; \
 [bg]scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920,gblur=sigma=40,eq=brightness=-0.06:saturation=0.8[bgb]; \
 [fg]scale=1080:-2:flags=lanczos[fgs]; \
 [bgb][fgs]overlay=(W-w)/2:(H-h)/2,setsar=1,fps=30,format=yuv420p[v]" \
  -map "[v]" -map 0:a? -c:v libx264 -crf 14 -c:a aac -b:a 320k -ar 48000 02_trabalho/c02.mp4
```

Preferência: (a)/(b)/(c) para ação; (d) apenas quando cortar perderia
informação essencial (técnica com os dois atletas abertos na horizontal).
Para enquadramento automático, ver §8.3.

### 4.3 Concatenação (corte seco)
Com trechos conformados de forma idêntica:

```bash
printf "file '%s'\n" "$PWD"/02_trabalho/c0{1,2,3}.mp4 > 02_trabalho/lista.txt
ffmpeg -hide_banner -f concat -safe 0 -i 02_trabalho/lista.txt -c copy 02_trabalho/montagem.mp4
```

Se houver qualquer divergência de parâmetros, use o filtro `concat`
(recodifica):

```bash
ffmpeg -i c01.mp4 -i c02.mp4 -i c03.mp4 -filter_complex \
"[0:v][0:a][1:v][1:a][2:v][2:a]concat=n=3:v=1:a=1[v][a]" -map "[v]" -map "[a]" \
  -c:v libx264 -crf 14 -c:a aac -b:a 320k montagem.mp4
```

### 4.4 Transições com `xfade`
`offset` = duração acumulada até o início da transição
(= duração do clipe A − duração da transição).

```bash
# A tem 4,0 s; fusão de 0,4 s → offset 3,6
ffmpeg -i A.mp4 -i B.mp4 -filter_complex \
"[0:v][1:v]xfade=transition=fade:duration=0.4:offset=3.6,format=yuv420p[v]; \
 [0:a][1:a]acrossfade=d=0.4[a]" -map "[v]" -map "[a]" -c:v libx264 -crf 14 -c:a aac -b:a 320k AB.mp4
```

Transições úteis: `fade`, `fadeblack` (passagem de tempo), `slideleft`,
`smoothleft` (simula whip pan), `zoomin`, `circleopen` (revelação).
Para encadear 3+ clipes, o offset de cada transição soma as durações já
montadas menos as sobreposições anteriores.

Efeito de **whip pan digital** (borrão direcional curto) sobre os 6 quadros
finais de A e iniciais de B: aplique
`gblur=sigma=25:sigmaV=0:enable='gte(t,DUR-0.2)'` no fim de A e
`gblur=sigma=25:sigmaV=0:enable='lte(t,0.2)'` no início de B
(substitua `DUR` pela duração de A).

### 4.5 Velocidade e speed ramp
- **Câmera lenta real** (origem 120 fps → 30 fps a 25%):
  `setpts=4.0*PTS,fps=30` (sem áudio, ou áudio substituído por SFX).
- **Acelerar 2x:** `setpts=0.5*PTS` + `atempo=2.0`.
- **Speed ramp:** divida em três trechos (normal → lento → normal) com
  `trim`/`setpts` e concatene; corte no ponto de impacto.
- Interpolação de quadros (`minterpolate=fps=60:mi_mode=mci`) só como último
  recurso: gera artefatos em membros cruzados e kimonos.

### 4.6 Punch-in (reenquadramento de 115%)
`scale=iw*1.15:-2,crop=1080:1920` — para variar jump cuts e enfatizar
detalhes. Não ultrapasse ~125% em material 1080p (perda de nitidez).

### 4.7 Sincronização com a batida
Com BPM conhecido e o tempo da primeira batida forte (`t0`, em segundos,
medido ouvindo a trilha):

```python
bpm, t0, duracao = 120, 0.48, 30.0
intervalo = 60.0 / bpm
batidas = [round(t0 + i * intervalo, 3) for i in range(int((duracao - t0) / intervalo) + 1)]
tempos_fortes = batidas[::4]   # início de cada compasso 4/4
print(batidas[:16], tempos_fortes[:4])
```

Ajuste os cortes do roteiro para esses tempos (tolerância ±1 quadro).

---

## 5. Color grading com FFmpeg

Ordem no filtro: correção → equalização → look → secundárias → vinheta.
Sempre aplique a cor nos trechos conformados (ou na montagem), nunca nos
originais. Gere frames de comparação antes/depois (§9.4).

### 5.1 Correção primária

```
eq=brightness=0.02:contrast=1.08:saturation=1.05:gamma=1.0
colortemperature=temperature=5600          # neutraliza/aquece (FFmpeg ≥ 5.0)
colorbalance=rs=0.03:bs=-0.03              # sombras: +vermelho, -azul (aquecer)
curves=master='0/0.02 0.25/0.22 0.75/0.80 1/0.98'   # S suave sem esmagar pretos/estourar brancos
```

### 5.2 Receitas dos looks (ver `direcao.md` §5.3)

```bash
# Tatame Noir
LOOK_NOIR="eq=contrast=1.18:saturation=0.75:gamma=0.95,curves=master='0/0 0.20/0.12 0.60/0.62 1/0.97',vignette=PI/5"
# Golden Family
LOOK_GOLD="colortemperature=temperature=4900,eq=contrast=0.96:saturation=1.08:brightness=0.02,curves=master='0/0.05 0.5/0.52 1/0.98'"
# Clean Pro
LOOK_CLEAN="eq=contrast=1.06:saturation=1.0,unsharp=5:5:0.4"
# Empower
LOOK_EMP="eq=contrast=1.12:saturation=1.02,colorbalance=rs=0.02:bs=0.04:rh=0.03:bh=-0.02,vignette=PI/6"
# Memória
LOOK_MEM="eq=saturation=0.55:contrast=0.92,curves=master='0/0.08 1/0.94',noise=alls=6:allf=t"
```

Uso: `-vf "$LOOK_NOIR,format=yuv420p"`.

### 5.3 Proteger o vermelho GB
Após looks dessaturados, devolva saturação apenas aos vermelhos:
`huesaturation=colors=r:saturation=0.25:strength=1` (FFmpeg ≥ 5.1).
Confira visualmente o escudo/patches contra a referência do brand kit.

### 5.4 LUT 3D (quando fornecida pelo usuário)
`lut3d=file='looks/gb.cube':interp=tetrahedral`
Aplique sobre imagem já corrigida; reduza a intensidade misturando com o
original quando necessário:

```
[0:v]split[a][b];[b]lut3d=file='looks/gb.cube'[l];[a][l]blend=all_mode=normal:all_opacity=0.7
```

### 5.5 HDR (HLG/PQ) → SDR BT.709
Requer filtro `zscale` (libzimg):

```
zscale=t=linear:npl=100,format=gbrpf32le,zscale=p=bt709,tonemap=tonemap=hable:desat=0,zscale=t=bt709:m=bt709:r=tv,format=yuv420p
```

O `zscale` depende das marcações de cor da origem. Se o arquivo HDR chegar
sem `color_transfer`/`color_primaries` (erro "no path between colorspaces"),
marque-o antes da cadeia com
`setparams=color_trc=arib-std-b67:color_primaries=bt2020:colorspace=bt2020nc:range=tv`
(use `smpte2084` no lugar de `arib-std-b67` para PQ) — somente se tiver
certeza de que a origem é HDR.

Sem `zscale`: informe a limitação; alternativa é exportar do celular em
SDR ("Mais compatível") ou usar `libplacebo` se houver GPU. Nunca entregue
HDR sem tonemapping — cores lavadas ou estouradas na plataforma.

### 5.6 Marcação de cor na saída
Sempre: `-colorspace bt709 -color_primaries bt709 -color_trc bt709 -color_range tv`.

### 5.7 Critérios de aprovação de cor
- Nenhum clipping relevante nos kimonos brancos (verificar com
  `signalstats`, §9.3: YMAX ≤ 235 na maior parte dos quadros).
- Pretos com detalhe (YMIN próximo de 16, não esmagados por longos trechos).
- Pele natural em todos os planos; planos da mesma cena sem saltos de
  temperatura perceptíveis.
- Vermelho da marca fiel à referência.

---

## 6. Áudio: limpeza, mixagem, ducking e loudness

### 6.1 Limpeza da voz

```
highpass=f=80,lowpass=f=14000,afftdn=nf=-25,equalizer=f=250:t=q:w=1.2:g=-3,equalizer=f=3500:t=q:w=1.0:g=3,acompressor=threshold=-20dB:ratio=3:attack=5:release=120:makeup=3
```

- `afftdn`: redução de ruído; valores agressivos deixam a voz "metálica".
- Corte de 250 Hz reduz embolo de sala; +3 dB em 3,5 kHz dá inteligibilidade.
- `deesser` (FFmpeg ≥ 4.4) se houver sibilância.

### 6.2 Ducking (música abaixa quando há voz)

```bash
ffmpeg -i video_com_voz.mp4 -i 03_audio/trilha.wav -filter_complex \
"[0:a]aresample=48000,asplit=2[voz][sc]; \
 [1:a]aresample=48000,volume=0.7[mus]; \
 [mus][sc]sidechaincompress=threshold=0.03:ratio=10:attack=15:release=350[musd]; \
 [voz][musd]amix=inputs=2:duration=first:normalize=0[mix]" \
  -map 0:v -map "[mix]" -c:v copy -c:a pcm_s24le 02_trabalho/mix.mov
```

`normalize=0` exige FFmpeg ≥ 5.0 (em versões antigas, compense com `volume`).

### 6.3 Efeitos sonoros posicionados
Atrasar um SFX para 2,35 s e somar:

```
[2:a]adelay=2350|2350,volume=0.9[sfx];[mix][sfx]amix=inputs=2:duration=first:normalize=0[mix2]
```

Fade de música no final: `afade=t=out:st=27.5:d=1.5`.

### 6.4 Loudness final (dois passes, EBU R128)
Passe 1 — medir:

```bash
ffmpeg -hide_banner -i 02_trabalho/mix.mov -af loudnorm=I=-14:TP=-1.5:LRA=11:print_format=json -f null - 2>&1 | tail -12
```

Passe 2 — aplicar com os valores medidos (`input_i`, `input_tp`,
`input_lra`, `input_thresh`, `target_offset`):

```
loudnorm=I=-14:TP=-1.5:LRA=11:measured_I=<input_i>:measured_TP=<input_tp>:measured_LRA=<input_lra>:measured_thresh=<input_thresh>:offset=<target_offset>:linear=true
```

Alvo: -14 LUFS integrado (±1), true peak ≤ -1,0 dBTP. Se `linear=true` não
puder ser aplicado (faixa dinâmica excessiva), comprima antes e repita.
O `loudnorm` reamostra internamente para 192 kHz: sempre adicione
`aresample=48000` depois.

---

## 7. Legendas animadas (ASS)

### 7.1 Requisitos
Filtro `subtitles` ou `ass` (libass). Fontes do brand kit em uma pasta
`fonts/`, passada com `fontsdir`. Verifique se a fonte foi realmente usada
(o log do libass avisa substituições: `fontselect: ... -> ...`).

### 7.2 Modelo de arquivo `.ass` (1080x1920)

```ini
[Script Info]
ScriptType: v4.00+
PlayResX: 1080
PlayResY: 1920
WrapStyle: 0
ScaledBorderAndShadow: yes

[V4+ Styles]
Format: Name, Fontname, Fontsize, PrimaryColour, SecondaryColour, OutlineColour, BackColour, Bold, Italic, Underline, StrikeOut, ScaleX, ScaleY, Spacing, Angle, BorderStyle, Outline, Shadow, Alignment, MarginL, MarginR, MarginV, Encoding
Style: Fala,Montserrat ExtraBold,66,&H00FFFFFF,&H000000FF,&H00000000,&H64000000,-1,0,0,0,100,100,0,0,1,5,2,2,90,160,560,1
Style: Titulo,Bebas Neue,104,&H00FFFFFF,&H000000FF,&H00000000,&H64000000,-1,0,0,0,100,100,1,0,1,6,3,8,90,160,330,1
Style: Destaque,Montserrat ExtraBold,66,&H00FFFFFF,&H000000FF,&H000000C8,&H00000000,-1,0,0,0,100,100,0,0,3,14,0,2,90,160,560,1

[Events]
Format: Layer, Start, End, Style, Name, MarginL, MarginR, MarginV, Effect, Text
Dialogue: 0,0:00:00.00,0:00:01.40,Titulo,,0,0,0,,{\fscx80\fscy80\t(0,100,\fscx100\fscy100)}JIU-JITSU NÃO É FORÇA
Dialogue: 0,0:00:01.40,0:00:02.60,Fala,,0,0,0,,{\fscx85\fscy85\t(0,90,\fscx100\fscy100)}é {\c&H2A1ED6&}técnica.
Dialogue: 0,0:00:02.60,0:00:04.20,Fala,,0,0,0,,{\k35}três {\k30}escapes {\k40}que {\k55}salvam
```

Notas:
- Cores ASS são `&HAABBGGRR` (alfa, azul, verde, vermelho). O vermelho
  `&H2A1ED6&` acima é **provisório**: substitua pelo vermelho oficial do
  brand kit convertido para BGR.
- `Alignment` 2 = centro inferior; 8 = centro superior. `MarginV` 560 coloca
  a fala por volta de y ≈ 1360 (acima da interface); `MarginR` 160 evita os
  botões da direita.
- `BorderStyle 3` desenha caixa atrás do texto (estilo "Destaque").
- `\k` (centésimos de segundo) faz destaque palavra a palavra com
  `SecondaryColour` → `PrimaryColour`.
- `\t(0,100,...)` cria o "pop" de entrada em 100 ms.

### 7.3 Aplicação

```bash
ffmpeg -hide_banner -i 02_trabalho/montagem_cor.mp4 \
  -vf "subtitles=04_legendas/legendas.ass:fontsdir=fonts" \
  -c:v libx264 -crf 14 -c:a copy 02_trabalho/montagem_leg.mp4
```

Em caminhos com `:` ou `'`, escape-os no filtro ou trabalhe com caminhos
relativos simples.

### 7.4 Sincronização
- Cada legenda: mínimo 0,6 s; ideal 1,0–1,8 s para 2–4 palavras.
- Entrada até 1 quadro antes do início da fala; saída no fim da palavra.
- Se houver transcrição automática disponível (ex.: Whisper já instalado),
  use-a como base e **revise 100%**; caso contrário, transcreva ouvindo.

### 7.5 Logotipo e elementos de marca

```
[0:v][1:v]overlay=x=W-w-60:y=180:enable='gte(t,0)'        # logo PNG com transparência, já em escala
```

Escale o PNG previamente (ex.: 120 px de largura) com `scale=120:-1`
e preserve a transparência (`format=rgba`).

---

## 8. Python e OpenCV: cenas, desfoque de rostos, enquadramento

Só execute se `cv2` estiver disponível (§1).

**Compatibilidade de versão (verificado em testes):** os scripts abaixo
foram validados com **OpenCV 4.x** (`opencv-python-headless` 4.14). O
**OpenCV 5.x removeu o `CascadeClassifier`** e os arquivos Haar de
`cv2.data`; nele, §8.2 falha. Antes de rodar §8.2, confira:

```bash
python3 -c "import cv2; print(cv2.__version__, hasattr(cv2, 'CascadeClassifier'))"
```

Se retornar `False`: use OpenCV 4.x (instalação somente com autorização)
ou o detector `cv2.FaceDetectorYN` (YuNet), que exige o arquivo de modelo
ONNX `face_detection_yunet_*.onnx` do repositório *opencv_zoo* — o download
do modelo também requer autorização. §8.1 e §8.3 funcionam nas duas versões.

Os scripts leem dos originais
ou de trabalho e escrevem **somente** em `02_trabalho/` ou `06_qc/`.
OpenCV não grava áudio: re-multiplexe o áudio com FFmpeg ao final.

### 8.1 Detecção de cenas
Sem OpenCV (FFmpeg):

```bash
ffmpeg -hide_banner -i "$IN" -vf "select='gt(scene,0.30)',showinfo" -f null - 2>&1 \
  | grep -o 'pts_time:[0-9.]*' | cut -d: -f2 > 06_qc/cortes.txt
```

Com OpenCV (diferença de histograma HSV):

```python
import cv2, sys
cap = cv2.VideoCapture(sys.argv[1]); fps = cap.get(cv2.CAP_PROP_FPS) or 30
prev, i, cortes = None, 0, []
while True:
    ok, frame = cap.read()
    if not ok: break
    hsv = cv2.cvtColor(cv2.resize(frame, (160, 90)), cv2.COLOR_BGR2HSV)
    h = cv2.calcHist([hsv], [0, 1], None, [32, 32], [0, 180, 0, 256]); cv2.normalize(h, h)
    if prev is not None and cv2.compareHist(prev, h, cv2.HISTCMP_BHATTACHARYYA) > 0.45:
        cortes.append(round(i / fps, 3))
    prev, i = h, i + 1
print(cortes)
```

### 8.2 Desfoque de rostos (proteção infantil)

```python
"""Uso: python3 -I blur_faces.py ENTRADA SAIDA_SEM_AUDIO [margem=0.35]
Desfoca rostos detectados e mantém o desfoque por alguns quadros quando a
detecção falha (persistência), para evitar 'piscadas' de rosto exposto."""
import cv2, sys
src, dst = sys.argv[1], sys.argv[2]
margem = float(sys.argv[3]) if len(sys.argv) > 3 else 0.35
frontal = cv2.CascadeClassifier(cv2.data.haarcascades + "haarcascade_frontalface_default.xml")
perfil = cv2.CascadeClassifier(cv2.data.haarcascades + "haarcascade_profileface.xml")
cap = cv2.VideoCapture(src)
fps = cap.get(cv2.CAP_PROP_FPS) or 30
w, h = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH)), int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))
out = cv2.VideoWriter(dst, cv2.VideoWriter_fourcc(*"mp4v"), fps, (w, h))
memoria, PERSIST = [], int(fps * 0.5)   # mantém caixas por 0,5 s
while True:
    ok, f = cap.read()
    if not ok: break
    g = cv2.equalizeHist(cv2.cvtColor(f, cv2.COLOR_BGR2GRAY))
    caixas = list(frontal.detectMultiScale(g, 1.1, 4, minSize=(24, 24)))
    caixas += list(perfil.detectMultiScale(g, 1.1, 4, minSize=(24, 24)))
    caixas += [(w - x - cw, y, cw, ch) for (x, y, cw, ch) in perfil.detectMultiScale(cv2.flip(g, 1), 1.1, 4, minSize=(24, 24))]
    memoria = [(b, PERSIST) for b in caixas] + [(b, n - 1) for b, n in memoria if n > 1]
    for (x, y, cw, ch), _ in memoria:
        mx, my = int(cw * margem), int(ch * margem)
        x0, y0, x1, y1 = max(0, x - mx), max(0, y - my), min(w, x + cw + mx), min(h, y + ch + my)
        roi = f[y0:y1, x0:x1]
        if roi.size:
            k = max(31, (min(roi.shape[:2]) // 2) | 1)
            f[y0:y1, x0:x1] = cv2.GaussianBlur(roi, (k, k), 0)
    out.write(f)
cap.release(); out.release()
```

Re-multiplexar áudio e recodificar corretamente:

```bash
ffmpeg -i 02_trabalho/blur_sem_audio.mp4 -i "$IN" -map 0:v -map 1:a? \
  -c:v libx264 -crf 14 -pix_fmt yuv420p -c:a copy 02_trabalho/blur.mp4
```

Não use `-shortest` aqui: em testes ele descartou o último quadro do vídeo.
Confira que a contagem de quadros é igual à da origem:
`ffprobe -v error -count_frames -select_streams v:0 -show_entries stream=nb_read_frames -of csv=p=0 <arquivo>`.

**Revisão manual obrigatória:** gere folha de contato a 4 quadros/s do
resultado (§9.4) e verifique quadro a quadro os trechos com menores.
Haar cascades falham em perfil extremo, rosto parcialmente coberto,
movimento rápido, cabeça invertida (comum no Jiu-Jitsu) e baixa luz. Se
houver qualquer quadro com rosto exposto sem autorização, aplique desfoque
manual por região (§8.4) ou descarte o trecho.

### 8.3 Enquadramento inteligente (horizontal → vertical)
Estratégia: calcular o centro de massa do movimento por quadro (diferença
entre quadros), suavizar a trajetória e gerar a expressão de `crop`.

```python
import cv2, sys, numpy as np
cap = cv2.VideoCapture(sys.argv[1]); fps = cap.get(cv2.CAP_PROP_FPS) or 30
W, H = int(cap.get(3)), int(cap.get(4)); cw = int(H * 9 / 16)
xs, prev = [], None
while True:
    ok, f = cap.read()
    if not ok: break
    g = cv2.GaussianBlur(cv2.cvtColor(f, cv2.COLOR_BGR2GRAY), (21, 21), 0)
    if prev is None: xs.append(W / 2)
    else:
        d = cv2.threshold(cv2.absdiff(prev, g), 18, 255, cv2.THRESH_BINARY)[1]
        m = cv2.moments(d); xs.append(m["m10"] / m["m00"] if m["m00"] > 0 else xs[-1])
    prev = g
xs = np.convolve(np.array(xs), np.ones(int(fps)) / int(fps), mode="same")   # suavização de 1 s
xs = np.clip(xs - cw / 2, 0, W - cw).astype(int)
# Amostra a cada 0,5 s para um arquivo de comandos do sendcmd
with open(sys.argv[2], "w") as fh:
    for i in range(0, len(xs), max(1, int(fps / 2))):
        fh.write(f"{i / fps:.3f} crop x {xs[i]};\n")
print("largura do recorte:", cw)
```

Aplicação (o recorte é feito na resolução de origem e depois escalado):

```bash
ffmpeg -i "$IN" -vf "sendcmd=f=06_qc/crop_cmds.txt,crop=w=<cw>:h=ih:x=0:y=0,scale=1080:1920:flags=lanczos,setsar=1,fps=30" \
  -c:v libx264 -crf 14 -c:a copy 02_trabalho/reframe.mp4
```

Revise o resultado: em rola com dois atletas, o centro de movimento pode
"pular" entre eles; aumente a suavização ou use o recorte manual (§4.2b/c).

### 8.4 Desfoque manual por região (somente FFmpeg)
Desfocar a região x=300, y=500, 260x260 entre 3,2 s e 5,0 s:

```bash
ffmpeg -i "$IN" -filter_complex \
"[0:v]split[base][r];[r]crop=260:260:300:500,gblur=sigma=30[b]; \
 [base][b]overlay=300:500:enable='between(t,3.2,5.0)'[v]" \
  -map "[v]" -map 0:a? -c:v libx264 -crf 14 -c:a copy 02_trabalho/blur_manual.mp4
```

Para alvos em movimento, divida em intervalos curtos com posições
diferentes, ou use `crop` com expressões em `t`.

---

## 9. Exportação e controle de qualidade (QC)

### 9.1 Exportação final

```bash
ffmpeg -hide_banner -i 02_trabalho/montagem_final.mov \
  -vf "scale=1080:1920:flags=lanczos,setsar=1,fps=30,format=yuv420p" \
  -af "aresample=48000" \
  -c:v libx264 -preset slow -crf 18 -profile:v high -level:v 4.2 \
  -g 60 -keyint_min 30 -bf 2 -maxrate 15M -bufsize 30M \
  -colorspace bt709 -color_primaries bt709 -color_trc bt709 -color_range tv \
  -c:a aac -b:a 256k -ar 48000 -ac 2 \
  -map_metadata -1 -map_chapters -1 -metadata title="<título>" \
  -movflags +faststart \
  05_exports/<tema>_v02_final.mp4
```

- `-map_metadata -1` remove metadados de origem (inclui localização GPS).
- 60 fps: troque `fps=60`, `-g 120`, mantenha nível 4.2.
- Versão sem música (para áudio da plataforma): exporte também
  `<tema>_v02_final_semmusica.mp4` com apenas voz/foley.

### 9.2 Verificação de especificação

```bash
F=05_exports/<tema>_v02_final.mp4
ffprobe -v error -show_entries stream=codec_name,profile,width,height,sample_aspect_ratio,pix_fmt,r_frame_rate,avg_frame_rate,color_space,color_transfer,color_primaries,sample_rate,channels,bit_rate:format=duration,bit_rate,format_name -of default=nw=1 "$F"
ffprobe -v error -show_entries format_tags -of default=nw=1 "$F"   # não deve conter location/GPS
```

| Verificação | Esperado |
|---|---|
| Vídeo | h264 / High, 1080x1920, SAR 1:1, yuv420p |
| fps | `r_frame_rate` = `avg_frame_rate` = 30/1 (ou 60/1) |
| Cor | bt709 / bt709 / bt709 |
| Áudio | aac, 48000 Hz, 2 canais |
| Duração | conforme roteiro (±0,1 s) |
| Metadados | sem `location`, `com.apple.quicktime.location*`, nomes de dispositivo |
| faststart | átomo `moov` antes de `mdat` (verifique com `ffprobe -v trace "$F" 2>&1 \| grep -m2 -o "type:'\(moov\|mdat\)'"`) |

### 9.3 Medições de áudio e imagem

```bash
# Loudness integrado e true peak
ffmpeg -hide_banner -nostats -i "$F" -af ebur128=peak=true -f null - 2>&1 | grep -A12 "Summary:"
# Silêncios indesejados (> 1 s)
ffmpeg -hide_banner -nostats -i "$F" -af silencedetect=n=-50dB:d=1 -f null - 2>&1 | grep silence_
# Quadros pretos (> 0,1 s) — inclusive no início/fim
ffmpeg -hide_banner -nostats -i "$F" -vf blackdetect=d=0.1:pix_th=0.10 -an -f null - 2>&1 | grep blackdetect
# Imagem congelada (> 0,5 s) — exceto congelamentos intencionais
ffmpeg -hide_banner -nostats -i "$F" -vf freezedetect=n=-60dB:d=0.5 -an -f null - 2>&1 | grep freeze_
# Níveis de luminância (clipping)
ffmpeg -hide_banner -nostats -i "$F" -vf "signalstats,metadata=print:key=lavfi.signalstats.YMAX:file=06_qc/ymax.txt" -an -f null -
```

| Medição | Critério de aprovação |
|---|---|
| Loudness integrado | -14 LUFS ± 1 |
| True peak | ≤ -1,0 dBTP |
| Silêncio | nenhum > 1 s não intencional |
| Preto | nenhum no início; nenhum > 0,1 s não intencional |
| Congelamento | nenhum não intencional |
| Clipping | YMAX > 235 em no máximo trechos curtos e não em kimonos/rostos |

### 9.4 Inspeção visual

```bash
# Folha de contato: 1 quadro/s, 6 colunas
ffmpeg -hide_banner -i "$F" -vf "fps=1,scale=270:-2,tile=6x6:padding=4:color=black" -frames:v 1 06_qc/contato.png
# Quadros do gancho (0–3 s) a 4 quadros/s
ffmpeg -hide_banner -i "$F" -t 3 -vf "fps=4,scale=360:-2,tile=6x2" -frames:v 1 06_qc/gancho.png
# Prévia com zonas seguras marcadas (NÃO publicar este arquivo)
ffmpeg -hide_banner -i "$F" -vf "drawbox=x=0:y=0:w=iw:h=250:color=red@0.35:t=fill,drawbox=x=0:y=ih-400:w=iw:h=400:color=red@0.35:t=fill,drawbox=x=iw-140:y=0:w=140:h=ih:color=red@0.25:t=fill" \
  -c:v libx264 -crf 23 -c:a copy 06_qc/previa_zonas_seguras.mp4
```

Observe as imagens geradas (abrindo-as) e confirme: gancho forte no
quadro 1, legendas fora das áreas vermelhas, nenhum rosto infantil sem
autorização, logotipo correto, nenhum quadro estranho (flash, frame preto,
erro de legenda).

### 9.5 Checklist final de QC técnico

```
[ ] Integridade dos originais conferida (sha256sum -c): OK
[ ] Especificação (§9.2) conforme
[ ] Loudness -14 ± 1 LUFS; true peak ≤ -1 dBTP
[ ] Sem pretos/silêncios/congelamentos indesejados
[ ] Sem clipping relevante; pele natural; vermelho GB fiel
[ ] Legendas: sincronia, ortografia, zona segura, fonte correta
[ ] Áudio conferido em fone e em alto-falante de celular
[ ] Assistido do início ao fim em tela de celular (ou prévia 9:16 em tamanho real)
[ ] Protocolo de Proteção Infantil: aprovado
[ ] Metadados de localização removidos
[ ] Arquivo final nomeado e versionado em 05_exports/
```

### 9.6 Relatório automático de QC (Python, sem dependências externas)

```python
import json, subprocess, sys, re
f = sys.argv[1]
p = json.loads(subprocess.run(["ffprobe", "-v", "error", "-show_streams", "-show_format", "-of", "json", f],
                              capture_output=True, text=True, check=True).stdout)
v = next(s for s in p["streams"] if s["codec_type"] == "video")
a = next((s for s in p["streams"] if s["codec_type"] == "audio"), None)
r = subprocess.run(["ffmpeg", "-hide_banner", "-nostats", "-i", f, "-af", "ebur128=peak=true", "-f", "null", "-"],
                   capture_output=True, text=True).stderr
lufs = float(re.findall(r"I:\s+(-?[\d.]+) LUFS", r)[-1]) if a else None
tp = float(re.findall(r"Peak:\s+(-?[\d.]+) dBFS", r)[-1]) if a else None
checks = {
    "resolucao_1080x1920": (v["width"], v["height"]) == (1080, 1920),
    "h264_yuv420p": v["codec_name"] == "h264" and v["pix_fmt"] == "yuv420p",
    "fps_constante": v["r_frame_rate"] == v["avg_frame_rate"] and v["r_frame_rate"] in ("30/1", "60/1"),
    "bt709": v.get("color_primaries") == "bt709" and v.get("color_transfer") == "bt709",
    "audio_aac_48k_estereo": bool(a) and a["codec_name"] == "aac" and a["sample_rate"] == "48000" and a["channels"] == 2,
    "loudness_-14_+-1": lufs is not None and abs(lufs + 14) <= 1,
    "true_peak_<=-1": tp is not None and tp <= -1.0,
    "sem_localizacao": not any("location" in k.lower() for k in p["format"].get("tags", {})),
}
print(json.dumps({"arquivo": f, "lufs": lufs, "true_peak": tp,
                  "duracao": float(p["format"]["duration"]), "checks": checks,
                  "aprovado": all(checks.values())}, indent=2, ensure_ascii=False))
```

Execute com `python3 -I qc.py 05_exports/<arquivo>.mp4 > 06_qc/qc.json`
(salve o script fora da pasta de mídia).

---

## 10. Solução de problemas

| Sintoma | Causa provável | Solução |
|---|---|---|
| Vídeo deitado após conversão | Rotação ignorada (`-noautorotate`) ou metadado incorreto | Remova `-noautorotate`; se necessário, `transpose=1` (90° horário) ou `transpose=2`. |
| Áudio dessincronizado | VFR do celular, corte com `-c copy` | Conformar com `fps=30` e recodificar; usar `aresample=async=1`. |
| Cores lavadas/acinzentadas | HDR sem tonemapping ou faixa full/limited trocada | §5.5; conferir `color_range` e forçar `-color_range tv`. |
| Banding no céu/parede | 8 bits + compressão | Graduar em 10 bits, adicionar ruído leve (`noise=alls=2:allf=t`), CRF menor. |
| Legenda com fonte errada | Fonte não encontrada pelo libass | `fontsdir`, nome exato da família, conferir log. |
| `xfade` falha | Clipes com resolução/fps/timebase diferentes | Conformar todos (§4.1) e usar `settb=AVTB` se necessário. |
| Plataforma reduz qualidade | Bitrate/perfil fora do esperado ou upload por dados móveis | Exportar conforme §9.1; enviar via Wi-Fi; ativar "upload em alta qualidade" no app. |
| Picotes no áudio após loudnorm | True peak alto/limitação excessiva | Comprimir antes, alvo TP -1,5 e reamostrar para 48 kHz. |
| Rosto infantil "pisca" no desfoque | Falha de detecção em quadros isolados | Aumentar persistência/margem (§8.2) ou desfoque manual (§8.4). |
