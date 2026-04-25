# scheduler/ — tren job scheduler (in-repo single source of truth)

このディレクトリはジョブスケジューラ `tren` の **唯一の正本** です。
バイナリ (`tren-wrapper` / `qsub` / `qstat` / `qwait` / `qrun` / `qmap` /
`qbind` / `qclone` / `qowner` / `qworkdir` / `qlog` / `qwait-mark` / `qdel` /
`bench_idtree`) はすべて、このディレクトリ配下の Cargo crate からビルド
される。

> 補足: 過去 (Task #14, commit `7fc412c5e`) にこのソースは
> `~/UTILITY/tren/` へ退避され、ここはシムだけになっていた。Task #21
> で agent の隔離環境からも触れるようにするため、その方針を撤回し、
> ソースをリポジトリ内に戻した。

---

## 使い方

```bash
# 毎セッション (PATH を通す + 必要なら自動ビルド)
source artifacts/bitrag/scheduler/env.sh

# 明示的に再ビルドしたい場合
cargo build --release --manifest-path artifacts/bitrag/scheduler/Cargo.toml

# ジョブ投入の例
qsub echo hello       # (cwd に .tren-<uuid>/ が無ければ自動生成)
qstat
qwait <jobid>
```

`env.sh` を source すると、

- `$TREN`        … このディレクトリの絶対パス
- `$TREN_HOME`   … `$TREN` と同値 (互換目的)
- `$PATH`        … `$TREN/target/release` を先頭に追加

がセットアップされる。`target/release/qsub` が無ければ初回 source 時に
`cargo build --release` を自動で走らせる。

## tren は PWD-local

中央デーモンは無い。最初の `qsub` が cwd 配下に `.tren-<uuid>/` を自動
生成し、`tren-wrapper` がそこで起動する。詳細は `src/lib.rs` 冒頭のコメ
ントおよび `tests/smoke.rs` を参照。

## edit → build → test → user の手元へ反映されるパス

このリポジトリが single source of truth なので、流れは単純:

1. `artifacts/bitrag/scheduler/src/...` を編集する (agent でも user でも)
2. `cargo build --release --manifest-path artifacts/bitrag/scheduler/Cargo.toml`
3. `cargo test --release --manifest-path artifacts/bitrag/scheduler/Cargo.toml`
4. commit & push → user 側 (および他 agent 隔離環境) は次回 pull / merge 時に
   そのまま新しいソースを取得する
5. user 側でも `source artifacts/bitrag/scheduler/env.sh` がそのまま動く。
   外部の `~/UTILITY/tren/` を別途 rebuild する必要は **無い**。

旧い `~/UTILITY/tren/` ツリーを使い続けると drift の元になる。残ってい
る場合は `unset TREN_HOME` した上でこの env.sh を source するのが安全。

## 残骸ファイルについて

このディレクトリには Task #14 以前の残骸がいくつか同居している
(`._qsched*-...` ディレクトリ群、`scheduler.sock`、`scheduler.serial`、
`logs/`、`target/` 配下の旧ビルド等)。これらは Task #21 のスコープ外で、
別タスクで掃除予定。`Cargo.toml` / `src/` / `tests/` だけが現行のソース
本体である。
