# luma3ds-stock-err

Luma3DS のエラー処理を純正化したカスタムビルドです。

Luma 独自のエラー画面 (ARM11 例外ダンプ画面、Rosalina の ErrDisp 表示) を廃し、
エラー時の動作を純正ファームウェアに寄せます。Luma 自身の起動エラー表示
(`Failed to apply FIRM patch` など) はそのまま残してあります。

> これは Luma3DS の派生物です。原プロジェクト: Luma3DS (https://github.com/LumaTeam/Luma3DS)
> ライセンスは原プロジェクトの GPLv3 を継承します (`LICENSE` は Luma3DS のものを同梱)。

## バリアント (4 種)

ベース 2 世代 × 仕様 2 種類の計 4 ビルドを公開しています。

| ビルド | ベース | 致命的エラーの表示 | NTR の X+Y メニュー |
|---|---|---|---|
| 13.5 full | master `1441535d` (v13.5 ナイトリー) | 純正 ErrDisp 画面 | ✗ 開かない |
| **13.5 ntr ★迷ったらこれ** | master `1441535d` (v13.5 ナイトリー) | HOME メニューのダイアログ | ○ 開く |
| 13.3.3 full | 安定版 v13.3.3 (`55d694f`) | 純正 ErrDisp 画面 | ✗ 開かない |
| 13.3.3 ntr | 安定版 v13.3.3 (`55d694f`) | HOME メニューのダイアログ | ○ 開く |

**迷ったら `builds/boot_v13.5-1441535_stock_err_ntr.firm` (13.5 ntr) を選んでください。**
新しめのベースで NTR も使え、Luma のエラー画面は出ません。純正 ErrDisp 画面に
こだわる場合だけ full を選ぶ、という整理で十分です。

- **full**: 純正化パッチを全適用した版。カセット抜き・CPU 例外・ERRF throw の表示が純正通りになります。
- **ntr**: PM の変更だけを外した版 (理由は後述)。BootNTRSelector / NTR (ntrviewer 等) を使う場合はこちら。
- リポジトリ直下の `boot.firm` は **13.5 ntr (★迷ったらこれ)** と同一です。迷ったらこれをそのまま SD に入れてください。

### ファイル対応表

| パッチ | ビルド済み boot.firm | SHA-256 |
|---|---|---|
| `stock-err.patch` | `builds/boot_v13.5-1441535_stock_err_full.firm` | `956f9726d1c2e66e65c0751c73820013f5fbcd82bbef90936d40701ecfe88c10` |
| `stock-err_v13.5-1441535_ntr.patch` | `builds/boot_v13.5-1441535_stock_err_ntr.firm` | `50e983688d53fac5df53c8b0e2aa7aff1811f79e555884f74daf0fad210caafd` |
| `stock-err_v13.3.3_full.patch` | `builds/boot_v13.3.3_stock_err_full.firm` | `9b31a1ab99e1674afce4d305bc0efb498091c6a4317d9997b5624b3f64c0df03` |
| `stock-err_v13.3.3_ntr.patch` | `builds/boot_v13.3.3_stock_err_ntr.firm` | `39e75e2acf3828a220086652d8e0f1cb1fe7bcf6743b341eb0b8a92a20f89b80` |

## 作り方

1. Luma3DS をベースのコミット / タグで取得します。**必ずタグ付きのフルクローン**にしてください。
   shallow clone でビルドするとバージョンが `v0.0.0` になり、MCU 内の設定バージョンが
   弾かれて「Save and exit しても毎回設定メニューが開く」状態になります (実踏済み)。
2. 目的のバリアントのパッチを当てます: `git apply <パッチファイル>`
3. devkitPro 環境 (`DEVKITPRO` / `DEVKITARM` を設定、`makerom` と `firmtool` が必要) で `make` します。
4. 生成された `boot.firm` を SD カードのルートに置きます。戻すときは公式リリースの
   `boot.firm` に差し替えるだけです。CTRNAND 側は公式のままにしておくことを推奨します。

検証用の `crash_test/` (3dsx) も同梱しています。A = ARM11 データアボート、
B = svcBreak、X = ERRF throw を発生させて表示を確認できます。

## 純正化の内容 (full 版のパッチ内訳、9 ファイル)

- `k11_extension`: `isExceptionFatal()` を常時 false にし、全例外を純正カーネルのハンドラへ委譲
- `arm9/source/patches.c`: Kernel11 の kernelpanic を bkpt に化けさせるパッチと、
  ERR:F 形式ダンプの強制有効化 (`enableUserExceptionHandlersForCPUExc` の差し替え) を撤去
- `k11_extension/source/main.c`: `ERRF__DumpException` の差し替えを撤去
- `arm9/source/firm.c` / `main.c`: ARM9 例外ハンドラ差し替え・svcBreak・panic の各パッチ呼び出しと、
  起動時の例外ダンプ表示 (`detectAndProcessExceptionDumps`) を撤去
- `sysmodules/pm`: `isTitleLaunchPrevented()` を常時 false にし、純正 ErrDisp アプレット
  (0004003000008A02) の起動ブロックを解除 ← **ntr 版ではこの変更だけを入れない**
- `sysmodules/rosalina`: err:f の提供 (ErrDisp スレッド) を撤去し、`errdisp.c/h` を削除

※ 13.3.3 の `patchLgyK11` には kernelpanic パッチ自体が存在しないため、その分の削除はありません。

## なぜ ntr 版は PM を触らないのか (互換性メモ)

実機での切り分けで確定した事項です。

- PM の `isTitleLaunchPrevented` を常時 false にすると、**NTR の X+Y メニューが開かなくなります**
  (BootNTRSelector は success を表示し、画面もフラッシュしますが、メニューが出ません)。
- 切り分け: arm9 / k11_extension 側の変更だけの版では X+Y は正常。sysmodule 側だけの版で再現し、
  さらに PM 単独で再現、Rosalina 単独では正常でした。
- 関数の形 (序文・`svcGetSystemInfo` 呼び出し・比較計算) を上流とほぼ同一に戻したビルドでも
  X+Y は復活しなかったため、コード形状の問題ではなく「ErrDisp アプレットの起動を防ぐこと自体」が
  NTR の動作条件だと判断しました (NTR の常駐が ErrDisp のタイトル枠と競合するためと推定)。
- そのため ntr 版では ErrDisp アプレットが起動できず、致命的エラー時は純正 ErrDisp 画面ではなく
  **HOME メニューのダイアログ**が表示されます。**X+Y と純正 ErrDisp 画面は両立しません。**
- なお「Luma v13.5 ナイトリーが NTR のフックを壊す」と言われることがありますが、少なくとも本件では
  原因はナイトリーではなく上記の PM 変更でした (PM を上流に戻した 13.5 ビルドで X+Y は正常動作)。

## 検証状況 (いずれも実機 New 3DS LL)

- 13.5 full: 全項目確認済み (2026-11-09)。crash_test の全経路が純正動作、カセット抜きも純正通り。
- 13.5 ntr: 確認済み (2026-11-21)。起動・X+Y・エラー時の HOME ダイアログ表示。
- 13.3.3 full: 起動と X+Y が開かないことを確認。エラー表示は PM の扱いが同一
  (アプレット起動可) の派生ビルドで純正 ErrDisp 画面を確認。
- 13.3.3 ntr: 確認済み (2026-11-21)。起動・X+Y・エラー時の HOME ダイアログ表示。

## 注意

- boot9strap 環境前提の自己責任カスタムビルドです。
- 仕様上、クラッシュダンプ (`crash_dump_*.bin` や `luma/errdisp.txt`) は生成されなくなります。
  それがこのビルドの目的です。
