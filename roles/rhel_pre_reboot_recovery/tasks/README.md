# rhel_pre_reboot_recovery
 
OS 再起動前の復旧処理を行うロール。現在は python-urllib3 の `ssl_match_hostname` がシンボリックリンクでなくなる問題への対応のみを実装している。
 
## 必要な変数
 
インベントリの `group_vars` で環境ごとに定義する。
 
| 変数 | 内容 |
| --- | --- |
| `base_directory` | 確認結果ファイルの保存先（対象サーバ上）。存在しない場合は作成する |
| `patch_directory` | パッチ（rpm）の配置先（対象サーバ上） |
 
未定義の場合は、変数の未定義エラーで停止する（事前の定義確認は行わない）。
 
## ファイル構成
 
| ファイル | 内容 |
| --- | --- |
| `main.yml` | 共通手順の実行後、`system_tasks.yml` を呼び出す |
| `system_tasks.yml` | 手順 1〜5 の全処理（個別側は本ファイルのみで完結する） |
 
## 処理内容
 
### 1. 対象判定
 
`patch_directory` 直下（サブディレクトリは対象外）の `python-urllib3-*.rpm` を検索する。
 
- 1 件も無い場合：その旨を表示して処理を終了する（`python3-urllib3` は対象外）
- 1 件以上ある場合：以降の処理を実施する（複数ある場合は全て `rpm` に渡す）
 
### 2. シンボリックリンクの現状確認（before）
 
`/usr/lib/python2.7/site-packages/urllib3/packages/ssl_match_hostname*`（過去の `.bak_*` は除く）を `find` モジュールで取得し、`ll` 相当の内容を `base_directory/python_urllib3_symlink_before` に保存する。
 
- シンボリックリンクでないものがある場合：手順 3 以降を実施する
- 全てシンボリックリンクの場合：「復旧処理は実施しません」と表示して終了する
- 該当ファイルが 1 件も無い場合：再実行時に復旧を続行できるよう、復旧対象として手順 3 以降を実施する（手順 4 の退避対象は無し）
 
### 3. バージョンの現状確認（before）
 
`rpm -qa --qf '%{NAME}-%{VERSION}-%{RELEASE}.%{ARCH}\n' python-urllib3` の結果を `base_directory/python_urllib3_version_before` に保存する。未インストールの場合は「python-urllib3 はインストールされていません」と保存する。
 
### 4. 退避
 
シンボリックリンクでないものを `<パス>.bak_YYYYMMDD` に退避する（root で実施）。
 
- 同日に退避済みの場合は、再実行しても二重退避しない
- 退避後、退避先が全て存在することを `assert` で確認する。存在しない場合はエラー終了する
 
### 5. パッケージ再適用と事後確認
 
1. root で `rpm -Uvh --replacepkgs <patch_directory 内の python-urllib3-*.rpm>` を実行する。失敗した場合はタスクがそのままエラーになる
2. `ssl_match_hostname*` の状態を `base_directory/python_urllib3_symlink_after` に保存し、全てシンボリックリンクであることを確認する。1 件も無い、またはシンボリックリンクでないものがある場合はエラー終了する
3. バージョンを `base_directory/python_urllib3_version_after` に保存し、before から変わっている（上がっている）ことを確認する
   - before と同じ、または after を取得できない場合：エラー終了する
   - before を取得できていない（未インストール）場合：比較せず、after のバージョンを表示して終了する
 
`rpm -Uvh` は `--oldpackage` 無しでは旧バージョンに戻さないため、after が before と異なっていればバージョンが上がったと判定している。
 
## 保存ファイル
 
保存先は全て `base_directory` 配下。拡張子は付けない。
 
| ファイル名 | 内容 |
| --- | --- |
| `python_urllib3_symlink_before` | 再適用前の `ssl_match_hostname*` の状態 |
| `python_urllib3_version_before` | 再適用前の python-urllib3 のバージョン |
| `python_urllib3_symlink_after` | 再適用後の `ssl_match_hostname*` の状態 |
| `python_urllib3_version_after` | 再適用後の python-urllib3 のバージョン |
 
## `command` モジュールを使用している箇所
 
専用モジュールで代替できないため、`changed_when` を指定したうえで使用している。
 
| 処理 | 理由 |
| --- | --- |
| `mv`（退避） | `ansible.builtin` に rename を行うモジュールが無い |
| `rpm -Uvh --replacepkgs` | 同一バージョンの再インストールが必要で、`dnf` / `yum` モジュールでは同等の動作にならない |
| `rpm -qa`（バージョン取得） | 要件として rpm による取得としている（参照のみ。`changed_when: false`） |
 
## 呼び出し元
 
`call_pre_reboot_recovery.yml`

