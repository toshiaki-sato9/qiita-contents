---
title: eMMC寿命を96%延ばした実測評価とLinuxカーネルパラメータチューニング
tags:
  - Linux
  - kernel
  - IoT
  - 組み込みLinux
  - emmc
private: true
updated_at: '2026-02-28T13:28:01+09:00'
id: 59eab7c641c63841937c
organization_url_name: null
slide: false
ignorePublish: false
---
## はじめに

私はエネルギー管理システムを支えるIoTエッジデバイスの開発を担当しています。

ある大規模IoT導入案件において、eMMCが半年程度で急速に劣化するという問題が発生しました。本記事では、原因の特定から対策の実測評価、Linuxカーネルパラメータによる追加最適化まで、実際の計測データをもとに解説します。

## 問題の背景

本デバイスが採用するeMMC（大容量pSLC、書き込み耐久数万サイクル）の理論寿命は約120TBです。通常の使用であれば数年は持つはずでした。

ところが大規模案件（数千台規模のIO接続）では、約6ヶ月で劣化兆候が現れるという問題が起きました。

実測した書き込みレートを試算すると、

```
2,493 MB / 60秒 = 41.5 MB/秒
41.5 MB/秒 × 86,400秒/日 = 3.5 TB/日
120 TB ÷ 3.5 TB/日 = 約34日（約1ヶ月）
```

最悪ケースでは1ヶ月で限界に達する計算です。

### ファイルの役割

本デバイスのアプリケーションは、接続されたIOごとに2種類のファイルを管理しています。

| ファイル | 役割 | 実際のサイズ |
|---------|------|----------|
| dat ファイル（IO毎に1個） | 各IOの最新計測値・状態を保持 | IOの種類により数十バイト〜数百KBの幅がある |
| `ring_info`（1個） | 全IO分の管理情報をまとめたリングバッファ | 約440KB（IO数 × エントリ固定サイズ） |

:::note
評価ツール `emmc_test` では計測しやすくするためファイルサイズを固定値（dat: 64バイト、info: 440KB）で仮実装しています。dat ファイルの実サイズは IO の種類によって幅がありますが、問題の主因は `ring_info` の書き込み頻度であり、dat ファイルのサイズは評価の結論に影響しません。
:::

### 原因の特定

調査の結果、アプリケーション内で管理情報ファイル（`ring_info`、約440KB）を、接続されている全IOのデータ更新のたびに毎回書き込んでいたことが原因でした。

例えば5,000 IOが頻繁に更新される環境では、

```
440KB × 5,000回/分 × 60分 = 約126GB/時
```

という異常な書き込み量が発生していました。

## 書き込みの仕組みを理解する

対策を評価するにあたって、Linux のファイル書き込みが物理ストレージに到達するまでの経路を把握することが重要でした。

### キャッシュの階層

Linux のファイル I/O には2段階のキャッシュが存在します。

```
アプリケーション
    │ fwrite()
    ▼
[ユーザー空間バッファ]        ← C 標準ライブラリ (stdio) が管理
    │ fflush()                ← バッファをカーネルへ転送（eMMC への物理 I/O は発生しない）
    ▼
[カーネルページキャッシュ]    ← DRAM 上のダーティページとして滞留
    │ ライトバック             ← カーネルが定期的にフラッシュ（遅延書き込み）
    │ ← dirty_writeback_centisecs / dirty_expire_centisecs が制御
    ▼
[eMMC フラッシュ（物理ストレージ）]
```

### fflush() の役割

`fflush()` は C 標準ライブラリが保持するユーザー空間バッファをカーネルへ転送します。この時点では DRAM 上のページキャッシュに乗るだけで、eMMC への物理書き込みは発生しません。このページは**ダーティページ**としてマークされ、後のライトバックを待つ状態になります。

### fsync() の役割

`fsync()` はカーネルに対して、指定ファイルのダーティページを即座に物理ストレージへ書き戻す（**強制ライトバック**）よう指示します。通常のライトバックスケジューラを待たずに同期的に実行されるため、完了後は eMMC へのデータ永続化が保証されます。

ここで**ライトバック**と**インバリデート**の違いを整理しておきます。

| 操作 | 内容 | ページキャッシュの状態 |
|-----|------|-------------------|
| ライトバック | ダーティページの内容を物理ストレージへ書き出す | クリーン（キャッシュは残る） |
| インバリデート | キャッシュエントリを無効化し、次回読み込み時にストレージから再取得させる | 無効（キャッシュは破棄） |

`fsync()` はライトバックであり、完了後もページキャッシュはクリーン状態で残ります。インバリデートは `posix_fadvise(POSIX_FADV_DONTNEED)` などで明示的に行う操作で、今回の問題の直接原因ではありません。eMMC の書き込み量を増大させていたのは、**`fsync()` による毎回の強制ライトバック**です。

### カーネルパラメータとライトバックタイミング

`fsync()` を使わない場合、ダーティページはカーネルのライトバックスケジューラが管理します。

| パラメータ | 役割 |
|-----------|------|
| `dirty_writeback_centisecs` | ライトバックデーモンがダーティページをチェックする間隔 |
| `dirty_expire_centisecs` | この時間を超えたダーティページを次回チェック時に強制ライトバック |

この遅延書き込みの仕組みにより、同じページへの短時間内の複数回書き込みが**1回のライトバックにまとめられます（コアレッシング）**。`expire` を長くするほど、まとめられる書き込みが増え、物理 I/O 回数が減ります。

### 今回の問題との接続

`ring_info` は `fsync()` を呼ぶため、コアレッシングが一切起きずに毎回 eMMC への物理書き込みが発生します。一方、dat ファイルは `fclose()` のみのためページキャッシュに滞留し、カーネルパラメータの影響を受けます。

```c
#define INFO_SIZE (440 * 1024)  // management info buffer size

// ring_info を書き込む関数
int write_info_file(void) {
    char *buf = malloc(INFO_SIZE);
    memset(buf, 0xAB, INFO_SIZE);

    FILE *fp = fopen(INFO_FILE, "w");
    fwrite(buf, 1, INFO_SIZE, fp);

    fflush(fp);            // ユーザー空間バッファ → ページキャッシュ
    fsync(fileno(fp));     // ページキャッシュ → eMMC（強制ライトバック）
    fclose(fp);
    free(buf);

    g_info_write_count++;
    return 0;
}
```

| ファイル | 書き込み経路 | コアレッシング | カーネルパラメータの影響 |
|---------|-----------|------------|---------------------|
| ring_info（440KB） | fflush → fsync → 即時ライトバック | **なし** | **なし** |
| dat ファイル（64バイト） | fflush → ページキャッシュ滞留 → 遅延ライトバック | **あり** | **あり** |

この違いが、後述する評価ツールの設計と結果の解釈に直結します。

## 対策の概要

2段階の対策を実施しました。

| 対策 | 内容 |
|-----|------|
| コード最適化 | `ring_info` の書き込み頻度を「毎IO」から「30秒ごと1回」に変更 |
| カーネルパラメータ最適化 | dirty page の書き込みタイミングを調整 |

### コード最適化

**修正前（mode 0）: 書き込みスレッドが dat ファイル更新のたびに毎回 `ring_info` を fsync**

```c
#define NUM_FILES        5000
#define DAT_SIZE         64    // Each dat file size (bytes)
#define FLUSH_INTERVAL_SEC 30  // Flush interval for mode 1

// 書き込みスレッド（100スレッドで5,000ファイルを分担）
void *write_thread(void *arg) {
    thread_arg_t *targ = (thread_arg_t *)arg;
    char filepath[256];
    char buf[DAT_SIZE];

    while (g_running) {
        for (int id = targ->start_id; id <= targ->end_id && g_running; id++) {
            snprintf(filepath, sizeof(filepath), "%s/%05d.dat", DATA_DIR, id);

            FILE *fp = fopen(filepath, "r+");
            // タイムスタンプを書き込む
            fwrite(buf, 1, DAT_SIZE, fp);
            fflush(fp);
            fclose(fp);  // fsync なし → ページキャッシュ経由

            g_write_count++;

            // Mode 0: dat ファイル更新のたびに毎回 ring_info を書き込む（fsync あり）
            if (targ->mode == 0) {
                write_info_file();  // ← ここが問題: 5,000回/周回 × fsync
            }
        }
        usleep(1000);
    }
    return NULL;
}
```

**修正後（mode 1）: モニタースレッドが 30 秒に 1 回だけ `ring_info` を fsync**

```c
// モニタースレッド: 30秒ごとに1回だけ ring_info を書き込む（mode 1 のみ）
void *monitor_thread(void *arg) {
    int mode = *(int *)arg;
    time_t start_time = time(NULL);

    while (g_running) {
        sleep(1);
        time_t elapsed = time(NULL) - start_time;

        if (elapsed >= FLUSH_INTERVAL_SEC) {  // 30秒経過したら
            int updated = g_files_updated;
            g_files_updated = 0;

            if (mode == 1 && updated > 0) {
                write_info_file();  // ← 30秒に1回だけ fsync
            }
            start_time = time(NULL);
        }
    }
    return NULL;
}
```

修正の本質は、**「dat ファイル更新ごとに fsync」から「30秒タイマーで1回 fsync」への変更**です。dat ファイル自体は引き続き `fclose()` のみ（ページキャッシュ経由）で書き込まれます。

### カーネルパラメータ

今回対象としたパラメータは以下の4つです。

| パラメータ | 役割 | デフォルト |
|-----------|------|---------|
| `vm.dirty_writeback_centisecs` | dirty pages チェック間隔 | 500（5秒） |
| `vm.dirty_expire_centisecs` | この時間を超えた dirty pages を次回チェック時にフラッシュ | 3000（30秒） |
| `vm.dirty_background_ratio` | バックグラウンド書き込み開始のメモリ占有率 | 10% |
| `vm.dirty_ratio` | 強制同期書き込みのメモリ占有率 | 20% |

評価したプロファイルは以下の3つです。

| プロファイル | writeback | expire | bg_ratio | ratio |
|-----------|-----------|--------|----------|-------|
| 安全重視型 | 700（7秒） | 4500（45秒） | 10% | 20% |
| バランス型 | 1000（10秒） | 6000（60秒） | 15% | 30% |
| 最大延命型 | 3000（30秒） | 12000（120秒） | 20% | 40% |

各プロファイルの適用コマンドは以下のとおりです。

```bash
# 現在値の確認
sysctl vm.dirty_writeback_centisecs vm.dirty_expire_centisecs \
       vm.dirty_background_ratio vm.dirty_ratio

# --- 安全重視型（UPSなし環境向け）---
sudo sysctl -w vm.dirty_writeback_centisecs=700
sudo sysctl -w vm.dirty_expire_centisecs=4500
sudo sysctl -w vm.dirty_background_ratio=10
sudo sysctl -w vm.dirty_ratio=20

# --- バランス型（推奨）---
sudo sysctl -w vm.dirty_writeback_centisecs=1000
sudo sysctl -w vm.dirty_expire_centisecs=6000
sudo sysctl -w vm.dirty_background_ratio=15
sudo sysctl -w vm.dirty_ratio=30

# --- 最大延命型（UPS必須）---
sudo sysctl -w vm.dirty_writeback_centisecs=3000
sudo sysctl -w vm.dirty_expire_centisecs=12000
sudo sysctl -w vm.dirty_background_ratio=20
sudo sysctl -w vm.dirty_ratio=40
```

再起動後も設定を維持する場合は `/etc/sysctl.d/` に書き出します。

```bash
# バランス型を永続化する例
sudo tee /etc/sysctl.d/99-emmc-optimize.conf <<EOF
vm.dirty_writeback_centisecs = 1000
vm.dirty_expire_centisecs    = 6000
vm.dirty_background_ratio    = 15
vm.dirty_ratio               = 30
EOF

sudo sysctl -p /etc/sysctl.d/99-emmc-optimize.conf
```

## 評価方法

### Phase 1: コード最適化の効果測定

コード最適化の効果を測定するため、`emmc_test` というツールを実装しました。

- **モード0（修正前）**: 5,000ファイルの更新のたびに毎回 `ring_info` を書き込む（fsync）
- **モード1（修正後）**: 30秒ごとに1回だけ `ring_info` を書き込む（fsync）

```bash
# 修正前の動作をシミュレート（60秒）
sudo ./emmc_test 0 60 100

# 修正後の動作をシミュレート（60秒）
sudo ./emmc_test 1 60 100
```

### Phase 2: カーネルパラメータの効果測定

ここで重要な前提があります。**`emmc_test` が報告する「Total write amount」はアプリケーションレベルのカウントであり、カーネルパラメータを変えても値は変わりません。**

カーネルパラメータが制御するのは「dirty pages がいつ物理フラッシュされるか」であり、これは `/proc/diskstats` の書き込みセクタ数として現れます。

```
# カーネルパラメータの影響が出る場所
アプリ書き込み → ページキャッシュ → 物理フラッシュ
                                      ↑
                               /proc/diskstats で計測
                               ここをカーネルパラメータが制御
```

そのため Phase 2 では、`emmc_test mode 1` をバックグラウンドで動かしながら、`/proc/diskstats` でデバイスへの実書き込み量を計測しました。

```bash
# /proc/diskstats から書き込みセクタ数を取得（外部ツール不要）
sectors_start=$(awk -v dev="mmcblkX" '$3==dev {print $10}' /proc/diskstats)
# ... テスト実行 ...
sectors_end=$(awk -v dev="mmcblkX" '$3==dev {print $10}' /proc/diskstats)

diff_sectors=$((sectors_end - sectors_start))
written_mb=$(awk "BEGIN {printf \"%.2f\", $diff_sectors * 512 / 1024 / 1024}")
```

`iostat` などの外部ツールは不要で、Linux 標準の `/proc/diskstats` のみで計測できます。

### なぜ emmc_test でカーネルパラメータの効果が出ないのか

`ring_info` ファイルへの書き込みには `fsync()` を使っているため、ページキャッシュを完全にバイパスします。一方、dat ファイル（64バイト）は `fclose()` のみでページキャッシュ経由です。

| ファイル | 書き込み方法 | ページキャッシュ経由 | カーネルパラメータの影響 |
|---------|-----------|-----------------|---------------------|
| ring_info（440KB） | `fsync()` | しない | **なし** |
| dat ファイル（64バイト） | `fclose()` のみ | する | **あり** |

mode 0/1 の差（96%削減）は `ring_info` の `fsync()` 回数が支配的なため、カーネルパラメータを変えても「Total write amount」の数値はほとんど変わりません。

## 実測結果

### Phase 1: コード最適化

```
デバイス: mmcblkX（eMMC）
テスト時間: 60秒
スレッド数: 100（5,000ファイルを分担）
```

| モード | Total write amount | 削減率 |
|-------|-------------------|------|
| モード0（修正前） | **2,498.56 MB** | — |
| モード1（修正後） | **85.20 MB** | **96.6%** |

期待値（約2,500 MB / 約85 MB）と一致し、コード最適化の効果を実測で確認できました。

### Phase 2: カーネルパラメータ

`emmc_test mode 1` 実行中の `/proc/diskstats` 計測結果です。

| プロファイル | 実書き込み量 | 書き込みレート | dirty_expire |
|-----------|-----------|-------------|-------------|
| default | 37.46 MB | 0.65 MB/s | 30秒 |
| 安全重視型 | 37.31 MB | 0.67 MB/s | 45秒 |
| バランス型 | 37.34 MB | 0.67 MB/s | 60秒 |
| **最大延命型** | **17.50 MB** | **0.31 MB/s** | **120秒** |

## 考察

### default / 安全重視型 / バランス型 がほぼ同じ理由

`dirty_expire_centisecs` がテスト時間（60秒）以下のプロファイルでは、dat ファイルの dirty pages がテスト中に全て期限切れになってフラッシュされます。

```
default:    expire 30秒  < テスト 60秒 → テスト中にフラッシュ完了
安全重視型:  expire 45秒  < テスト 60秒 → テスト中にフラッシュ完了
バランス型:  expire 60秒 ≈ テスト 60秒 → テスト終了直前にフラッシュ完了
```

結果として3プロファイルとも実フラッシュ量が揃いました。

### 最大延命型が半分以下になった理由

```
最大延命型: expire 120秒 > テスト 60秒
→ dat ファイルの dirty pages がテスト終了時点で未期限切れ
→ RAM 上に滞留したまま diskstats に現れない
```

17.50 MB の内訳は主に `ring_info` の fsync（2回 × 440KB ≈ 0.86 MB）とファイルシステムのメタデータ・ジャーナルです。

**重要**: この削減は書き込みの「消滅」ではなく「遅延」です。継続稼働する実機では dirty pages は最終的にフラッシュされます。ただし、複数の更新が1回のフラッシュにまとめられる（コアレッシング）効果は実際に寿命延長に寄与します。

### 寿命試算（コード最適化後）

```
コード最適化後: 85 MB / 60秒 = 1.42 MB/秒
1.42 MB/秒 × 86,400秒/日 = 122 GB/日
120 TB ÷ 122 GB/日 ≈ 984日（約2.7年）
```

さらに最大延命型カーネルパラメータを加えた場合（50%削減想定）：

```
61 GB/日
120 TB ÷ 61 GB/日 ≈ 1,967日（約5.4年）
```

## 結論: どのプロファイルを選ぶべきか

コード最適化だけで **96.6%** の削減は既に達成されています。カーネルパラメータはあくまで上乗せです。

| 条件 | 推奨プロファイル | 理由 |
|-----|--------------|------|
| UPS あり | **最大延命型** | 停電時の2分データ欠損リスクをUPSで回避できる |
| UPS なし | **バランス型** | データ欠損リスクを60秒以内に抑えつつ追加効果を得る |

データ欠損リスクの比較：

| プロファイル | 停電時の最大データ欠損 |
|-----------|------------------|
| デフォルト | 最大30秒 |
| バランス型 | 最大60秒 |
| 最大延命型 | 最大120秒 |

### 最大延命型を選ぶ場合の前提条件

`dirty_expire_centisecs=12000`（120秒）に設定すると、ページキャッシュ上のダーティページが最大2分間フラッシュされない状態になります。この間に電源断が発生した場合、RAM 上のデータはすべて失われます。

そのため最大延命型を採用する際は、**電源断・瞬停対策が必須**です。

| 対策 | 内容 |
|-----|------|
| UPS（無停電電源装置） | 停電時に電源を維持し、正常なシャットダウンシーケンスを実行する時間を確保する |
| 瞬停対策（コンデンサ・バッテリー） | 数秒〜数十秒の瞬時電圧低下に対応し、ダーティページのフラッシュ完了まで電源を保持する |
| シャットダウンスクリプト | 電源断検知時に `sync` コマンドを実行してダーティページを強制フラッシュする |

逆に言えば、こうした電源対策が整っている環境であれば、最大延命型は eMMC 寿命を大幅に延ばす有効な選択肢となります。

## まとめ

| 対策 | 書き込み削減率 | 推定eMMC寿命 |
|-----|-------------|-----------|
| 対策なし | — | 約1〜2ヶ月 |
| コード最適化のみ | **96.6%** | **約2.5〜3年** |
| コード最適化 + バランス型 | 約97% | **約3〜4年** |
| コード最適化 + 最大延命型 | 約98% | **約5〜6年** |

今回の調査で得られた最大の知見は、**問題の根本原因はアプリケーションコードにあり、カーネルパラメータはあくまで補助手段**だという点です。

fsync() の有無によって、書き込みがカーネルパラメータの影響を受けるかどうかが根本的に変わります。問題調査の際は、まずアプリケーションの書き込みパスを確認することが重要です。

また、カーネルパラメータの効果測定には、アプリケーションレベルの指標だけでなく、`/proc/diskstats` による物理デバイスレベルの計測が必要です。外部ツール不要でLinux標準機能だけで計測できるため、組み込み環境でも実用的です。

## おわりに

eMMC を採用した IoT エッジデバイスの長期運用では、書き込み量の管理が安定稼働の鍵になります。

本記事で紹介した評価手法（emmc_test による比較計測 + `/proc/diskstats` によるカーネルレベル計測）は、特定の環境に依存せず汎用的に使えます。組み込み Linux デバイスの eMMC 劣化問題に向き合っている方の参考になれば幸いです。

### 注意事項

- 本記事は、一般的な技術知見をもとに個人の経験として執筆しています。
- 製品名・組織名・内部ファイル名など特定可能な固有情報は記載していません。ファイル名・数値は説明用の例示値です。
- 設定値・計測値は実験環境での結果であり、環境によって異なります。

記載内容はあくまで一例であり、すべての IoT 機器・システムにそのまま適用できるものではありません。実際の運用にあたっては、対象環境やリスクレベルに応じた検討が必要です。

### 参考資料

- [Linux kernel documentation - sysctl/vm.txt](https://www.kernel.org/doc/Documentation/sysctl/vm.txt)
- [Linux /proc/diskstats documentation](https://www.kernel.org/doc/Documentation/iostats.txt)
- [eMMC Specification - JEDEC](https://www.jedec.org/standards-documents/docs/jesd84-b51)
