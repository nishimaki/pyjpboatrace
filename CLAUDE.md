# CLAUDE.md (fork 作業用)

メッセージは全て分かり易い日本語にする。

## このリポジトリの目的

`hmasdev/pyjpboatrace` の fork。**テレボート投票フローのサイト drift 追従**のみが目的。
データ取得系 (scraper/) は upstream のままで問題なく動いているので触らない。

親プロジェクト (boatrace 予測システム) は `~/aicode/boatrace` にあり、本 fork を
`pyproject.toml` の `[tool.uv.sources]` 経由で依存として参照する予定。

## upstream との関係

- origin: https://github.com/nishimaki/pyjpboatrace (自分の fork)
- upstream: https://github.com/hmasdev/pyjpboatrace
- 作業ブランチ: `fix/site-drift`
- upstream main は 2025-09-23 の v0.5.0 が最新、2026-04-14 時点で drift 修正は**入っていない**

## 既知の drift (2026-04-14 調査)

1. `brpos-predict monitor` から `get_bet_limit()` を呼ぶと `TimeoutException` で落ちる
2. エラー画面の URL: `boatrace.jp/owpc/sp/ibm/sliphttp/signless.jsp?params=...` (スマホ版の予期せぬエラーページ)
3. `const.py:13-19` の `IBMBRACEORJP = owpc/VoteBridgeNew.jsp?param=H0JS00000stContens&kbn=1&voteActionUrl=...` が古い PC 版フロー前提で、現行のテレボート SP 版に届かない疑い
4. 手動ブラウザログイン (`im.mbrace.or.jp` 経由) は成功する → 認証情報は正しい、ログインフローだけが壊れている

## 触ってよい / 触ってはいけない

### 運用方針: 事前手動入金前提 (2026-04-15 確定)

親プロジェクト側で**当日分を毎朝手動でテレボートに入金する**運用にする。
理由:
- 物理的な日次キャップ (入金額 = その日の最大損失額)
- 銀行連携を auto run 中に通さない (連鎖事故防止)
- `BRPOS_MAX_MONTHLY_INVEST` の上に追加の防御層
- **fork 側の実装範囲を 1/3 に削減できる** (deposit/withdraw を作らなくて良い)

### 触ってよい (投票系)
- `pyjpboatrace/const.py` — URL 定数
- `pyjpboatrace/certification.py` — ログイン (130 行)
- `pyjpboatrace/operator/better.py` — 投票 (181 行)
- `pyjpboatrace/operator/static.py` — ログイン後の残高取得 (`get_bet_limit`)
- 対応するテスト

### 実装しない (事前入金前提のため不要)
- `pyjpboatrace/operator/depositor.py` — 入金。手動でやる
- `pyjpboatrace/operator/withdrawer.py` — 出金。手動でやる
- これらの drift 修正は upstream に任せる (我々の運用には不要)

### 触ってはいけない (データ取得系 — 既に動いてる)
- `pyjpboatrace/scraper/**` 全て
- `pyjpboatrace/scraper/_parser/**` 全て
- データ取得 API のテスト

親プロジェクトの `brpos-fetch` が scraper に依存しているので、ここを壊すと
データ取得パイプラインが即死する。

## 作業方針

1. **まず壊れている箇所を特定する** — 実ブラウザでログインフローを追跡し、`const.py` と実物を照合
2. **最小差分で修正する** — URL 1-2 個 + セレクタ数個程度を想定。リファクタは禁止
3. **テストを書く** — 既存テスト (`tests/`) を参考に、新規挙動のユニットテストを追加
4. **動いたら実投票 1 回で検証** — 最小額 100円 で 1 レース、即確認
5. **upstream に Issue + PR** — 動く修正ができた段階で upstream にフィードバック

## テスト・ビルド

```bash
# 環境セットアップ (既存)
uv sync

# テスト
uv run pytest tests/ -v

# 型チェック (upstream が mypy 使ってるか確認してから)
# uv run mypy pyjpboatrace/
```

## 安全ルール

- **実投票テストは必ず最小額** — 100 円 = 1 単位。複数レースに分散しない
- **vote_pass を扱うコードは `__repr__` / pickle を封じた DTO 経由** — upstream の `UserInformation` を直接露出しない
- **検証用の認証情報を git にコミットしない** — `.env` / `.gitignore` 徹底
- **upstream への PR 時は実 ID / PIN を絶対に含めない** — ログ・スクショ全てマスク

## 親プロジェクトへの反映手順 (修正完了後)

1. fork 側で動作確認 + PR を upstream へ
2. 親プロジェクト `~/aicode/boatrace/pyproject.toml` の依存を一時的に fork に切替:
   ```toml
   [tool.uv.sources]
   pyjpboatrace = { git = "https://github.com/nishimaki/pyjpboatrace", branch = "fix/site-drift" }
   ```
3. 親プロジェクト側で `uv sync` → `brpos-predict monitor --auto-vote` の smoke test
4. upstream PR がマージされたら依存を通常の PyPI 版に戻す

---

# 状況詳細 (2026-04-14 夜 時点の完全スナップショット)

次のセッションで説明し直すのは大変で間違うので、ここに全部書き切る。

## 背景ストーリー

### 親プロジェクト (boatrace) の位置づけ
- 場所: `~/aicode/boatrace`
- 作者: nishimaki (MacMini 常駐で運用)
- 目的: 競艇の荒れレース戦略 (3連単 2 点買い) を Mac mini + launchd で自動運用
- 現行運用モード: **通知 → 手動購入**
  - 06:30 スケジュール生成、07:00 常駐監視 (monitor) 起動、レース時間帯に風速・オッズ取得→4 条件判定→Discord 通知
  - 人間 (運用者) が通知を見てスマホのテレボートアプリから手動購入
  - 実績: 2026-04-14 時点 4 戦 2 勝 +1,380 円 (memory `project_operation_202604.md`)
- 戦略 (BL24、24期間Odds<40): 回収率 153.8%、CI[113.7〜198.4%]、421R 判定 [GO]

### 親プロジェクトでの pyjpboatrace の使い方 (現状)
- **使っている**: `scraper/` 系経由で `get_stadiums`, `get_12races`, `get_race_info`,
  `get_odds_trifecta`, `get_race_result`, `get_just_before_info` などのデータ取得
  - これらは v0.5.0 で正常動作中、**触らない**
- **使っていない (現在)**: `bet`, `get_bet_limit`, `deposit`, `withdraw`, `get_vote_history`
  などの投票系 Selenium メソッド
- 投票系は親プロジェクトの `brpos_fetch/prediction/pyjpboatrace_adapter.py` で
  `TelevoteSession` Protocol としてラップされているが、**Step 6c (実 driver)**
  が未実装で `FakeTelevoteSession` で置き換えられている

### 自動投票機能 (auto_voting) の設計状態
設計書: `~/aicode/boatrace/docs/design/auto_voting_design.md`
進捗メモ: `~/.claude/projects/-Users-tosnis-aicode-boatrace/memory/project_auto_voting.md`

完了済:
- Step 1: TelevoteSession Protocol
- Step 2: vote_log SQLite + 楽観ロック + CHECK + schema v3
- Step 3: safety_guards pure 判定層 + SnapshotReader Protocol
- Step 4: FakeTelevoteSession + 例外階層
- Step 5: vote_executor + Post-flight 三段検証
- Step 6a: Keychain credential loader (brpos_userid / brpos_pin / brpos_auth_pass / brpos_vote_pass)
- Step 6b: PyjpboatraceAdapter (factory DI、実 driver 未実装)
- Step 7-pre: ntp_offset / vote_candidate_builder / Notifier.notify_emergency / auto_voting_bootstrap
- Step 7: race_monitor スレッド分離統合 (_VoteWorker, queue, sentinel shutdown)
- Step 8-1〜3: vote_metrics + candidate_log + Discord /voting_status
- Step 9: Chaos test (multiprocess + os._exit、pytest -m chaos)
- Step 10: DOM healthcheck (毎朝 07:00 Discord Bot scheduler)
- **Step 11 (今夜完了)**: Phase 0 dry-run 配線 (`cli.py` に `--auto-vote` フラグ、
  `daily_operation.sh` に `.env BRPOS_DRY_RUN=true` ガード付きで有効化)

未完:
- **Step 6c**: テレボート投票履歴 Selenium 実装 ← **これが本 fork の真の目的**
- Step 12: verify_dry_run.py
- Step 13: Phase 1 移行判定

## 今夜の接続確認で起きたこと (drift 発覚の経緯)

### 目的
ユーザーが「入金だけ試せるか？」と聞いた → 実銭リスク回避のため、まず
`get_bet_limit()` で **login + 残高読み取りだけ** を試すことにした (お金は動かない)。

### 1 回目の試行
```bash
uv run python -c "
from pyjpboatrace import PyJPBoatrace
from pyjpboatrace.user_information import UserInformation
from selenium import webdriver
from brpos_fetch.prediction.credentials import load_credentials_from_keychain

creds = load_credentials_from_keychain()
user = UserInformation(userid=creds.userid, pin=creds.pin,
    auth_pass=creds.auth_pass, vote_pass=creds.vote_pass)
driver = webdriver.Chrome()
try:
    with PyJPBoatrace(driver=driver, user_information=user) as br:
        print(f'balance = {br.get_bet_limit()} 円')
except Exception as e:
    print(f'[FAIL] {type(e).__name__}: {e}')
"
```

結果: `TimeoutException` (詳細メッセージなし、chromedriver のネイティブスタックのみ)。
Chrome 画面には **「予期せぬエラーが発生しました。ログインページへ」** と表示。
URL は `boatrace.jp/owpc/sp/ibm/sliphttp/signless.jsp?params=hk...&Commandation=...&kunumCommand=crossip/slip.htm`
(スマホ版 "signless" (無ログイン) フローにリダイレクトされていた)。

### 原因切り分け
1. **Keychain 値の汚染チェック**: userid=8 / pin=4 / auth_pass=6 / vote_pass=6、
   空白・改行なし → 認証情報は形式的にクリーン
2. **別セッション衝突疑い**: ユーザーが Safari で別途テレボートにログイン中だった
   ことが判明 → ログアウトして 2 回目実行 → **同じ `TimeoutException`**
3. **ライブラリ側の URL 調査**:
   ```
   .venv/lib/python3.11/site-packages/pyjpboatrace/const.py:13-19
   IBMBRACEORJP = ''.join([
       f'{BOATRACEJP_MAIN_URL}',
       'owpc/VoteBridgeNew.jsp?',
       'param=H0JS00000stContens'
       '&kbn=1'
       '&voteActionUrl=/owpc/pc/site/index.html'
   ])
   ```
   `VoteBridgeNew.jsp?param=H0JS00000stContens&kbn=1&voteActionUrl=/owpc/pc/site/index.html`
   という URL を叩いているが、boatrace.jp 側は既に別フロー (SP 版の
   `sliphttp/signless.jsp` 経由？) に移行している模様
4. **Python traceback 付き再実行**:
   ```
   File ".venv/.../pyjpboatrace/operator/static.py", line 58, in get_bet_limit
       WebDriverWait(driver, timeout).until(
           EC.presence_of_element_located((By.ID, 'currentBetLimitAmount'))
       )
   File ".venv/.../selenium/webdriver/support/wait.py", line 121, in until
       raise TimeoutException(message, screen, stacktrace)
   ```
   → `visit_ibmbraceorjp()` 内で `driver.get(IBMBRACEORJP)` → タイトル確認 →
   その後の `presence_of_element_located((By.ID, 'currentBetLimitAmount'))` で
   15 秒タイムアウト。login 画面を抜けても `currentBetLimitAmount` という ID
   の要素に辿り着けない。

### 結論
**`IBMBRACEORJP` が叩く URL が現行 boatrace.jp と整合しない、またはログイン後
のページ構造が変わって `currentBetLimitAmount` ID が存在しなくなっている** と
仮定される。手動ブラウザログイン (`im.mbrace.or.jp`) は別フローで成功するので、
**認証情報でもネットワークでもなく、ライブラリの想定するフロー/DOM が古い**
ことが原因。

### upstream 確認結果
- upstream master (https://github.com/hmasdev/pyjpboatrace) = インストール済み v0.5.0 と完全一致
- 最新 commit: `5a2b8b5` (2025-09-23、約 6.5 ヶ月前)
- 投票フロー drift 関連の open issue: **なし**
- 半年放置気味で、upstream が直す見込みは薄い → 自力 fork で直すしかない

## 再現手順 (drift を確認するため新セッションで再実行できるようにする)

**注意**: 以下はテレボート営業時間内 (8:30〜20:45 目安) で実行すること。
メンテ時間帯だと別のエラーで止まる可能性がある。

```bash
# 1. 親プロジェクトの venv に入る (Keychain ローダーを使うため)
cd ~/aicode/boatrace
source .venv/bin/activate   # もしくは uv run で直接実行

# 2. テレボートに別セッションでログインしていないことを確認
#    (Safari / Chrome / モバイルアプリ全部ログアウト)

# 3. 再現スクリプト
uv run python -c "
import traceback
from pyjpboatrace import PyJPBoatrace
from pyjpboatrace.user_information import UserInformation
from selenium import webdriver
from brpos_fetch.prediction.credentials import load_credentials_from_keychain

creds = load_credentials_from_keychain()
user = UserInformation(userid=creds.userid, pin=creds.pin,
    auth_pass=creds.auth_pass, vote_pass=creds.vote_pass)
driver = webdriver.Chrome()  # headless なしで画面を出す (DOM 観察のため)
try:
    with PyJPBoatrace(driver=driver, user_information=user) as br:
        print('balance:', br.get_bet_limit())
except Exception:
    traceback.print_exc()
"
```

**観察ポイント** (スクリーンショットを撮っておく):
- Chrome にどの URL が表示されるか
- ログインフォームの ID / name 属性
- エラーページの HTML 構造

## 修正方針 (推奨順)

### Step A: 現行フロー調査 (先に必須)
1. Chrome DevTools で手動ログイン (`https://www.boatrace.jp/owpc/pc/login?authAfterUrl=/`)
   を記録し、以下をメモする:
   - ログインフォームの action URL / input name / hidden field
   - ログイン成功後のリダイレクト先 URL
   - 残高表示ページの URL / 残高要素の CSS セレクタ or ID
   - **PC 版と SP 版でフローが別**。どちらを採用するか決める (PC 版の方が pyjpboatrace の元設計に近い)
2. 差分を列挙 (URL 定数 / DOM セレクタ)

### Step B: 最小差分修正
- `pyjpboatrace/const.py` の `IBMBRACEORJP` を現行 URL に差し替え
- `pyjpboatrace/operator/static.py` の `currentBetLimitAmount` ID を現行セレクタに差し替え
- `pyjpboatrace/certification.py` のログインフォーム操作を現行 form 構造に追従
- `pyjpboatrace/operator/better.py` の投票フローは後回し (bet まで到達できてから)

### Step C: テスト
- 既存 `tests/` のログイン関連テストを確認 (mock 使っているかどうか)
- 実 Chrome + 実認証情報でのテストは **別 fixture** で用意 (通常の pytest から外す)
- 最小確認: `get_bet_limit()` が数値を返す

### Step D: 実投票 (最小額、事前入金前提)
- **テスト前に手動で 1,000 円 (1 単位) を入金** (Web ブラウザのテレボートサイトから)
- **100 円 1 単位の trifecta** で 1 レースだけ投票
- `bet(..., {"trifecta": {"1-2-3": 100}})` → 結果確認
- `withdraw()` は使わない、残高 (900 円) はそのまま翌日のテストへ繰越 or 手動精算
- 実投票は営業時間内 + 非ナイター場 (昼レース) を選ぶ (時間を焦らないため)

### Step E: upstream への貢献
- まず Issue を立てて drift を報告
- PR を投げる (受け入れられるかは別問題)

## 親プロジェクトから参照する方法 (修正完了後)

親プロジェクト `~/aicode/boatrace/pyproject.toml`:
```toml
[project]
dependencies = [
    # ... 他の依存 ...
    # 以下を変更
    "pyjpboatrace @ git+https://github.com/nishimaki/pyjpboatrace@fix/site-drift",
]
```
または uv の sources 機能:
```toml
[tool.uv.sources]
pyjpboatrace = { git = "https://github.com/nishimaki/pyjpboatrace", branch = "fix/site-drift" }
```

反映確認:
```bash
cd ~/aicode/boatrace
uv sync
uv run python -c "from pyjpboatrace import PyJPBoatrace; print(PyJPBoatrace.__module__)"
# → pyjpboatrace.pyjpboatrace (git 版に切り替わっていることを確認)
```

## 親プロジェクト側の既知制約 (fork 修正が効くまで動かない機能)

| 機能 | 現状 | 修正後 |
|---|---|---|
| `brpos-predict monitor --auto-vote` with `BRPOS_DRY_RUN=true` | ✅ 動作中 (Fake session) | 変わらず動く |
| `brpos-predict monitor --auto-vote` with `BRPOS_DRY_RUN=false` | ❌ Step 6c 未実装で LIVE 投票不可 | ✅ 実投票可能に |
| `PyjpboatraceAdapter.get_vote_history` | ⏸ Step 6c 保留中 | 実装する必要あり |
| Discord `/voting_status` | ✅ 動作中 (空データ → 明日から dry-run データ) | 変わらず動く |
| 入金 | ❌ 手動 (Web から) | **❌ 手動継続** (運用方針で自動化対象外) |

## セキュリティ (厳守)

- **Keychain の 4 項目は本 fork の git には**絶対にコミットしない
- テストで使う認証情報は `.env.test.local` (gitignore 済み) に書く、もしくは
  環境変数で渡す
- スクリーンショットに userid / pin が写り込んだら必ず編集で消す
- upstream への PR 時、ログ・トレースバックから個人情報を除去する
- 親プロジェクトの `brpos_fetch/prediction/credentials.py:35-81` の
  `TelevoteCredentials` は `__repr__` / pickle / copy を封じている。本 fork の
  `UserInformation` は平文 str をそのまま持つので、**fork 側でも同等の防漏処理**
  を検討する (Low 優先度)

## 引き継ぎチェックリスト (次セッション開始時に確認する)

- [ ] `git status` で `fix/site-drift` ブランチにいる
- [ ] `git log --oneline -3` で upstream 最新 (5a2b8b5) が見える
- [ ] `cat CLAUDE.md` で本ファイルが読み込まれている
- [ ] 親プロジェクト `~/aicode/boatrace/memory/project_auto_voting.md` と
      `docs/design/auto_voting_design.md` を読んだ (背景理解)
- [ ] テレボート営業時間内 (8:30〜20:45) であることを確認
- [ ] 他のブラウザ/アプリでテレボートにログインしていないことを確認
- [ ] Chrome が起動でき、Selenium 4.41 + Selenium Manager で chromedriver 自動取得できることを確認

## 改訂履歴

| 日付 | 内容 |
|---|---|
| 2026-04-14 | 初版、fork clone + fix/site-drift ブランチ作成、drift 発覚の経緯・再現手順・修正方針を全量記録 |
