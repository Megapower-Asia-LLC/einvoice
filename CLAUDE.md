# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 專案概述

統一的台灣電子發票 SDK。pnpm monorepo，一個與供應商無關的 core 套件 + 多個供應商轉接器（Amego、ECPay、ezPay、ezPay 跨境、ezReceipt）。所有加值中心都包裝同一份財政部 MIG 4.0 規格，因此 core 把五個核心操作（**開立 / 作廢 / 折讓 / 折讓作廢 / 查詢**）建模一次，每個供應商只是一層薄轉接器。應用程式相依 `InvoiceProvider` 介面，換供應商只需換建構子。

## 常用指令

```bash
pnpm install
pnpm build         # tsup 建置所有套件 → 每套件輸出 ESM + CJS + d.ts
pnpm typecheck     # 各套件 tsc --noEmit
pnpm test          # vitest run（全 monorepo 的離線測試）
pnpm test:watch
pnpm lint          # 等同 tsc -b --noEmit；本專案沒有 ESLint/Prettier
```

跑單一測試 / 單一套件（vitest 設定在 root，`include` 涵蓋 `packages/*/src`）：

```bash
pnpm vitest run packages/einvoice-amego/src/amounts.test.ts   # 單一檔案
pnpm vitest run -t "computes tax"                             # 依測試名稱
pnpm --filter @paid-tw/einvoice-amego exec vitest run         # 單一套件
```

**Live 測試**：各 adapter 有打真實 sandbox 的 `live.test.ts`，預設被 `describe.skipIf` 跳過（CI 與一般測試只跑 MSW 離線測試），需用環境變數開啟（`AMEGO_LIVE` / `ECPAY_LIVE` / `EZPAY_LIVE`）。憑證預設用各 adapter 的公開 sandbox：

```bash
# bash
AMEGO_LIVE=1 pnpm --filter @paid-tw/einvoice-amego exec vitest run live
# PowerShell（本機環境）
$env:AMEGO_LIVE='1'; pnpm --filter @paid-tw/einvoice-amego exec vitest run live
```

> ⚠️ `track.get`（字軌取號）等 live 測試會**真的消耗發票號碼且不可逆**，每次預約 50 號。執行 mutate 類 live 測試前務必清楚後果。

發版用 changesets：`pnpm changeset` → `pnpm version` → `pnpm release`。

## 架構

### core（`@paid-tw/einvoice`）

定義整個契約，不含任何網路程式碼：

- `provider.ts` — `InvoiceProvider` 介面：`name`、`capabilities`，以及五個操作方法。所有方法失敗時 reject 一個 `InvoiceError`。
- `types.ts` — 統一輸入/輸出型別（`IssueInvoiceInput`、`...Result` 等）、`Buyer`、`Carrier`、`InvoiceStatus`、`TaxType`、`InvoiceCategory`。
- `schemas.ts` — 對應上述型別的 Zod schemas，adapter 在送出請求前用它驗證輸入。
- `capabilities.ts` — `Capability` 列舉 + `supports()` / `assertSupports()`。各供應商支援的選用功能不同，呼叫端應在執行期 feature-detect，而不是等呼叫失敗。`assertSupports` 失敗會丟 `UnsupportedCapabilityError`（本身是 `InvoiceError`，code 為 `UNSUPPORTED`）。
- `errors.ts` — 單一 `InvoiceError` 型別 + 正規化的 `InvoiceErrorCode`（`AUTH`/`VALIDATION`/`NOT_FOUND`/`CONFLICT`/`NUMBER_EXHAUSTED`/`NETWORK`/`PROVIDER`/`UNSUPPORTED`/`UNKNOWN`）。adapter 把供應商/MIG 的原始代碼對應到這組，並用 `rawCode`/`rawMessage`/`raw` 保留原始回應。
- `utils.ts` — 金額換算 helper：`composeTaxExclusive`（未稅→含稅摘要）、`splitTaxInclusive`（含稅→拆稅）、`deriveCategory`（依買受人統編判 B2B/B2C）。
- `ubn.ts` / `mobile-barcode.ts` — 統一編號與手機條碼的本地格式/檢查碼驗證。
- `mock.ts` — `MockProvider`：相同驗證邏輯、不發網路請求，供無憑證測試。

### adapters（`@paid-tw/einvoice-<provider>`）

每個 adapter 以 `workspace:*` 相依 core，實作 `InvoiceProvider`，把**統一模型 ⇄ 供應商連線格式**雙向對應。各家差異主要在認證：Amego 用 MD5 簽章、ezPay/ECPay 用 AES、ezReceipt 用 token。加解密一律使用 Node 內建 `node:crypto`，無第三方加密相依。

adapter 內部的標準分層（以 amego 為例）：

- `config.ts` — 設定型別、sandbox 常數、base URL / 重試解析。
- `client.ts` — HTTP 傳輸層：簽章/加密、`fetch` 包裝（可由 `config.fetch` 注入）、錯誤對應、重試、伺服器時間同步（amego 用 `md5(data + time + appKey)` 並有 5 分鐘 time-offset 快取）。
- `provider.ts` — 實作五個統一方法（每個方法第一步都是 `xxxInputSchema.parse(input)`），外加：
  - `raw(path, data)` escape hatch — 直打任意端點。
  - provider-specific extension 命名空間（如 amego 的 `invoice` / `allowances` / `lottery` / `track`），暴露非跨供應商的原生端點，欄位形狀多以「verified live」註記。
- `validation.ts` — 供應商 payload 層級的 Zod schemas（送出前最後一道驗證，可由 `config.validatePayload === false` 關閉）。
- `amounts.ts` — 供應商特有的稅額/稅別換算（統一 `TaxType` ⇄ 供應商數字碼）。
- `index.ts` — 公開 API；除了 `createXxxProvider`，也 re-export 大量 provider-specific 型別與 helper。

### 跨層的關鍵不變式

- **金額一律整數新台幣**。法定欄位永遠以 TWD 申報財政部，連跨境發票也是。外幣只能透過 `currency`（ISO 4217）+ `exchangeRate` **註記**原始幣別；具 `FOREIGN_CURRENCY` 能力的供應商會記錄，其餘供應商收到非 TWD 的 `currency` 會丟 `UNSUPPORTED`，**不可靜默丟棄**。
- **錯誤一律正規化**成 `InvoiceError` + 穩定 `code`，並保留供應商原始代碼/訊息。新增 adapter 時務必把它的錯誤碼對應進 `InvoiceErrorCode`。
- **能力要誠實宣告**。adapter 的 `capabilities` 集合決定 `supports()` 的結果；只實作了部分核心操作的降級 adapter 也要如實反映。

## 程式碼慣例

- **ESM-only，相對 import 必須帶 `.js` 副檔名**（即使原始碼是 `.ts`）— `moduleResolution: Bundler` + `verbatimModuleSyntax` 的要求。型別 import 用 `import type`。
- **TypeScript strict 全開**，且額外啟用 `noUncheckedIndexedAccess`、`noUnusedLocals`、`noUnusedParameters`、`noImplicitOverride` —— 未使用的變數/參數會直接編譯失敗。型別正確性就是這裡的 lint，沒有額外 linter。
- adapter 的每個對外方法都應先用 core 的 input schema `parse()` 驗證，再做欄位對應。
- 測試與原始碼放一起（`*.test.ts`）或放 `src/__tests__/`。離線測試用 **MSW**（`__tests__/server.ts` 提供 `setupServer` 與指向 mock host 的 `testProvider()`）攔截網路邊界；vitest 的 alias 把 `@paid-tw/einvoice` 指向 core 原始碼，所以**測試不需要先 build**。

## 新增一個供應商

1. 建 `packages/einvoice-<provider>/`，`workspace:*` 相依 `@paid-tw/einvoice`，沿用既有 adapter 的 `package.json`（`tsup` build、雙格式 exports）與分層。
2. 實作 `InvoiceProvider`，對應統一模型 ⇄ 供應商欄位，並宣告正確的 `capabilities`。
3. 把供應商錯誤碼對應到 `InvoiceErrorCode`。
4. 用 MSW fixtures 補離線測試，並加一支 env-gated 的 `live.test.ts`。
5. 加一個 changeset。
