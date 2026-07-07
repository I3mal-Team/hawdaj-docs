# Feature 32 — AI ChatGPT Assistant (WEB-ONLY)

> **Status:** ✅ analyzed · **Priority:** P3 · **Category:** Web-only (discovery enrichment)
> **Web:** `hawdaj-frontend` Places/Stores detail components · **Mobile:** ❌ none · **Laravel:** `getChatGpt`/`addChatGpt` endpoints
> **One-line:** A ChatGPT-backed assistant surfaced only on the **web** Place and Store detail pages: on load it fetches AI-generated info about the item (`GET getChatGpt`) and can send a user message (`POST addChatGpt`). The Flutter mobile app has **no equivalent** — this is a genuine web-only feature and a mobile parity gap.

---

## 1. Feature Name & Identity
- **Name:** AI ChatGPT Assistant — AI info/chat about a Place or Store.
- **Platform:** Web only (Angular). Not in mobile. Backend endpoints exist (`getChatGpt`, `addChatGpt`).
- **Surface:** Place detail + Store detail pages (ChatGPT SVG icon `/assets/images/places/chatgpt.svg`).

## 2. Business Goal & Rules
- **Goal:** Enrich content discovery with AI-generated info/answers about a specific place/store.
- **Rules (observed):** loads AI info on detail init (`getChatGpt`); user can submit a message (`addChatGpt`). Loading state `isLoadingChat`. Response stored in `chat`.

## 3. User Flow (web)
1. Open Place/Store detail → component `ngOnInit` calls `getChatGpt()` → shows loading → renders AI `chat` content.
2. User submits a query → `addChatGpt(data)` → response appended/updated.

## 4. Screens / Views
- Place detail: `components/places/components/place-details/place-details.component.ts:448–460` (+ place-details-v2). Store detail: `stores-management/components/store-details/store-details.component.ts:589–602`. Templates show chatgpt.svg + loading + chat output.

## 5. Widgets
- Chat/info panel on detail page; ChatGPT icon; loading spinner (`isLoadingChat`).

## 6. Services / State
- `PlacesService.getChatGpt()` (`places.service.ts:111–112`), `PlacesService.addChatGpt(data)` (`:114–115`).
- `StoresService.getChatGpt()` (`stores.service.ts:78–79`), `addChatGpt(data)` (`:81–82`).
- RxJS service state (no NgRx).

## 7. API Endpoints
| Method | Endpoint | endPoints.ts | Purpose |
|--------|----------|--------------|---------|
| GET | `getChatGpt` | 67, 92 | fetch AI info for item |
| POST | `addChatGpt` | 68, 93 | submit user message / generate |

- Consumed for Places + Stores. (Other entities not wired.)

## 8. Request / Response
- **GET resp:** `{code, message?, data: <ai_text/info>}`.
- **POST req:** `addChatGpt(data)` — payload (item context + message) posted; response is AI content. Exact schema server-defined.

## 9. Laravel Side
- Endpoints `getChatGpt`/`addChatGpt` handled by the API (likely a controller wrapping an OpenAI/ChatGPT call). **Not analyzed in the mobile/backend passes** because mobile never calls them — flagged here for backend follow-up (auth? rate limit? API key location? cost controls?).

## 10. Edge Cases
- AI service slow/unavailable → loading hangs / empty chat (web handles via `isLoadingChat`).
- No mobile parity → inconsistent product experience across platforms.
- Backend cost/rate exposure if endpoint unauthenticated or unthrottled (verify server-side).

## 11. Differences vs Mobile
- **Mobile:** feature absent entirely.
- **Web:** integrated into detail pages for Places/Stores only.

## 12. Dependencies
- Web: Angular HttpClient + Places/Stores services + detail components.
- Backend: `getChatGpt`/`addChatGpt` controller + (presumably) an LLM provider.
- **Cross-feature:** [[04_Places]], [[05_Stores]] (host pages), [[24_Networking]] (HTTP), [[WEB_FRONTEND]], [[WEB_MOBILE_COMPARISON]].

## 13. Files Involved
`places.service.ts:111–115`, `stores.service.ts:78–82`, `place-details.component.ts:448–460`, `place-details-v2.component.ts`, `store-details.component.ts:589–602`, `endPoints.ts:67–68,92–93`, chatgpt.svg asset. Backend: getChatGpt/addChatGpt controller (hawdaj-api).

## 14. Full Execution Flow
Detail page init → `getChatGpt()` → GET `getChatGpt` (Bearer+locale+geo headers) → API → LLM → AI text → `chat` rendered. User message → `addChatGpt(data)` → POST → AI response.

## 15. Performance / Cost
| Issue | Severity | Note |
|-------|----------|------|
| Synchronous fetch on every detail open | 🟡 | LLM latency on page load |
| Potential per-request LLM cost | 🟡 | verify caching/rate-limit server-side |
| No mobile parity | 🟢 | product gap |

## 16. Security
| Issue | Severity | Note |
|-------|----------|------|
| Backend auth/throttle on getChatGpt/addChatGpt | ❓ verify | if public+unthrottled → cost/abuse risk |
| LLM API key location (server) | ❓ verify | must be server-side, not exposed |
| Prompt-injection via addChatGpt input | 🟡 | sanitize/limit server-side |

## 17. Improvement Opportunities
- Add auth + rate limiting + response caching on the backend endpoints.
- Bring parity to mobile (or document as intentionally web-only).
- Cache AI info per item (don't re-generate every visit).

---

## 32. Related Features
- **[[04_Places]] [[05_Stores]]** — host detail pages.
- **[[24_Networking]]** — HTTP layer.
- **[[WEB_FRONTEND]] [[WEB_MOBILE_COMPARISON]]** — web context + gap.

## 33. How to Modify
- **Add to another entity (web):** add getChatGpt/addChatGpt to that service + wire into its detail component (copy Places pattern).
- **Add to mobile:** implement a repo/cubit calling getChatGpt/addChatGpt + a chat widget on Place/Store detail.
- **Secure backend:** add auth:api + throttle + cache to the controller.

## 34. Regression Checklist
- [ ] Place detail loads AI info.
- [ ] Store detail loads AI info.
- [ ] User message returns AI response.
- [ ] Loading/empty/error states handled.
- [ ] Backend endpoint auth/throttle verified.

## 35. Common Bugs / Debug
- Empty chat → check getChatGpt response + isLoadingChat.
- Slow detail page → LLM latency; consider caching.
- 401 → web auto-logout (ErrorInterceptor) may fire if token invalid.

## 37. Search Keywords
`getChatGpt`, `addChatGpt`, `ChatGpt`, `chatgpt.svg`, `isLoadingChat`, `place-details chat`, `store-details chat`, AI assistant, مساعد ذكي, ChatGPT.

## 38. Future Improvements
- Cross-platform parity, cached/streamed responses, secured+throttled backend, broaden to events/guides.
