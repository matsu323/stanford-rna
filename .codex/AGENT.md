目的：
現行の Stanford RNA 3D Folding Part 2 用 notebook を改善したい．
ただし，改善方針は「過去の公開結果で有効だった要素の模倣だけ」に限定する．
新規性の高い研究的変更は不要．
低〜中工数で，できるだけスコア改善が見込める改造を段階的に実装したい．

重要な前提条件：
1. ネットワークは使えない．実行時に外部API，外部DB検索，オンラインMSA生成，外部ダウンロードは禁止．
2. Kaggle オフライン環境を壊さないこと．使用可能なのは /kaggle/input に事前配置されたファイルのみ．
3. 既存 notebook の submission.csv 形式を厳密に維持すること．
4. 既存の TBM + Protenix ハイブリッドを主軸として残すこと．全面作り直しは禁止．
5. 改造は「公開上位解法の要素を簡略模倣する」範囲に限定すること．
6. すべての変更は ON/OFF フラグ付きで実装し，A/B テスト可能にすること．
7. 各 target について，どのルールで処理されたかログ出力すること．
8. 長時間の外部前処理を前提にしないこと．ただし /kaggle/input に事前配置済みデータを読むのは可．
9. コードは壊れにくさと再現性を優先し，既存関数の入出力互換をできるだけ維持すること．
10. 変更ごとに，差分，追加フラグ，意図をコメントで明示すること．

今回の実装方針：
過去の公開結果から，以下の要素だけを模倣する．
- TBM を主役にする
- Hybrid TBM 的に，局所的に良いテンプレート断片を別テンプレートから持ってくる
- template retrieval を複数化する
- 幾何補完を強化する
- エネルギー風の cheap rerank を入れる
- モデルの多様性向上のために Boltz-2 を補助的に使う
- 部分置換または slot 単位のアンサンブルを行う

実装優先順位：
優先度1：
A. multi-retrieval
B. Hybrid TBM-lite
C. quality-aware stitching
D. cheap physics rerank

優先度2：
E. TBM 採用本数の動的化
F. template diversity-aware selection
G. fixed identity threshold の length/gap aware 化
H. Protenix 出力の異常検知強化

優先度3：
I. Boltz-2 を weak-target 用の補助 branch として追加
J. slot-level ensemble
K. 必要なら loop/gap 領域のみの部分置換を追加

禁止事項：
- オンラインBLAST，オンラインMSA，オンラインembedding検索は禁止
- 新規の大規模学習や fine-tuning は不要
- 既存の Protenix branch を削除しない
- Boltz-2 を全 target の主力モデルとして使わない
- 外部モデルが必要でも，/kaggle/input にない前提では書かない
- 仕様が不明な RNA template / RNA MSA を Boltz-2 に無理に統合しない

--------------------------------------------------
実装タスク詳細
--------------------------------------------------

[Task A] multi-retrieval の追加
目的：
既存の global alignment 一本依存をやめて，template 候補の取りこぼしを減らす．

要件：
1. 既存の global alignment retrieval は残す
2. 追加で local alignment retrieval を入れる
3. さらに relaxed length-band search を入れる
4. 3系統の候補を union して再ランキングする
5. 再ランキング feature は最低限，global_score，local_score，pct_identity，gap_fraction，length_ratio を使う
6. 出力インターフェースは現行維持
7. USE_MULTI_RETRIEVAL フラグを追加

期待効果：
公開上位解法の「複数検索器併用」の簡略模倣．

[Task B] Hybrid TBM-lite の追加
目的：
全長1本テンプレートではなく，局所的に良いテンプレート断片を取り込む．

要件：
1. 現行の全長 TBM を残す
2. local alignment で高信頼な query 区間を抽出する
3. その区間だけ別テンプレートの座標で上書きできるようにする
4. 境界は Kabsch alignment と補間で滑らかにつなぐ
5. full redesign は不要．existing `adapt_template_to_query()` の前後に差し込む
6. USE_HYBRID_TBM_LITE フラグを追加
7. どの residue range をどの template から採用したかログに出す

期待効果：
公開上位解法の Hybrid TBM の低工数版模倣．

[Task C] quality-aware stitching
目的：
長鎖 chunk の継ぎ目品質を改善する．

要件：
1. 既存の `stitch_chunk_coords()` を拡張する
2. overlap ごとに Kabsch 後 RMSD，adjacent distance error，self-collision を計算する
3. 品質の高い chunk を重くする adaptive blend を実装する
4. 既存の線形 ramp は fallback として残す
5. BLEND_MODE フラグで切替可能にする

期待効果：
公開上位解法の「座標マージ改善」の簡略模倣．

[Task D] cheap physics rerank
目的：
生成済み候補から，不自然な構造を下位に送る．

要件：
1. Phase 3 の直前で各候補に cheap score を付ける
2. score は最低限，
   - adjacent distance penalty
   - non-adjacent clash penalty
   - abnormal compactness / expansion penalty
   を含める
3. 候補生成自体は変更しない
4. USE_PHYSICS_RERANK フラグを追加
5. 各 candidate に対し score をログ出力できるようにする

期待効果：
公開上位解法の「エネルギーベース選択」の低工数模倣．

[Task E] TBM 採用本数の動的化
目的：
悪い TBM を減らし，必要な target にだけ Protenix を多く使う．

要件：
1. 固定的な `n_needed = N_SAMPLE - len(preds)` をやめる
2. top1_sim，top2_sim，top1_pct_id，top1-top2 gap，length_ratio，gap_fraction に基づいて
   `decide_tbm_vs_protenix_allocation()` を作る
3. 戻り値は `n_tbm_keep` と `n_protenix_request`
4. USE_DYNAMIC_ROUTING フラグを追加

期待効果：
公開上位解法の routing / mixture 的発想の簡略模倣．

[Task F] template diversity-aware selection
目的：
同系統 template ばかり採用しないようにして，多様性を増やす．

要件：
1. 1本目は最高スコア template
2. 2本目以降は MMR 風に選ぶ
3. score = alpha * query_similarity - beta * similarity_to_selected
4. template 間類似は配列 identity か alignment score で近似
5. USE_DIVERSITY_TEMPLATE_SELECTION フラグを追加

期待効果：
公開上位解法の多様性確保の簡略模倣．

[Task G] identity threshold の length/gap aware 化
目的：
固定 50% 閾値の粗さを減らす．

要件：
1. `required_pct_identity(seq_len, gap_fraction)` を追加
2. 短鎖は緩く，長鎖と高 gap は厳しく
3. 実際に適用した閾値を target ごとにログ出力
4. USE_ADAPTIVE_IDENTITY_THRESHOLD フラグを追加

期待効果：
危険な TBM 採用を減らす軽量改善．

[Task H] Protenix 出力異常検知の強化
目的：
壊れた prediction を提出群から除外する．

要件：
1. `_extract_c1_coords()` の後段に `validate_coords()` を追加
2. 検査項目は NaN/Inf，adjacent distance 異常，bounding box 異常，重複点比率，過度な self-collision
3. 異常時は None 扱いにしてログに理由を書く
4. USE_COORD_VALIDATION フラグを追加

期待効果：
大外れを減らす事故防止．

[Task I] Boltz-2 補助 branch の追加
目的：
公開上位解法の「Boltz を多様性向上に使う」方針を模倣する．
Boltz-2 は主役ではなく，weak-target 用の補助モデルとする．

要件：
1. Boltz-2 は optional branch として追加
2. 入力は RNA single-chain YAML を生成
3. 対象は weak-target のみ
4. 1 target あたり 1〜2 predictions のみ
5. `--use_potentials` の有無で別サンプルを作れるようにする
6. 既存の TBM / Protenix を壊さず，最終スロットに Boltz-2 候補を混ぜる
7. USE_BOLTZ2_AUX フラグを追加
8. /kaggle/input に Boltz-2 実行に必要なモデルやコードがある前提でのみ実装し，なければ graceful skip

注意：
Boltz-2 は現行 docs 上，RNA sequence 入力は可能だが，MSA や templates の説明は protein 中心なので，
RNA custom MSA / RNA template 統合はこの段階では行わない．
あくまで single-sequence RNA predictor として使う．

[Task J] slot-level ensemble
目的：
異なる branch の候補をスロット単位で混ぜる．

要件：
1. slot 1..k は highest-confidence TBM/Protenix
2. slot k+1..m は Boltz-2 や diversity candidate
3. 残りは fallback / variant
4. 将来の部分置換に備えて，candidate ごとに source metadata を保持する
5. USE_SLOT_ENSEMBLE フラグを追加

--------------------------------------------------
出力してほしいもの
--------------------------------------------------
1. 差分コード
2. 追加フラグ一覧
3. 各変更の意図
4. 既存 notebook との互換性
5. target ごとの処理ログ強化
6. 可能なら簡易ベンチ用のログ集計関数

作業方針：
- 一度に全部は変えず，Task A → B → C → D → E → F → G → H → I → J の順に実装する
- 各段階で notebook が壊れず最後まで submission.csv を出せる状態を保つ
- 変更箇所には日本語コメントを残す
- テスト不能な外部依存は仮定しない
追加制約：
- 実行環境は Kaggle notebook オフラインのみ
- apt install, pip install のオンライン取得は禁止
- /kaggle/input に存在しないモデルやDBを前提にしない
- ネットワークアクセスコードを書かない
- shell command はローカルファイルのみ対象
- 実行時間とメモリを悪化させすぎないこと
- 長鎖 target に対しては exponential に重くなる処理を避けること
- 既存の Protenix checkpoint と既存 dataset 構造を前提にすること
