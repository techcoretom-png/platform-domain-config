# Calculator Config

控制 App「計算機偽裝模式」相關行為的遠端設定檔。App 啟動時讀取本檔，可在**不發版**的情況下調整引導彈窗、或遠端關閉功能。

- 檔案：`calculator-config.json`
- 讀取方：App（native 在啟動時抓取並快取，App 內各畫面再讀快取）
- 生效時間：App **下次啟動**時讀取（已在 App 內使用中不會即時變更）
- 注意：raw 檔案有 CDN 快取（約數分鐘），改動不會即時反映

---

## 檔案結構

```json
{
  "default": {
    "intro": {
      "enabled": true,
      "title": "Lindungi privasi Anda",
      "body": "Aktifkan Mode Kalkulator agar aplikasi tampil sebagai Kalkulator di layar utama untuk menjaga privasi Anda.",
      "primaryButton": "Ke Pengaturan",
      "secondaryButton": "Nanti Saja"
    },
    "controls": {
      "revertCalculatorMode": { "enabled": false, "version": 1 },
      "revertAppearance":     { "enabled": false, "version": 1 }
    }
  },
  "platforms": {
    "arpy": {
      "controls": {
        "revertCalculatorMode": { "enabled": true, "version": 2 }
      }
    }
  }
}
```

### default vs platforms

- `default`：所有 platform 的基準設定
- `platforms.<shortName>`：
	- shortName使用全小寫
	- 個別 platform 的覆寫，只需填**要改的欄位**，其餘自動繼承 `default`
- App 端會做 deep merge：`finalConfig = merge(default, platforms[shortName])`
- 不需要特別設定的 platform **不用**出現在 `platforms` 裡

---

## 欄位說明

### `intro` — 首次開啟引導彈窗（"Protect your privacy"）

| 欄位 | 型別 | 說明 |
|------|------|------|
| `enabled` | boolean | 是否顯示引導彈窗 |
| `title` | string | 標題文字 |
| `body` | string | 內文說明 |
| `primaryButton` | string | 主按鈕文字（前往設定）|
| `secondaryButton` | string | 次按鈕文字（稍後再說）|

- 文字欄位**省略不填**時，App 會使用內建的多語系預設字串（跟著裝置語言）。**填了**就以本檔為準（固定該字串，不隨裝置語言變）。
- 彈窗每位使用者只會看到一次（看過後不再顯示），與 `version` 無關。

### `controls` — 遠端強制動作（kill switch）

兩個動作，結構相同：

| 動作 | 說明 |
|------|------|
| `revertCalculatorMode` | 強制關閉計算機模式（取消偽裝鎖屏，使用者直接進入 App）|
| `revertAppearance` | 強制還原 App 外觀（icon / 名稱改回正常）|

每個動作的欄位：

| 欄位 | 型別 | 說明 |
|------|------|------|
| `enabled` | boolean | 是否啟用此強制動作 |
| `version` | integer | 指令版本號，用來控制「再觸發一次」（見下）|

---

## `version` 怎麼運作（重要）

`controls` 的動作是**一次性指令**：

- 每台裝置會記住「上次套用過的 `version`」。
- 當 `enabled = true` 且 **`version` 與裝置記錄的不同**時，動作觸發一次，然後裝置記下這個 `version`。
- 之後就算 App 重開、讀到相同的 `version`，**不會**再觸發 —— 尊重使用者後續自己的設定。

### 我要再強制一次怎麼辦？

把 `version` **加 1**（例如 `1 → 2`）。值變了，所有裝置就會再觸發一次。

> 只把 `enabled` 維持 `true` 不改 `version` 是**不會**再觸發的。要再觸發一定要 bump `version`。

### 觸發對象

只影響「該指令第一次被讀到時，當下正開著該功能」的使用者。當下沒開的人不受影響（之後自己開也不會被這個舊 `version` 關掉）。

---

## 常見操作

**① 對所有 platform 強制關閉計算機模式**
```json
"default": {
  "controls": {
    "revertCalculatorMode": { "enabled": true, "version": 2 }   // 從現值 +1
  }
}
```

**② 只對某個 platform 關閉**
```json
"platforms": {
  "ARPY": {
    "controls": {
      "revertCalculatorMode": { "enabled": true, "version": 2 }
    }
  }
}
```

**③ 修改引導彈窗文案（即時，不需 version）**
```json
"default": {
  "intro": { "title": "新標題", "body": "新內文" }
}
```

**④ 關閉引導彈窗**
```json
"default": {
  "intro": { "enabled": false }
}
```

**⑤ 同一個指令想再強制一次** → 把對應 `version` 加 1。

---

## 注意事項

- `version` 一律用**整數**，要再觸發就 +1。請勿改成相同值或往回改。
- `revertAppearance` 觸發後會顯示提示彈窗並**重啟 App**（換回正常 icon），屬正常行為。
- App 取不到本檔（離線 / 失敗）時，使用上次成功讀取的設定；都沒有則使用內建預設，不影響既有功能。
- 修改請走 PR / commit，方便追蹤誰在何時改了什麼。
