# Texture Boil Effect

이미지에 표면이 끓어오르는 듯한 불연속 왜곡 효과를 적용하여 영상으로 출력하는 방법.

## 핵심 아이디어

두 개의 프레임 레이트를 분리:
- **Video FPS**: 영상 재생 속도 (부드럽게, e.g. 30)
- **Effect FPS**: 왜곡 패턴이 바뀌는 속도 (끊기게, e.g. 6)

Effect FPS마다 완전히 새로운 랜덤 왜곡 맵을 생성. 이전 프레임과 연속성이 전혀 없어 표면이 "팍팍" 튀는 Boil 느낌이 나타남.

## 알고리즘

### 4-Layer Random Distortion

각 효과 프레임마다 4개의 사인파 레이어를 독립적으로 생성. 모든 위상, 주파수, 진폭은 매 프레임 재생성.

```
dx = dx1 + dx2 + dx3 + dx4
dy = dy1 + dy2 + dy3 + dy4

output[x, y] = input[x + dx, y + dy]
```

### 레이어 파라미터

| 레이어 | 스케일 | 주파수 범위 | 진폭 범위 (기본) | 진폭 범위 (x0.5) | 역할 |
|---|---|---|---|---|---|
| 1 | 대형 | 30-70 px | 2.5 - 6 px | 1.25 - 3 px | 넓은 연결 표면 흐름 |
| 2 | 중형 | 15-35 px | 1.5 - 3.5 px | 0.75 - 1.75 px | 각진 파형 왜곡 |
| 3 | 세밀 | 8-20 px | 1 - 2 px | 0.5 - 1 px | 보일 텍스처 입자감 |
| 4 | 미세 | ~10 px | 0.5 - 1.25 px | 0.25 - 0.625 px | 미세한 쉬머 |

전체 진폭은 `strength` 배수로 조정. 기본 1.0, 절제된 효과는 0.5.

### 랜덤 시드 전략

각 레이어는 효과 프레임 인덱스에 고정된 시드 오프셋을 사용하여 독립적인 랜덤성을 보장:
- Layer 1: `seed = e_idx * 7919 + 12345`
- Layer 2: `seed = e_idx * 6899 + 54321`
- Layer 3: `seed = e_idx * 4241 + 99999`
- Layer 4: `seed = e_idx * 3323 + 11111`

## Python 구현 (numpy + Pillow)

```python
import numpy as np
from PIL import Image

def generate_distortion_maps(width, height, num_effect_frames, strength=1.0):
    x = np.arange(width, dtype=np.float32)
    y = np.arange(height, dtype=np.float32)
    X, Y = np.meshgrid(x, y)
    maps = []

    for e_idx in range(num_effect_frames):
        # Layer 1: large-scale
        np.random.seed(e_idx * 7919 + 12345)
        a1 = np.random.uniform(2.5, 6) * strength
        dx1 = a1 * np.sin(Y/np.random.uniform(30,70) + np.random.uniform(0, 2*np.pi)) * \
              np.cos(X/np.random.uniform(30,70) + np.random.uniform(0, 2*np.pi))
        dy1 = a1 * np.cos(X/np.random.uniform(30,70) + np.random.uniform(0, 2*np.pi)) * \
              np.sin(Y/np.random.uniform(30,70) + np.random.uniform(0, 2*np.pi))

        # Layer 2: medium-scale
        np.random.seed(e_idx * 6899 + 54321)
        a2 = np.random.uniform(1.5, 3.5) * strength
        dx2 = a2 * np.sin(Y/np.random.uniform(15,35) + np.random.uniform(0, 2*np.pi)) * \
              np.cos(X/np.random.uniform(15,35) + np.random.uniform(0, 2*np.pi))
        dy2 = a2 * np.cos(X/np.random.uniform(15,35) + np.random.uniform(0, 2*np.pi)) * \
              np.sin(Y/np.random.uniform(15,35) + np.random.uniform(0, 2*np.pi))

        # Layer 3: fine boil
        np.random.seed(e_idx * 4241 + 99999)
        a3 = np.random.uniform(1.0, 2.0) * strength
        dx3 = a3 * np.sin(X/np.random.uniform(8,20) + np.random.uniform(0, 2*np.pi)) * \
              np.cos(Y/np.random.uniform(8,20) + np.random.uniform(0, 2*np.pi))
        dy3 = a3 * np.cos(Y/np.random.uniform(8,20) + np.random.uniform(0, 2*np.pi)) * \
              np.sin(X/np.random.uniform(8,20) + np.random.uniform(0, 2*np.pi))

        # Layer 4: micro shimmer
        np.random.seed(e_idx * 3323 + 11111)
        a4 = np.random.uniform(0.5, 1.25) * strength
        dx4 = a4 * np.sin(X/10 + Y/14 + np.random.uniform(0, 2*np.pi))
        dy4 = a4 * np.cos(Y/12 + X/8 + np.random.uniform(0, 2*np.pi))

        dx, dy = dx1+dx2+dx3+dx4, dy1+dy2+dy3+dy4
        map_x = np.clip((X + dx), 0, width - 1).astype(np.int32)
        map_y = np.clip((Y + dy), 0, height - 1).astype(np.int32)
        maps.append((map_x, map_y))

    return maps

def render(img_path, out_dir, width=1280, height=720,
           video_fps=30, effect_fps=6, duration=4, strength=1.0):
    img = Image.open(img_path).convert('RGB')
    img.thumbnail((width, height), Image.LANCZOS)
    canvas = Image.new('RGB', (width, height), (0,0,0))
    canvas.paste(img, ((width-img.width)//2, (height-img.height)//2))
    img_arr = np.array(canvas).astype(np.float32)

    num_effects = effect_fps * duration
    maps = generate_distortion_maps(width, height, num_effects, strength)

    frames_per = video_fps // effect_fps
    for v_idx in range(video_fps * duration):
        e_idx = min(v_idx // frames_per, len(maps)-1)
        distorted = img_arr[maps[e_idx][1], maps[e_idx][0]]
        Image.fromarray(distorted.astype(np.uint8), 'RGB').save(
            f'{out_dir}/frame_{v_idx:04d}.png')

# 사용 예
# render('input.png', 'frames', video_fps=30, effect_fps=6, strength=0.5)
# ffmpeg -y -framerate 30 -i frames/frame_%04d.png -vf format=yuv420p -c:v libx264 -crf 18 output.mp4
```

## ffmpeg 인코딩

```bash
ffmpeg -y -framerate 30 -i frames/frame_%04d.png \
  -vf "format=yuv420p" -c:v libx264 -crf 18 \
  -pix_fmt yuv420p -movflags +faststart output.mp4
```

## 조정 가이드

| 원하는 느낌 | video_fps | effect_fps | strength |
|---|---|---|---|
| 강한 boil, 끊김 강함 | 30 | 4 | 1.0 |
| 적당한 boil, 적당한 끊김 | 30 | 6 | 1.0 |
| 절제된 boil, 약한 끊김 | 30 | 12 | 0.5 |
| 매우 절제된 흔들림 | 30 | 6 | 0.3 |

**주의:** video_fps는 반드시 effect_fps로 나누어 떨어져야 함.

## 하면 안 되는 것

1. **시간 기반 연속 왜곡 금지**: `sin(T * omega)` 형태로 시간에 따라 부드럽게 변화시키면 flow 효과가 되고 boil이 아님. 반드시 매 프레임 독립 랜덤
2. **블록 기반 분리 금지**: 픽셀 단위 연속 표면 왜곡을 유지. 블록 단위로 쪼개면 다른 효과가 됨
3. **Video fps = Effect fps 금지**: 두 값을 같게 하면 그냥 낮은 fps 영상이 됨. 분리가 핵심
