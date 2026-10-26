# 확장 패턴 레퍼런스

템플릿에 없는 기능이 필요할 때만 읽는다. 템플릿의 구조(상태 클래스 `is-prev`/`is-active`/`is-next`, `paint()` 한 곳에서 갱신)를 깨지 않고 얹는 방식이다.

## 목차
1. 전환 효과 바꾸기
2. 클릭마다 하나씩 등장 (data-step)
3. 코드 줄 강조
4. 화자 노트
5. 고정 16:9 캔버스 (scale 방식)
6. 호환성·함정

## 1. 전환 효과 바꾸기

기본은 세로 이동 + 페이드다. `.slide.is-next` / `.is-prev`의 `transform`만 바꾸면 효과가 바뀐다. `transform`과 `opacity`만 애니메이션해야 GPU 합성으로 부드럽다. `left`, `top`, `width`는 피한다.

```css
/* 가로 슬라이드 */ .slide.is-next { transform: translate3d(100%,0,0); } .slide.is-prev { transform: translate3d(-100%,0,0); }
/* 줌 */           .slide.is-next { transform: scale(1.25); }           .slide.is-prev { transform: scale(.75); }
/* 페이드 */        .slide.is-next, .slide.is-prev { transform: none; }
```

`display:none`은 transition을 끊으므로 쓰지 않는다. 대신 `visibility` + `transition-delay`로 퇴장 슬라이드를 전환이 끝날 때까지 보이게 한다(템플릿에 이미 적용됨).

## 2. 클릭마다 하나씩 등장 (data-step)

`.reveal` 자동 등장 대신 → 키마다 한 항목씩 보여 주고 싶을 때 쓴다. step을 먼저 소비하고, 더 없으면 다음 슬라이드로 간다.

```html
<li data-step="1">…</li> <li data-step="2">…</li>
```
```css
[data-step] { opacity: 0; transform: translateY(24px); transition: opacity .4s, transform .4s; }
[data-step].visible { opacity: 1; transform: none; }
@media print { [data-step] { opacity: 1 !important; transform: none !important; } }
```
```js
const maxStep = s => Math.max(0, ...[...s.querySelectorAll('[data-step]')].map(e => +e.dataset.step));
const applyStep = (s, n) => s.querySelectorAll('[data-step]').forEach(e => e.classList.toggle('visible', +e.dataset.step <= n));
// next(): step < maxStep(slide) ? applyStep(slide, ++step) : goTo(current + 1)
// prev(): step > 0 ? applyStep(slide, --step) : goTo(current - 1)  → 이전 슬라이드는 step을 maxStep으로 열어 둔다
```
슬라이드가 바뀔 때 `step = 0`으로 초기화하고, 뒤로 이동할 때만 전부 공개한 상태로 보여 준다.

## 3. 코드 줄 강조

```html
<pre data-highlight="1-2|4|5-6"><code>첫 줄
둘째 줄
…</code></pre>
```
`|`로 단계를 나누고 `,`와 `-`로 줄 범위를 쓴다. 코드의 `<`와 `&`는 이스케이프한다.
```js
const esc = s => s.replace(/&/g,'&amp;').replace(/</g,'&lt;');
code.innerHTML = code.textContent.split('\n').map(l => `<span class="ln">${esc(l)||' '}</span>`).join('');
const parseRange = str => { const set = new Set(); str.split(',').forEach(p => { if(!p) return;
  const [a,b] = p.split('-').map(Number); for (let i=a;i<=(b||a);i++) set.add(i); }); return set; };
```
```css
.ln { display:block; transition: opacity .3s, background .3s; }
pre.dim .ln { opacity:.25; } pre.dim .ln.hl { opacity:1; background: var(--accent-dim); }
```
강조 단계는 data-step 시스템의 step 수에 합산한다.

## 4. 화자 노트

```html
<aside class="notes">발표자만 볼 메모</aside>
```
```css
aside.notes { display:none; }
body.show-notes .is-active aside.notes { display:block; position:absolute; left:0; right:0; bottom:0;
  padding:1rem 2rem; background:rgba(0,0,0,.85); color:var(--text-primary); }
```
S 키로 `document.body.classList.toggle('show-notes')`. 별도 발표자 창은 `window.open` + `BroadcastChannel`로 만들 수 있지만 `file://`에서는 브라우저에 따라 동작하지 않을 수 있다.

## 5. 고정 16:9 캔버스 (scale 방식)

템플릿은 vw/rem clamp 기반의 반응형이다. 픽셀 단위로 정확히 맞춘 디자인이나 PDF 출력이 필요하면 1920×1080 stage를 두고 JS로 scale한다.

```css
.stage { position:absolute; left:50%; top:50%; width:1920px; height:1080px; overflow:hidden;
  transform:translate(-50%,-50%) scale(var(--s,1)); }
```
```js
const fit = () => document.documentElement.style.setProperty('--s', Math.min(innerWidth/1920, innerHeight/1080));
addEventListener('resize', fit); visualViewport?.addEventListener('resize', fit); fit();
```
번호·진행 바는 stage 바깥에 둔다. scale 아래에서는 `getBoundingClientRect` 값이 스케일돼 나오고, 텍스트가 약간 흐릿해질 수 있다.

## 6. 호환성·함정

- 모바일 `100vh`는 주소창 때문에 잘린다. `.deck`은 `position:fixed; inset:0`이라 영향이 없다. 다른 곳에서 높이를 쓸 때는 `height:100vh; height:100dvh;` 순서로 폴백을 둔다.
- `transitionend`는 탭이 숨겨지거나 duration이 0이면 오지 않는다. 템플릿은 타이머로 잠금을 해제한다.
- iPhone Safari는 문서 전체 Fullscreen API를 지원하지 않는다. `requestFullscreen`이 없으면 조용히 무시한다.
- 해시는 `#slide-3` 같은 id 형태 대신 `#3`처럼 숫자로 쓴다. id 형태는 앵커 스크롤이 튄다.
- 슬라이드 안에 스크롤 영역이 생기면 스와이프·휠 핸들러와 충돌한다. 내용을 줄이거나 슬라이드를 나누는 쪽이 낫다.
