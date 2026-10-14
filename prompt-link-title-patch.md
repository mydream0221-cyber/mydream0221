# 프롬프트 수정안: 링크 제목 확보

원인: t.me/s 페이지에는 링크 미리보기 제목/설명(`tgme_widget_message_link_preview_title`, `..._description`)이 들어 있는데, 기존 스크립트가 이를 버리고 있었음. 네이버처럼 WebFetch가 막히는 링크도 이 제목은 얻을 수 있음 (이번 실행에서 확인).

## 수정 1 — 1단계 스크립트의 parse() 교체

```python
def parse(channel, content):
    blocks = re.split(r'(?=<div class="tgme_widget_message[ "])', content)
    out = []
    clean = lambda s: html.unescape(re.sub(r'<[^>]+>', '', s)).strip()
    for b in blocks:
        m_id = re.search(r'data-post="%s/(\d+)"' % channel, b)
        m_time = re.search(r'<time datetime="([^"]+)"', b)
        if not m_id or not m_time:
            continue
        m_text = re.search(r'<div class="tgme_widget_message_text[^"]*"[^>]*>(.*?)</div>', b, re.S)
        text = ""
        if m_text:
            raw = re.sub(r'<br\s*/?>', '\n', m_text.group(1))
            text = clean(raw)
        m_pt = re.search(r'link_preview_title[^>]*>(.*?)</div>', b, re.S)
        m_pd = re.search(r'link_preview_description[^>]*>(.*?)</div>', b, re.S)
        out.append({"id": int(m_id.group(1)), "time": m_time.group(1), "text": text,
                    "preview_title": clean(m_pt.group(1)) if m_pt else "",
                    "preview_desc": clean(m_pd.group(1)) if m_pd else ""})
    return out
```

## 수정 2 — 3단계 "링크만 있는 글 처리" 문단을 아래로 교체

**링크만 있는 글 처리**: `text`가 URL(또는 URL과 짧은 부가어) 외에 실질 본문이 없는 글은 건너뛰지 말고 다음 순서로 처리한다.
1. JSON의 `preview_title`/`preview_desc`(텔레그램 링크 미리보기)가 있으면 이것을 1차 근거로 삼아, 다른 글과 동일한 형식으로 요약하고 시황/호재/악재로 분류한다. 이 경우 WebFetch는 생략한다(네이버 링크 포함). 미리보기에 없는 내용은 추가하지 않는다.
2. 미리보기가 없고 네이버 계열이 아니면 WebFetch로 제목과 핵심 내용을 확인한다(링크당 재시도 1회, 채널당 최대 10건).
3. 위 두 방법으로도 제목을 못 얻은 경우에만 "(링크: 내용 확인 불가)" 대신 **글에 함께 적힌 텍스트나 URL 도메인이라도 활용해 제목 수준으로 표기**하고(예: `[주제어] 연합인포맥스 링크, 본문 미확인`), 완전히 알 수 없을 때만 "(링크: 제목 확인 불가)"로 표시한다.

## 수정 3 — 참고사항의 네이버 문구 교체

기존: "네이버 계열 링크(naver.me, n.news.naver.com)는 항상 차단되므로 시도하지 않고 바로 '(링크: 내용 확인 불가)'로 처리해도 된다."
변경: "네이버 계열 링크(naver.me, n.news.naver.com)는 WebFetch가 차단되므로 시도하지 말고, 텔레그램 미리보기 제목(`preview_title`)으로 처리한다."
