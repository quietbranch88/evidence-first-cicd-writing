# Railway 顯示 SUCCESS，我為什麼還是不讓它上線？

> CI/CD 實戰系列：**第二篇** ｜
> [第一篇：改一篇文件也跑完整 CI：我的 2,000 分鐘就是這樣沒的](article-01-ci-cost-and-speed.md) ｜
> [Evidence-first CI/CD writing Skill](../SKILL.md)

有一次 TCF Canada 的 Backend 已經部署完成。

Railway 顯示：

```text
status = SUCCESS
```

Backend readiness 也正常：

```text
status = ok
database = reachable
```

CLI 的 deployment message 裡甚至有我預期的 commit SHA。

照一般流程來看，這次部署應該可以繼續了。但我還是讓 Pipeline 停下來，
沒有部署 eval-worker，也沒有發布 Frontend。

原因是 Railway 回傳的：

```text
metadata.commitHash = null
```

我可以證明某個版本正在正常運作，但不能證明它就是剛剛通過 CI 的版本。

## CI 綠不等於正式環境正確

很多 Pipeline 最後只檢查：

- CI 通過
- Provider 顯示 SUCCESS
- Health endpoint 回傳 200
- 網站可以打開

這些條件都很重要，但還是不夠。因為它們沒有回答：正式環境現在執行的
是哪一個 commit？

如果 CI 通過後，main 又多了一個 commit；部署腳本重新 checkout 最新
main，就可能出現：

```text
CI 驗證 A
正式環境部署 B
```

兩邊都顯示成功，但部署出去的 B 根本沒有通過那次 CI。

## Exact-SHA 是什麼

我後來要求整條發布鏈都要對上相同 SHA：

```text
CI SHA
  = Deployment source SHA
  = Railway API SHA
  = Railway Worker SHA
  = Cloudflare marker SHA
  = Release receipt SHA
```

SpeakUp 的 deployment workflow 不會在發布時重新猜最新 main。它使用觸發
該次 CI 的 `workflow_run.head_sha`，先 checkout 那個 exact SHA，再開始部署。

這樣 CI 驗證的是誰，部署的就是誰。

## Health check 證明健康，不證明身份

這是我在 TCF Canada 遇到的狀態：

```text
Railway status       = SUCCESS
Backend /ready       = ok
Database             = reachable
CLI message          = 包含預期 SHA
metadata.commitHash  = null
```

前四項只能證明服務可以運作，不能取代最後一項的版本身份。

所以 verifier 拒絕通過，流程停在：

```text
provider-verification blocked
```

我沒有重複部署 Backend、提前部署 eval-worker、用 CLI message 取代 commit
metadata，也沒有因為 health 正常就假裝完成。

這次雖然沒有順利上線，但 Pipeline 做了它應該做的事：在證據不足時停下來。

## API 和 Worker 可以並行，但不是永遠

Railway API 和 Worker build 很花時間，看起來最直接的方法就是並行。

但第一次從 embedded worker 切換到 external worker 時，順序不能亂：

```text
調整容量
   ↓
Worker verified
   ↓
API 切換 external
```

如果 API 先切換，而 Worker 還沒準備好，背景工作可能沒有正確接手。

只有系統已經穩定在 external topology 時，下一次替換才可以：

```text
API build    ─┐
              ├→ Paired readback
Worker build ─┘
```

兩邊都成功後，才可以繼續 Cloudflare publication。任一邊失敗，都不能寫入
成功的 release receipt。

## Runtime 沒變，不需要重建全部服務

後來我把版本身份拆成兩組：

```text
Runtime digest
- Backend source
- Frontend source
- Schema
- Public runtime configuration

Control digest
- GitHub Actions workflow
- Publisher scripts
- Deployment verifier
- Deployment CLI
```

如果只改 Publisher 或 verifier，Runtime 沒變，就不用重新 Build Railway
和 Cloudflare。

但 Pipeline 仍然要重新確認 Railway API、Worker、Database readiness、
Cloudflare marker、Frontend asset hash、正式網址與 release receipt。

有一次 control-only 驗證在 1 分 32 秒內完成，而且沒有重新發布 Runtime。

這不能寫成完整部署從 13 分鐘降到 1 分鐘。正確說法是：Runtime 沒有
變更，Pipeline 在 1 分 32 秒內完成既有部署的嚴格讀回驗證。

## Release receipt 是做什麼的

發布證據原本散落在 GitHub Actions、Railway、Cloudflare、Health endpoint、
Database readiness 和 Frontend marker。

Release receipt 把這些結果綁在一起：

```text
source SHA
runtime digest
control digest
API deployment
Worker deployment
database readiness
frontend marker
asset hashes
publication result
verification timestamps
```

最後只有兩邊 Provider 身份、健康狀態和正式站讀回都正確，才會記錄：

```text
paired_verified
```

它讓之後的人可以回答：哪個 commit 被發布、哪些服務重新 Build、哪些
服務沿用既有 Runtime、哪些檢查執行過，以及哪些部分仍然沒有驗證。

## 我現在怎麼區分發布狀態

我不再只使用完成或沒完成，而是分成：

```text
Implemented
Tested
Merged
CI green
Deploying
Provider verified
Live readback verified
Customer flow verified
```

因為：

```text
CI 綠        ≠ 已部署
已合併       ≠ 已上線
Railway 成功 ≠ Exact-SHA 正確
Health 200   ≠ Database 正常
網站能開     ≠ Frontend assets 正確
部署成功     ≠ 真實使用者流程完成
```

## 我最後學到的事情

第一篇處理速度與成本，第二篇處理部署正確性。

真正可靠的 CI/CD，需要不重複執行沒有必要的工作，也不能因為追求速度
而跳過身份驗證。

Pipeline 最重要的功能不只是自動部署，而是在證據不足時，知道自己應該
停下來。

回到第一篇：
[改一篇文件也跑完整 CI：我的 2,000 分鐘就是這樣沒的](article-01-ci-cost-and-speed.md)
