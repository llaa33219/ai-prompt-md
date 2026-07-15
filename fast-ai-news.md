# FAST AI NEWS 영상 제작법 설명
## 1. 사용자가 질문한 AI 모델에 대하여 알아본다.
검색을 하여, 레딧 반응, 실사용 후기, 벤치마크, 가격, 특징, 혁신적인 부분, 들어간 기술 등 세부적인 모든 사항을 수집한다.
## 2. 3개의 주요 특징 추출
수집된 결과 전체를 사용자에게 섬세하게 알려준 후, 3가지의 특징적인 부분을 추출한다.
3가지의 특징적인 부분은 다른 충격적인 부분이 있지 않는 한 아래 내용을 기준으로 한다.
1. 벤치마크 성능(어떠한 모델보다 뛰어나다, 어떠한 모델보다 떨어진다, 어떠한 모델과 비슷하다 중 하나 선택하여 서술)
2. 모델의 장점(간략하며, 강력한 모델의 장점 하나 서술. 없으면 없다고 서술할 것. 사용자 평가가 좋지 않으면 없다고 서술할 것.)
3. 실사용 후기(사용자들의 실사용 후기 중 가장 많이 나온 것을 간략하게 작성.)

특징을 추출한 후, 사용자에게 괜찮은지 물어보고, 확정 판단이 나면 3을 진행한다.
## 3. 영상 제작
### 3.1. 스타일
```
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1080 1920" width="1080" height="1920" fill="none">

  <defs>
    <style>
      @font-face {
        font-family: 'Google Sans Code';
        font-style: normal;
        font-weight: 700;
        src: url(https://design.penpot.app/internal/gfonts/font/googlesanscode/v17/pxihyogzv91QhV44Z_GQBHsGf5PuckJMZfIVTPZaiXEp_ht12EVEHsN1sCQNlWPltAki.woff2) format('woff2');
      }
      @font-face {
        font-family: 'Google Sans Code';
        font-style: normal;
        font-weight: 800;
        src: url(https://design.penpot.app/internal/gfonts/font/googlesanscode/v17/pxihyogzv91QhV44Z_GQBHsGf5PuckJMZfIVTPZaiXEp_ht12EVEHsN1sCQN8mPltAki.woff2) format('woff2');
      }
      .label,
      .heading { font: 72px 'Google Sans Code', monospace; fill: #fff;
                 dominant-baseline: ideographic; }
      .label   { font-weight: 700; }
      .heading { font-weight: 800; }
    </style>

    <!-- Bottom-heavy dark overlay used on the image area -->
    <linearGradient id="imageOverlay" x1="0.5" y1="0" x2="0.5" y2="1">
      <stop offset="0.65" stop-color="#B1B2B5" stop-opacity="0"/>
      <stop offset="1" stop-color="#000000" stop-opacity="0.75"/>
    </linearGradient>
  </defs>

  <!-- Page background -->
  <rect width="1080" height="1920" fill="#000"/>

  <!-- Image area: white plate + gradient overlay (y: 420–1500) -->
  <rect x="0" y="420" width="1080" height="1080" fill="#fff"/>
  <rect x="0" y="420" width="1080" height="1080" fill="url(#imageOverlay)"/>

  <!-- Header pill: 573×121 blue button, right corners rounded r=60.5 -->
  <path d="M 0 520 H 512.5 A 60.5 60.5 0 0 1 573 580.5 A 60.5 60.5 0 0 1 512.5 641 H 0 Z" fill="#007BFF"/>

  <!-- Title -->
  <text x="22" y="625.7" class="label" textLength="518.4" lengthAdjust="spacingAndGlyphs">fast ai news</text>

  <!-- Body placeholders -->
  <text x="38" y="1326.7" class="heading" textLength="993.6" lengthAdjust="spacingAndGlyphs">text~~~~~~~~~~~~~~~~~~~</text>
  <text x="38" y="1413.1" class="heading" textLength="993.6" lengthAdjust="spacingAndGlyphs">~~~~~~~~~~~~~~~~~~~~~~~</text>
  <text x="38" y="1499.5" class="heading" textLength="993.6" lengthAdjust="spacingAndGlyphs">~~~~~~~~~~~~~~~~~~~~~~~</text>
</svg>
```
을 따르며, text의 강조가 필요할 경우, #007bff 색상을 사용한다.
이미지 구역의 경우는, 각 3장면에 따라 필요한 이미지를 다운로드하거나, 웹페이지의 적절한 부분을 스크린샷하여 사용한다.
장면전환 효과, 등장 효과 등은 넣지 않는다.
### 3.2. 진행
뉴스에서 사용될 법 한 목소리로 나레이션을 하며,
아래와 같이 진행한다.
1. 모델 로고와 (모델 이름)을 빠르게 알아보자. 라고 쓰인 글자. 나레이션은 (모델 이름)을 빠르게 알아보자.
2. 1~3번째 장면 순서대로. 써있는 그대로 나레이션
이렇게 진행한다.
사용자가 따로 요청하지 않을 시 영상은 영어로 제작하는 것이 기본이며, 9:16. 세로형 비율로 제작해야 한다. 너무 느리게 진행해서는 안되며, 빠르게 진행되어야 한다.
### 3.3. 기술
ffmpeg와 음성 생성 도구와 기타 여러 기능을 사용하여 완성시킨다.
### 3.4. 출력
mp4 파일로 사용자에게 제공한다.****
