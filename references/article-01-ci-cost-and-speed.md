# 改一篇文件也跑完整 CI：我的 2,000 分鐘就是這樣沒的

> CI/CD 實戰系列：**第一篇** ｜
> [第二篇：Railway 顯示 SUCCESS，我為什麼還是不讓它上線？](article-02-exact-sha-release.md) ｜
> [Evidence-first CI/CD writing Skill](../SKILL.md)

事情是這樣的。

我的 GitHub Actions 額度不是偶爾不夠，而是連續幾個月，每個月 2,000
分鐘都被燒光。

最後我只好把自己的 Mac 拿來當 self-hosted runner。

但問題沒有因此消失。

```text
改 Backend  → BE、FE、App 全跑
改 Frontend → BE、FE、App 全跑
改文件      → 還是全部跑
```

把 CI 搬到 Mac，只是把 GitHub 分鐘換成另一種成本：

- Mac 必須一直開著
- Backend 和 Frontend 搶 CPU
- VM 偶爾卡住
- Cache 和磁碟會越來越大
- Runner 掛掉後，GitHub job 只會一直排隊
- 不必要的 Build 還是照跑

我後來才發現，真正的問題不是 GitHub 給的分鐘太少，而是 Pipeline 根本
不知道這次改了什麼。

## 先不要急著加 Runner

一開始我也想過增加 cache、CPU、VM，或直接減少測試。

但如果不知道時間花在哪裡，這些都只是在猜。

所以我先把 CI 拆開量：

```text
Runner 排隊
Checkout
Dependencies install
Static checks
Security checks
Tests
Build
Provider deployment
Production readback
```

這時才發現，CI 慢和部署慢其實是兩件事。

有一次 SpeakUp 完整 Publisher 花了 13 分 34 秒，其中 Railway API 和
Worker publication 約占量測時間的 79%。所以再怎麼優化 Cloudflare，都
不會解決主要問題。

## Backend 和 Frontend 可以並行

原本的流程比較接近：

```text
Backend → Frontend → Deploy
```

我先改成：

```text
Backend checks  ─┐
                 ├→ Aggregate Gate → Deploy
Frontend checks ─┘
```

Backend 和 Frontend 使用不同 runner，同時安裝依賴、執行測試。

最近一次 PR exact-head CI：

- Backend：5 分 21 秒
- Frontend：4 分 16 秒
- GitGuardian：15 秒

如果串行執行，Backend 加 Frontend 會接近十分鐘。並行後，主要等待時間
是比較慢的 Backend，大約五分多鐘。

但是我沒有拿掉任何 required check。部署仍然要求：

```text
backend-complete = success
frontend-complete = success
GitGuardian = success
```

任何一條失敗，都不能繼續發布。

## Backend 裡面再拆一次

Backend 自己也不是只能從頭排到尾。

我保留一次 dependency install、一個完整 regression suite owner，以及
一個 PostgreSQL 測試 owner，再把互不衝突的工作並行：

```text
Dependencies install
   ├─ Source / static checks
   ├─ Readiness / security checks
   └─ Migration / full regression
```

三條 lane 最後仍然回到同一個 `backend-complete`。任何一條失敗，整個
Backend check 就失敗。

一次 main CI 的 Backend 從 6 分 14 秒降到 5 分 05 秒，少了 69 秒。
這只是一筆真實樣本，還不能說是穩定提升 18%。要累積相同 workload
的多次結果，才能談 P50 或 P95。

## 有變更才跑需要的部分

完成並行後，我回頭看最大的浪費：為什麼只改文件，也要跑 3,000 多個
Backend tests 和 700 多個 Frontend tests？

所以我加入 changed-scope classifier：

```text
Backend 變更    → Backend
Frontend 變更   → Frontend
文件變更        → 略過昂貴的產品測試
CI／Release     → Backend + Frontend
Shared contract → Backend + Frontend
無法判斷        → Backend + Frontend
```

Pull Request 使用三點比較：

```text
merge-base...PR-head
```

Main push 使用這次真正進入 main 的兩點比較：

```text
before..head
```

如果 SHA 無效、Git history 不完整或 classifier 發生錯誤，我不會把它
當成沒有變更，而是全部跑。

```python
if diff_is_unknown:
    run_backend = True
    run_frontend = True
```

我寧願多花幾分鐘，也不要因為分類器失敗就漏掉必要測試。

## 測試不用真的睡在那裡

另一個容易被忽略的浪費，是測試真的等待時間經過。

例如 keepalive 測試為了跨過五秒期限，真的等待十幾秒。但我要驗證的
是 timeout 規則，不是電腦能不能等五秒。

後來我改成推進測試使用的 monotonic clock，同時保留真實 socket、TLS、
Authentication isolation 和 Connection reuse。

單次結果：

```text
12.44 秒 → 1.73 秒
```

Linux 完整測試也從 196.36 秒降到 174.72 秒。Production timeout 沒有改，
只是測試不用真的睡在那裡。

Retry 測試也使用相同方式。測試環境取消 backoff 等待，但另外保留
production policy test，確認正式程式仍然使用原本的一秒、兩秒
exponential backoff。

四組 retry tests 的本地時間：

```text
15.74 秒 → 1.92 秒
```

## Self-hosted runner 不是免費

Mac runner 解決了 GitHub 分鐘問題，但成本只是換成電力、磁碟空間、VM
維護、Runner recovery、Cache cleanup 和自己的維運時間。

我最後把 Mac 當成 controller，每次 CI 交給獨立 Linux VM：

```text
Mac
├─ Backend Linux runner
└─ Frontend Linux runner
```

Cache 則分成三類：

```text
可以重新產生 → 定期清理
需要短期保留 → 設定 retention
不能遺失     → 不碰
```

Dependency cache、Docker build cache 可以重建。VM image、production
receipt、資料庫 volume 和專案原始碼不能因為顯示 reclaimable 就直接刪除。

## 我最後學到的事情

CI/CD 要變快，不是一直增加 cache，也不是直接刪除測試。

我的順序是：

1. 先量測時間。
2. 找出 critical path。
3. Backend 和 Frontend 並行。
4. Backend 內部再拆 lane。
5. 移除測試中的固定等待。
6. 有變更才執行需要的部分。
7. 無法判斷時全部執行。
8. 最後再處理 runner 和 cache。

我的目標不是少跑測試，而是不要再為完全沒變的東西，重複付出時間、
機器和維護成本。

下一篇會繼續談部署正確性：
[Railway 顯示 SUCCESS，我為什麼還是不讓它上線？](article-02-exact-sha-release.md)
