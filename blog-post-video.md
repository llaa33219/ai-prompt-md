# 블로그 포스트 형식으로 영상을 제작하는 방법
## 0. 정보
이것은 블로그 포스트 형식으로 영상을 제작하는 방법이 담긴 가이드이다. 아래 내용을 따라야 한다.
## 1. 유저에게 요청하기
1. 영상에 담을 내용([소스 자동]을 서술하면, 소스를 자동으로 제작하며, [링크 또는 파일 이름, 파일 경로]를 서술하고 파일이라면 소스를 제공해 준다면 첨부된 소스를 사용하겠다고 알리고, [자동 생성]내용 을 입력하면 그 부분은 직접 생성하여 작성한다는 것을 알리고, 소스 아래 글자 부분은 간단 명료하고 최대 3줄까지라는걸 알리고, **내용**을 사용하면 그 부분은 강조로 치부된다는 것을 알릴 것.)
2. 영상 형식(9:16, 16:9 중 어떤 비율인지 물어볼 것)
3. 영상에서 사용할 언어
## 2. 소스 제작
1에서 유저가 기획한 부분에 [소스 자동]이 있다면, 적절한 비율로 알아서 생성할 것.(9:16 시 1:1 비율, 16:9 시 19:9 비율로 생성.)
시각화 시 manim/차트js 등을 사용해도 좋다.
### 2.1. 소스 검증
소스를 직접 보고 정상적으로 생성되었는지, 깨진 부분은 없는지, 검증함
## 3. 번역
사용자가 준 대사를 요청된 언어로 번역한다.
직접 생성한 대사는 애초부터 요청된 언어로 생성하도록 한다.
## 4. 영상 생성
ffmpeg를 사용한다.
### 4.1. 글꼴과 미디아, 배경
폰트는
```
@font-face {
    font-family: 'Paperozi';
    src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/2408-3@1.0/Paperlogy-1Thin.woff2') format('woff2');
    font-weight: 100;
    font-display: swap;
}

@font-face {
    font-family: 'Paperozi';
    src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/2408-3@1.0/Paperlogy-2ExtraLight.woff2') format('woff2');
    font-weight: 200;
    font-display: swap;
}

@font-face {
    font-family: 'Paperozi';
    src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/2408-3@1.0/Paperlogy-3Light.woff2') format('woff2');
    font-weight: 300;
    font-display: swap;
}

@font-face {
    font-family: 'Paperozi';
    src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/2408-3@1.0/Paperlogy-4Regular.woff2') format('woff2');
    font-weight: 400;
    font-display: swap;
}

@font-face {
    font-family: 'Paperozi';
    src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/2408-3@1.0/Paperlogy-5Medium.woff2') format('woff2');
    font-weight: 500;
    font-display: swap;
}

@font-face {
    font-family: 'Paperozi';
    src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/2408-3@1.0/Paperlogy-6SemiBold.woff2') format('woff2');
    font-weight: 600;
    font-display: swap;
}

@font-face {
    font-family: 'Paperozi';
    src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/2408-3@1.0/Paperlogy-7Bold.woff2') format('woff2');
    font-weight: 700;
    font-display: swap;
}

@font-face {
    font-family: 'Paperozi';
    src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/2408-3@1.0/Paperlogy-8ExtraBold.woff2') format('woff2');
    font-weight: 800;
    font-display: swap;
}

@font-face {
    font-family: 'Paperozi';
    src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/2408-3@1.0/Paperlogy-9Black.woff2') format('woff2');
    font-weight: 900;
    font-display: swap;
}
```
이것을 사용한다.
기본은 700, 볼드는 900을 사용하며, 필요 시 굵기를 변경하여도 된다.(**내용**인 부분을 강조로 감지한다. ****은 굵기 변경 시 자동으로 제거한다.)
미디아는 세로 길이는 auto(최대 세로 길이는 화면의 70%), 가로 길이는 화면 가로길이의 80%가 되게 하며, 모서리를 6px만큼 둥글게 처리한다.
배경은 완전한 검정색을 사용한다.
글자와 미디아는 화면 상단에서 35% 떨어진 곳에 상단정렬(9:16 일 때)이나 미디아 기준 화면의 중앙(16:9 일 때)에 정렬한다.
미디아에서 10px 아래에 글자를 배치하며, 글자는 화면 가로의 90% 가로길이 최대값을 가지게 한다.(글자는 기본적으로 중앙정렬)
폰트 크기는 36px를 사용한다.
### 4.2. 글자 등장과 미디아 등장 및 나레이션
맨 처음 등장을 제외한 모든 것은 이전 요소가 스크롤되어 위로 올려지며, 아래에서 올라와 등장한다.
이전 요소가 화면에 있는 상태일 때, 아래(다음에 등장할)요소는 미디아 윗부분의 35%정도가 화면 아래에 보이게 한다.(16:9시에는 화면에 들어오지 않게 아래에 둔다.)
스크롤 이동은 ease-out으로 진행하며, 모든 요소가 동일하게 움직인다.
스크롤 시작 시간은 현재 읽는 부분의 나레이션 종료 직후이다.
미디아 요소가 영상이라면(또는 gif를 제외한 애니메이션 포함 요소라면) 완전히 스크롤이 완료되어 정확한 위치에 들어왔을 시 재생을 시작하며 한 번만 재생한다. gif라면 처음부터 재생되어 반복되고 있으면 된다.
글자는 최대 3줄까지 추가 가능하며, 기본적으로 첫 줄만 등장한 후, 나레이션에 따라 첫 줄의 나레이션이 끝나면 둘째줄이 애니메이션 없이 딱딱하게 바로 등장한다.(당연히 둘째줄 나레이션이 끝나면 셋째줄이 등장한다.)
맨 마지막에는 더이상 등장할 요소가 존재하지 않는다면, 나레이션 종료 후 0.5초 후 영상을 종료한다.
나레이션 목소리는 알아서 선정한다.
### 4.3. 주의점
화면을 "스크롤"하듯 부드럽게 "모든 요소가 동일한 속도로" 움직여야 하며, 한 요소가 올라가고 다음 요소가 나오거나, 올라간 요소가 올라가다 소멸하거나, 아래 요소가 미리 보이지 않고 없다가 등장(9:16 경우)하거나, 움직일 시 요소들 속도의 차이가 발생하지 않아야 한다.
### 4.4. 출력
사용자에게 영상을 mp4로 제공한다.
