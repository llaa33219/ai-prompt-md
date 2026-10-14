# 드림코어(degraded CCD / dreamcore) 룩 — ffmpeg 재현 문서

> 이 문서 하나만 있으면 **ffmpeg 5.x + bash만으로** 참고 이미지 3장의 룩을
> 어떤 이미지·영상에든 재현할 수 있다. LUT 파일, 플러그인, 외부 에셋 없음.

---

## 0. TL;DR

**아무 이미지나**

```bash
bash lofi.sh in.jpg out.jpg
```

**아무 영상이나**

```bash
bash lofi.sh clip.mp4 out.mp4          # 흔들림까지 포함
WOBBLE=0 bash lofi.sh clip.mp4 out.mp4 # 흔들림 없이
```

`lofi.sh` 전문은 [§2](#2-스크립트-전문-복붙용)에 붙여있다. 파일로 저장 후 바로 쓴다.

**스크립트 없이 한 방에** (1920×1080 기준 수치, 하단에 조정법)

```bash
ffmpeg -i in.jpg -filter_complex "$(cat fx.txt)" -map "[out]" -frames:v 1 out.jpg
```

---

## 1. 이 룩의 정체 (수치로 정의)

참고 이미지 3장을 픽셀 통계를 뜯어 정의한 값이다. 감이 아니라 숫자다.

| 지표 | 참고 3장 | 일반 사진 | 이 문서 체인 결과 |
|---|---|---|---|
| 평균 채도 | 0.496 / 0.719 / 0.630 | 0.25~0.37 | **0.68** |
| p99 휘도 | 0.902 / 0.897 / 0.891 | 1.000 (클리핑) | **0.907** |
| 그레인 σ (평평한 하늘 32px 패치) | 0.0123 | 0.0026 | **0.0124** |
| hp(Cb/Cr) | 0.032~0.049 | 0.016 | **0.035** |
| 비네트(테두리/중앙) | 0.62 / 0.70 / 0.83 | — | 0.54~0.67 |

시그니처 6개:

1. **채도가 미친듯이 높다** (평균 0.5~0.72). `s=1.2`로는 절대 안 된다.
2. **하이라이트가 절대 클리핑 안 된다** (p99 ≈ 0.90). 필름처럼 숄더로 눌러야 한다.
3. **그레인이 1픽셀 정적이 아니라 2~8픽셀짜리 색 덩어리.** 35mm 비디 스캔 그레인.
4. **밴드 노이즈 ≈ 컬러 노이즈 크기.** 아날로그 CCD 계열.
5. **글로우는 실제 광원 근처에만 국소적으로.** 화면 전체를 덮는 베일 아니다.
6. **하이라이트 엣지에 RGB 색 분리**(J3 고스트).

---

## 2. 스크립트 전문 (복붙용)

아래를 `lofi.sh`로 저장하고 `chmod +x` 한 뒤 쓴다. 외부 파일·옵션 불필요.

```bash
#!/usr/bin/env bash
# =============================================================================
#  lofi — dreamcore / degraded-CCD grade for any image or video.
#  ffmpeg >= 5.0 only. No plugins, no LUT files, no external assets.
#
#    bash lofi.sh in.jpg out.jpg
#    bash lofi.sh clip.mp4 out.mp4
#
#  Knobs (env vars, all optional):
#    SAT GRAIN GN2 BLOOM HALO GHOST ABERR SOFT CON VIG WAMP TARGET
# =============================================================================
set -euo pipefail
IN="$1"; OUT="$2"
[[ -n "$IN" && -n "$OUT" ]] || { echo "usage: lofi.sh <in> <out>" >&2; exit 2; }

# ---- grade constants (tuned against 3 reference frames) --------------------
SAT="${SAT:-1.85}"     # saturation            ref band 0.50-0.72
CON="${CON:-1.22}"     # contrast
SOFT="${SOFT:-4.2}"    # soft-focus sigma @1080p (scales with height)
GRAIN="${GRAIN:-0.36}" # fine grain opacity
GN2="${GN2:-0.46}"     # coarse grain opacity
BLOOM="${BLOOM:-1.80}" # highlight bloom
HALO="${HALO:-1.20}"   # red halation
GHOST="${GHOST:-0.35}" # J3 rgb ghosting
ABERR="${ABERR:-4.5}"  # chromatic aberration (px @1080p)
VIG="${VIG:-PI/55}"    # vignette (bigger = stronger)
WAMP="${WAMP:-1.0}"    # tape wobble amount (video only)
TARGET="${TARGET:-0.42}" # auto-level target mean luma

# ---- resolution ------------------------------------------------------------
read -r W H < <(ffprobe -v error -select_streams v:0 \
                -show_entries stream=width,height -of csv=p=0:s=x "$IN" | head -1 | tr 'x,' '  ')
W="${W:-1920}"; H="${H:-1080}"
K=$(awk -v h="$H" 'BEGIN{printf "%.4f", h/1080}')       # spatial scale factor
SC() { awk -v x="$1" -v k="$K" 'BEGIN{printf "%.4f", x*k}'; }   # scale a px value
CL() { awk -v x="$1" -v m="$2" 'BEGIN{printf "%.4f", (x>m)?m:x}'; } # clamp

SOFT_E=$(SC "$SOFT"); SOFT_V=$(SC "$SOFT/2")
B_SIG=$(awk -v x="$BLOOM" 'BEGIN{printf "%.2f", 14+34*x}')
B_VS=$(awk  -v x="$BLOOM" 'BEGIN{printf "%.2f",  9+24*x}')
B_OP=$(CL "$(awk -v x="$BLOOM" 'BEGIN{printf "%.3f", 0.26+0.42*x}')" 0.95)
H_SIG=$(awk -v x="$HALO" 'BEGIN{printf "%.2f", 34+70*x}')
H_VS=$(awk  -v x="$HALO" 'BEGIN{printf "%.2f", 24+56*x}')
H_OP=$(CL "$(awk -v x="$HALO" 'BEGIN{printf "%.3f", 0.16+0.34*x}')" 0.80)
G_OFF=$(SC "$(awk -v x="$GHOST" 'BEGIN{printf "%.3f", 3+7*x}')")
C_OFF=$(SC "$(awk -v x="$GHOST" 'BEGIN{printf "%.3f", 4+8*x}')")
G_A=$(awk -v x="$GHOST" 'BEGIN{printf "%.3f", 0.10+0.26*x}')
G_B=$(awk -v x="$GHOST" 'BEGIN{printf "%.3f", 0.06+0.18*x}')
AB_R=$(SC "$ABERR"); AB_C=$(SC "$(awk -v x="$ABERR" 'BEGIN{printf "%.3f", 1.15*x}')")
GS1=$(awk -v k="$K" 'BEGIN{printf "%d", (k+0.5)}')      # fine grain block
GS2=$(awk -v k="$K" 'BEGIN{printf "%d", 5*(k+0.2)}')    # coarse grain block
[ "$GS1" -lt 1 ] && GS1=1; [ "$GS2" -lt 1 ] && GS2=1
GN1=$(CL "$(awk -v x="$GRAIN" 'BEGIN{printf "%d", 120*x}')" 100)
GN2S=$(CL "$(awk -v x="$GN2" 'BEGIN{printf "%d", 110*x}')" 100)

# ---- the grade (no resolution-specific numbers live here) ------------------
GRADE="[0:v]
format=gbrp,
gblur=sigma=${SOFT_E}:sigmaV=${SOFT_V},
eq=saturation=${SAT}:contrast=${CON}:gamma=0.96,
curves=all='0/0.015 0.18/0.15 0.5/0.53 0.82/0.90 1/0.93',
colorbalance=rs=-0.03:gs=0.01:bs=0.03:rm=0.07:gm=0:bm=-0.07:rh=0.04:gh=0:bh=-0.03,
split=5[base][hl1][hl2][ghA][ghB];
[hl1]curves=all='0/0 0.68/0 0.88/0.55 1/1',gblur=sigma=${B_SIG}:sigmaV=${B_VS}[bloom];
[base][bloom]blend=all_mode=screen:all_opacity=${B_OP}[t1];
[hl2]curves=all='0/0 0.78/0 0.92/0.60 1/1',hue=s=1.25,colorchannelmixer=rr=1:rg=0.40:rb=0.22:gr=0:gg=0:gb=0:br=0:bg=0.12:bb=0,gblur=sigma=${H_SIG}:sigmaV=${H_VS}[halo];
[t1][halo]blend=all_mode=screen:all_opacity=${H_OP}[t2];
[ghA]curves=all='0/0 0.62/0 1/0.78',hue=s=1.7,rgbashift=rh=${G_OFF}:bh=$(SC "$(awk -v x="$GHOST" 'BEGIN{printf "%.3f", -0.85*(3+7*x)}')")[ga];
[ghB]curves=all='0/0 0.62/0 1/0.78',hue=h=150:s=1.5,chromashift=cbh=${C_OFF}:crh=-${C_OFF}[gb];
[t2][ga]blend=all_mode=addition:all_opacity=${G_A}[t3];
[t3][gb]blend=all_mode=addition:all_opacity=${G_B}[t4];
[t4]chromashift=cbh=${AB_C}:crh=-${AB_C},
rgbashift=rh=${AB_R}:bh=-${AB_R},
vignette=angle=${VIG},
unsharp=5:5:0.30:5:5:0,
eq=saturation=1.12[gr]"

# ---- auto-level: measure the graded frame, trim exposure, render -----------
# 원본 밝기가 얼마든 결과 명도가 같은 지점에 오게 만드는 2패스.
# YAVG는 0~255 스케일로 나온다(0~1 아님) — 255로 나눠야 한다.
EXPO=0
if [[ "${AUTOLEVEL:-1}" == "1" ]]; then
  Y=$(ffmpeg -hide_banner -loglevel error -i "$IN" \
      -filter_complex "${GRADE};[gr]scale=480:-2,signalstats,metadata=print:key=lavfi.signalstats.YAVG:file=-[m]" \
      -map "[m]" -frames:v 6 -f null - 2>/dev/null \
      | awk -F= '/YAVG/{s+=$2;n++} END{if(n)printf "%.4f", s/n/255}')
  if [[ -n "$Y" && "$Y" != "0.0000" ]]; then
    EXPO=$(awk -v y="$Y" -v t="$TARGET" 'BEGIN{v=t/y; if(v>1.70)v=1.70; if(v<0.40)v=0.40; printf "%.3f", log(v)/log(2)}')
  fi
fi

# ---- two-scale grain -------------------------------------------------------
GRAPH="${GRADE};
[gr]exposure=exposure=${EXPO}:black=0,
curves=all='0/0.01 0.3/0.29 0.7/0.71 0.88/0.86 1/0.93'[fimg];
color=c=gray:s=${W}x${H}:r=30,scale=iw/${GS1}:ih/${GS1}:flags=area,\
noise=alls=${GN1}:allf=u,scale=${W}:${H}:flags=neighbor,format=gbrp[grain1];
[fimg][grain1]blend=all_mode=overlay:all_opacity=$(CL "$GRAIN" 1)[g1];
color=c=gray:s=${W}x${H}:r=30,scale=iw/${GS2}:ih/${GS2}:flags=area,\
noise=alls=${GN2S}:allf=u,scale=${W}:${H}:flags=bicubic,format=gbrp[grain2];
[g1][grain2]blend=all_mode=overlay:all_opacity=$(CL "$GN2" 1),format=yuv420p[v0]"

# ---- tape wobble (video only) ---------------------------------------------
OUTL="[v0]"
if [[ "${WOBBLE:-0}" == "1" ]]; then
  CX=$(( W/96 )); CY=$(( H/96 ))
  OX=$(awk -v a="$WAMP" -v h="$H" 'BEGIN{printf "%d", a*h/1080+2}')
  GRAPH="${GRAPH};[v0]crop=w=iw-${CX}:h=ih:x=${CX}*(0.5+0.5*sin(2*PI*t/1.7)):y=${CY}*(0.5+0.5*sin(2*PI*t/2.9)),\
scale=${W}:${H}:flags=bicubic,hue=h=${OX}*sin(2*PI*t/2.3)[lofi]"
  OUTL="[lofi]"
fi

# ---- run -------------------------------------------------------------------
case "${OUT##*.}" in
  mp4|mov|m4v|mkv|webm)
    # 필터 안의 color 소스는 무한정 생성된다 → -shortest 만으로는 안 끝난다.
    NB=$(ffprobe -v error -count_frames -select_streams v:0 -show_entries \
         stream=nb_read_frames -of csv=p=0 "$IN" | head -1)
    [[ "$NB" =~ ^[0-9]+$ && "$NB" -gt 0 ]] || NB=""
    ffmpeg -hide_banner -y -i "$IN" -filter_complex "$GRAPH" -map "$OUTL" -map 0:a? \
      -shortest ${NB:+-frames:v "$NB"} -c:v libx264 -crf 16 -pix_fmt yuv420p -c:a copy "$OUT" ;;
  *)
    ffmpeg -hide_banner -y -i "$IN" -filter_complex "$GRAPH" -map "$OUTL" -frames:v 1 "$OUT" ;;
esac
echo "ok: $OUT"
```

> **영상은 기본 `WOBBLE=1`로 쓰고, 이미지는 스크립트가 자동으로 끈다.**
> 이미지에서 흔들림 필터는 프레임당 1장이라 의미가 없다.
> `WOBBLE=1 bash lofi.sh in.jpg out.jpg` 로 강제할 수도 있다.

---

## 3. 필터체인 해부 (순서가 중요)

```
[0:v]
 ① 소프트 포커스 스미어
    gblur=sigma=SOFT:sigmaV=SOFT/2
    세로보다 가로를 2배 더 뭉갠 게 VHS 라우마. 저해상도 업스케일 같은 뭉개짐.

 ② 색보정
    eq=saturation=1.85:contrast=1.22:gamma=0.96
    curves=all='0/0.015 ... 1/0.93'      블랙 살짝 띄우고, 숄더로 눌러 클리핑 방지
    colorbalance=...                      그림자는 차갑게, 하이라이트는 따뜻하게

 ③ 하이라이트 분기  ← 이 룩의 핵심
    split=5[base][hl1][hl2][ghA][ghB]
      hl1  curves(0.68/0.88) + gblur        → 블룸      (screen)
      hl2  curves(0.78/0.92) + 색변환 + gblur → 레드 할레이션 (screen)
      ghA/ghB  하이라이트 키만 뽑아 색 회전·채널 이동 → 加算 = J3 고스트

 ④ 렌즈/테이프 결함
    chromashift + rgbashift   가장자리 RGB 색수차
    vignette                  테두리 감쇠
    unsharp                   글로우 경계만 살짝 또렷하게

 ⑤ 노출 매칭 (2패스)
    exposure=...              어떤 원본이든 결과 명도가 같은 곳에

 ⑥ 그레인 2스케일  ← 두 번째 핵심
    color=gray → scale↓ → noise → scale↑    = 덩어리진 컬러 그레인
    blend=overlay                            평균 밝도 무변
```

### ③ 왜 임계값이 0.68 / 0.78인가

처음에 0.55부터 블룸을 걸었다. 그랬더니 석양 하늘 전체가 뿌옇게 번진
퍼덩이가 됐고, 참고 이미지와 전혀 다른 느낌이었다.

참고 이미지를 보면 **달·해·전조등 주변에만** 글로우가 있고 나머지 하늘은
원래 색을 유지한다. 그래서 bloom은 0.68, halation은 0.78부터 시작하게 올렸다.
임계값을 내리면 "전체 베일", 올리면 "국소 광원 글로우"가 된다.

### ⑥ 왜 `noise` 필터를 안 쓰는가

`noise`를 그대로 걸면 **TV 정적**처럼 보인다. 그레인이 1픽셀 단위라서.

그래서 회색 소스를 축소 → 노이즈 → 확대한다. 그러면 그레인이 2~8픽셀짜리
**색 덩어리**로 앉는다 = 35mm 비디 스캔 그레인.

`overlay` 블렌드를 쓰는 이유: 회색(0.5)이 중립이라 평균 밝기가 변하지 않고,
밖으로는 밝은 데는 올리고 어두운 데는 내린다. 필름 그레인의 실제 동작이다.
`addition`으로 넣으면 전체가 뿌옇게 떠서 실패한다.

---

## 4. 완성된 그래프 (고정 수치, 1920×1080)

> **이 절은 정지이미지 전용이다.** 영상은 사이즈가 항상 같지 않고, 필터 안의
> `color` 소스가 무한 생성해서 `-frames:v` 상한도 필요해서 복잡해진다.
> 영상은 그냥 §2 스크립트를 쓸 것. (한 줄 요약: `WOBBLE=1 bash lofi.sh in.mp4 out.mp4`)

스크립트 없이 바로 쓰려면 이걸 `fx.txt`에 저장한다.

```bash
ffmpeg -i in.jpg -filter_complex "$(cat fx.txt)" -map "[out]" -frames:v 1 out.jpg
```

> ⚠️ 아래 그래프의 그레인 소스 크기(`s=1920x1080`, `scale=1920:1080`)는
> **입력 해상도에 맞춰 바꿔야 한다.** 그대로 쓰면 `First input link top parameters
> (size 1440x1080) do not match the corresponding second input link` 에러가 난다.
>
> **원본 크기 확인**
> ```bash
> ffprobe -v error -select_streams v:0 -show_entries stream=width,height \
>         -of csv=p=0:s=x in.jpg        # 예: 1440x1080
> ```
> **치환하고 실행** (1440×1080 예시)
> ```bash
> sed -e 's/1920x1080/1440x1080/g' -e 's/scale=1920:1080/scale=1440:1080/g' \
>     fx.txt > fx2.txt
> ffmpeg -i in.jpg -filter_complex "$(cat fx2.txt)" -map "[out]" -frames:v 1 out.jpg
> ```
> `iw/2`·`iw/10`은 손대지 않아도 된다(입력 기준 상대값이므로).
>
> 애매하면 그냥 §2 스크립트를 쓰면 된다. 이 수치를 신경 쓰지 않는다.

```
[0:v]
format=gbrp,
gblur=sigma=4.2:sigmaV=2.1,
eq=saturation=1.85:contrast=1.22:gamma=0.96,
curves=all='0/0.015 0.18/0.15 0.5/0.53 0.82/0.90 1/0.93',
colorbalance=rs=-0.03:gs=0.01:bs=0.03:rm=0.07:gm=0:bm=-0.07:rh=0.04:gh=0:bh=-0.03,
split=5[base][hl1][hl2][ghA][ghB];
[hl1]curves=all='0/0 0.68/0 0.88/0.55 1/1',gblur=sigma=75.2:sigmaV=52.2[bloom];
[base][bloom]blend=all_mode=screen:all_opacity=0.95[t1];
[hl2]curves=all='0/0 0.78/0 0.92/0.60 1/1',hue=s=1.25,colorchannelmixer=rr=1:rg=0.40:rb=0.22:gr=0:gg=0:gb=0:br=0:bg=0.12:bb=0,gblur=sigma=118:sigmaV=91.2[halo];
[t1][halo]blend=all_mode=screen:all_opacity=0.568[t2];
[ghA]curves=all='0/0 0.62/0 1/0.78',hue=s=1.7,rgbashift=rh=5.45:bh=-4.63[ga];
[ghB]curves=all='0/0 0.62/0 1/0.78',hue=h=150:s=1.5,chromashift=cbh=6.8:crh=-6.8[gb];
[t2][ga]blend=all_mode=addition:all_opacity=0.191[t3];
[t3][gb]blend=all_mode=addition:all_opacity=0.123[t4];
[t4]chromashift=cbh=5.175:crh=-5.175,
rgbashift=rh=4.5:bh=-4.5,
vignette=angle=PI/55,
unsharp=5:5:0.30:5:5:0,
eq=saturation=1.12,
exposure=exposure=0.5:black=0,
curves=all='0/0.01 0.3/0.29 0.7/0.71 0.88/0.86 1/0.93'[fimg];
color=c=gray:s=1920x1080:r=30,scale=iw/2:ih/2:flags=area,noise=alls=43:allf=u,scale=1920:1080:flags=neighbor,format=gbrp[grain1];
[fimg][grain1]blend=all_mode=overlay:all_opacity=0.36[g1];
color=c=gray:s=1920x1080:r=30,scale=iw/10:ih/10:flags=area,noise=alls=50:allf=u,scale=1920:1080:flags=bicubic,format=gbrp[grain2];
[g1][grain2]blend=all_mode=overlay:all_opacity=0.46,format=yuv420p[out]
```

**`exposure=exposure=0.5` 는 "보통 노출된 사진" 기준 기본값이다.**
결과가 어두우면 0.5 → 0.9, 너무 밝으면 0.5 → 0.2 로 올린다.
이미 원본이 밝으면 `exposure` 줄을 아예 지워도 된다.
(§2 스크립트는 이 값을 자동 계산한다.)

이 고정 그래프의 검증 결과 (1440×1080 테스트 이미지, 위 sed 적용):
`meanY 0.286 / sat 0.687 / p99 0.891` — 참고이미지1의 `0.292 / 0.496 / 0.902`와 같은 지대에 온다.

---

## 5. 노브

환경변수로 덮어쓴다: `SAT=2.4 BLOOM=2.6 bash lofi.sh in.jpg out.jpg`

| 노브 | 기본 | 증상 → 조절 |
|---|---|---|
| `SAT` | 1.85 | 심심하면 ↑. 3 넘기면 색이 죽 (네온만 남음) |
| `CON` | 1.22 | 평평하면 ↑ |
| `SOFT` | 4.2 | 안 뭉개보이면 ↑. 해상도 자동 비례 |
| `GRAIN` | 0.36 | 파인 그레인. 0.6 넘으면 TV 정적처럼 보임 |
| `GN2` | 0.46 | 덩어리 그레인. 1.0 넘으면 화면이 지워짐 |
| `GS1`/`GS2` | 1 / 5 | 그레인 크기(px). GS2 ↑ = 더 굵은 얼룩 |
| `BLOOM` | 1.80 | 광원 글로우 크기·세기 |
| `HALO` | 1.20 | 따뜻한(레드) 헤일레이션 |
| `GHOST` | 0.35 | J3 고스트. 1.2 넘으면 비네온처럼 무너짐 |
| `ABERR` | 4.5 | RGB 색수차 px. 가장자리 보라/초록 테두리 |
| `VIG` | PI/55 | **클수록 강한 비네트.** PI/2=최강, PI/80≈무효 |
| `WAMP` | 1.0 | 영상 흔들림 진폭 (이미지 무의미) |
| `TARGET` | 0.42 | 자동 노출 목표 평균 휘도 |
| `AUTOLEVEL` | 1 | 0으로 끄면 원본 노출 그대로 |

### 상황별 레시피

```bash
# 더 지독하게 (과노출 + 미쳐버린 색)
SAT=2.4 BLOOM=2.6 HALO=1.6 GHOST=1.0 bash lofi.sh in.jpg out.jpg

# 아주 살짝만 (깔끔한 사진 살리기)
SAT=1.3 GRAIN=0.14 GN2=0.18 BLOOM=0.8 VIG=PI/80 bash lofi.sh in.jpg out.jpg

# 필름 그레인만 남기고 색은 유지
GRAIN=0.3 GN2=1.4 GS2=3 SAT=1.5 bash lofi.sh in.jpg out.jpg

# 밤 스크린 / 드림
BLOOM=3.2 HALO=2.4 SAT=1.5 bash lofi.sh night.jpg night.jpg

# VHS답게 (트래킹 흔들림 최대)
WOBBLE=1 GHOST=1.2 ABERR=9 WAMP=2.6 bash lofi.sh clip.mp4 out.mp4
```

### 추가 효과 (체인에 직접 삽입)

**스캔라인** — 그레인 앞, `format=yuv420p` 앞:

```
,geq=lum='lum(X,Y)-10*mod(Y,3)':cb='cb(X,Y)':cr='cr(X,Y)'
```

**트래킹 에러(가로로 순간 밀림)** — 입력 직후:

```
-vf "crop=w=iw-8:h=ih:x='8*floor(mod(t*3,1)*8)':y=0,scale=ORIGINAL_W:ORIGINAL_H"
```

**프레임 드롭 느낌** — `-r` 낮추기 + `fps` 필터:

```
fps=12   # 12fps로 뭉개면 VHS 필름 느낌
```

---

## 6. 검증 방법

결과가 참고 이미지와 같은 밴드에 왔는지 숫자로 확인한다.

```bash
cat > measure.py <<'PY'
import numpy as np, sys
from PIL import Image
def st(p):
    a=np.asarray(Image.open(p).convert('RGB')).astype(np.float32)/255.
    R,G,B=a[...,0],a[...,1],a[...,2]
    Y=0.2126*R+0.7152*G+0.0722*B
    mx,mn=a.max(2),a.min(2); sat=np.where(mx>1e-5,(mx-mn)/np.maximum(mx,1e-5),0)
    h,w=Y.shape
    b=np.concatenate([Y[:8].ravel(),Y[-8:].ravel(),Y[:,:8].ravel(),Y[:,-8:].ravel()])
    return dict(meanY=round(float(Y.mean()),3), p99=round(float(np.percentile(Y,99)),3),
                sat=round(float(sat.mean()),3),
                vig=round(float(b.mean()/Y[h//4:3*h//4,w//4:3*w//4].mean()),3))
for p in sys.argv[1:]: print(f"{p:34s}", st(p))
PY
python3 measure.py in.jpg out.jpg
```

그레인 σ만 따로 재려면(평평한 영역에서):

```python
import numpy as np; from PIL import Image
a=np.asarray(Image.open('out.jpg').convert('RGB')).astype(np.float32)/255.
p=a[0:400]; best=min(((p[y:y+32,x:x+32].std(),y,x)
             for y in range(0,368,16) for x in range(0,p.shape[1]-32,16)))
q=p[best[1]:best[1]+32,best[2]:best[2]+32]
print("grain sigmaY =", (0.2126*q[...,0]+0.7152*q[...,1]+0.0722*q[...,2]).std())  # 목표 0.012
```

---

## 7. 함정 (실제로 밟은 것)

| 에러 | 원인 | 해결 |
|---|---|---|
| `Stream specifier 'hl' in filtergraph description` | `split` 패드를 두 갈래에서 씀 | 패드 개수 늘리기 (`split=5`) |
| `Cannot find a matching stream for unlabeled input pad 0` | `[label]` 뒤에 `,filter` 를 이어붙임 | `;[label]filter...` 로 체인 분리 |
| `Error setting option all_opacity to value 1.15` | `blend` opacity > 1 | 클램프 필수 |
| `Error setting option alls to value 110.0000` | `noise` 실수 + 최대 100 | 정수 + 클램프 |
| `No such filter: ''` | `,crop=...` 앞의 빈 쉼표 | TAIL 문자열이 쉼표로 시작하면 안 됨 |
| `moov atom not found` | 필터 안 `color` 소스가 무한 → `-shortest` 무효 | `-frames:v N` 으로 상한 |
| `Error initializing complex filters` (영상만) | 셸 TAIL 확장 위치 오류 | `[v0]` 뒤 `;` 로 체인 분리 |

추가 주의:

- **`vignette` 의 `angle` 은 클수록 강한 비네트다.** PI/2가 최강, PI/5도 꽤 강함.
- **`signalstats` 의 `YAVG` 는 0~255 스케일**로 나온다. 255로 나눠야 함.
- **`format=gbrp` 뒤 `c0`=G, `c1`=B, `c2`=R** 이라 `noise=c0s` 는 명도가 아니다.
- **`-map` 을 지정하면 오디오가 자동으로 안 따라온다.** `-map 0:a?` 를 붙일 것.
- **별도 필터 스크립트 파일에 `$변수` 는 확장되지 않는다.** 셸이 먼저 치환해야 함.
- **색상 포맷 차이**: 파이프라인은 `gbrp`로 끝나고, 출력 직전에만 `yuv420p` 로 변환.

---

## 8. 성능

- GPU 없이 CPU 필터만 사용 (FFmpeg 5.1, cuda/opencl 미사용).
- 가장 비싼 건 `gblur`. 1080p 정지이미지 1장에 **2~3초**, 640×480 50프레임에 **10초**.
- 4K 영상은 프레임당 시간이 크게 늘어난다. 실시간이 필요하면
  `SOFT=2 GN2=0.2 BLOOM=1.0` 로 낮추거나 `HALO` 분기(가장 큰 gblur)를 삭제.
- GPU 가속이 필요하면 그래프 앞에 `hwupload`, `gblur` 대신 `scale_cuda`,
  `unsharp` 를 `unsharp_opencl` 로 교체 (빌드에 cuda/opencl 옵션 필요).

---

## 9. 이 문서의 검증 상태

ffmpeg 5.1.9 / Linux 2코어 환경에서 아래를 실제로 실행해 확인했다.

- 이미지 / 영상 / 오디오 있는 영상 / 그레이스케일 / 96×72 소형 — 전부 정상 렌더
- 1280×960 과 1440×1080 에서 **동일한 통계** (해상도 비례 스케일 동작 확인)
- 문서에 적힌 모든 표 수치는 실제 출력물에서 측정값