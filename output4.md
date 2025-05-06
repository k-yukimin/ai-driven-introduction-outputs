<!-- model：OpenAI o3 -->

**実装手順の作成**

以下では\*\*要件定義書 v0.9（2025‑05‑04）\*\*をもとに、プログラミング初学者でも迷わず進められるよう *MVP（Phase 1）～ Phase 4* までの実装ロードマップを「手を動かす順番」で整理しました。
「やること → コマンド例 → 期待結果」の三段構成なので、上から順に実行すれば ToDo アプリが段階的に完成します。

---

## 0. 共通準備（全フェーズ共通）

| 手順  | やること                    | コマンド／補足                                                                                                                              |
| --- | ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| 0‑1 | **Python 3.12** をインストール | [https://www.python.org/downloads/](https://www.python.org/downloads/)                                                               |
| 0‑2 | プロジェクト用ディレクトリを作成        | `mkdir todo_cli && cd todo_cli`                                                                                                      |
| 0‑3 | **仮想環境**を作る             | mac/Linux:`python3 -m venv .venv && source .venv/bin/activate`<br>Windows:`py -m venv .venv && .venv\Scripts\activate`               |
| 0‑4 | 最低限のツールを入れる             | `pip install --upgrade pip pytest coverage pylint`                                                                                   |
| 0‑5 | Git 初期化 & .gitignore    | `git init && echo ".venv/\n__pycache__/" > .gitignore`                                                                               |
| 0‑6 | ディレクトリ構成のひな形            | `text<br>todo_cli/  (プロジェクト直下)<br>└─ src/<br>   ├─ models/<br>   ├─ repositories/<br>   ├─ controllers/<br>   └─ cli/<br>tests/<br>` |

---

## Phase 1 ― CLI × JSON（MVP）

### 1. データモデル実装

1. `src/models/task.py` を作る

   ```python
   from dataclasses import dataclass, field
   from datetime import datetime
   from uuid import uuid4

   @dataclass
   class Task:
       title: str
       due: str | None = None
       note: str | None = None
       done: bool = False
       id: str = field(default_factory=lambda: str(uuid4()))
       created_at: str = field(default_factory=lambda: datetime.utcnow().isoformat())
       updated_at: str = field(default_factory=lambda: datetime.utcnow().isoformat())
   ```

### 2. JSON リポジトリ

1. `src/repositories/json_repo.py`
2. ↑で `tasks.json` を読み書きするクラスを実装
   *ポイント：* **with 文**でファイルを開き、`json.dumps(..., indent=2, ensure_ascii=False)` で日本語文字化けを防止。

### 3. サービス（ビジネスロジック）

1. `src/controllers/service.py`

   * `add_task`, `edit_task`, `delete_task`, `toggle_done`, `list_tasks` を実装
   * ここでバリデーション（タイトル空チェック・日付フォーマット）を行う

### 4. CLI インターフェース

1. `src/cli/main.py`

   * `argparse` でサブコマンド `add`, `list`, `done`, `edit`, `delete` を用意
   * `python -m src.cli.main add "牛乳を買う" --due 2025-05-10` のように使える
2. エントリポイントを `pyproject.toml` あるいは `setup.cfg` で設定しておくと `todo` コマンドで呼び出せる

### 5. 単体テスト

1. `tests/test_service.py` を作成
2. `pytest` で CRUD 全関数をテスト
3. カバレッジ：

   ```bash
   pytest --cov=src
   ```

### 6. 静的解析 & コミット

1. `pylint src` のスコアが 9.0 以上になるよう修正
2. ここで **GitHub に初 push**
3. **チェックリスト**

   * [x] `tasks.json` の追加・編集・削除が CLI で動く
   * [x] テスト成功 & Coverage ≧ 80 %

> *2025‑05‑18 までに完了*（要件定義のマイルストーンと対応）

---

## Phase 2 ― SQLite（DAO パターン）

### 1. 依存追加 & DB 初期化

```bash
pip install sqlalchemy
python - <<'PY'
from sqlalchemy import create_engine, Column, String, Date, Boolean
from sqlalchemy.orm import declarative_base
Base = declarative_base()
class Task(Base):
    __tablename__ = "tasks"
    id = Column(String, primary_key=True)
    title = Column(String, nullable=False)
    due = Column(Date)
    note = Column(String)
    done = Column(Boolean, default=False)
    created_at = Column(String)
    updated_at = Column(String)
engine = create_engine("sqlite:///todo.db")
Base.metadata.create_all(engine)
PY
```

### 2. Repository 切り替え

1. `repositories/sqlite_repo.py` を新規実装（SQLAlchemy セッション管理を隠蔽）
2. `controllers/service.py` の依存をインターフェース化して、JSON ↔ SQLite をスワップできるようにする（**依存性注入**）。

### 3. テスト更新

* テスト前後に `todo_test.db` を作り、テスト後に削除する fixture を用意
* CRUD テストを再実行し、既存ケースが全て通ることを確認

> *2025‑06‑01 までに完了*

---

## Phase 3 ― Tkinter GUI（MVC）

### 1. 必要ライブラリ

```bash
pip install tk  # mac は不要、Windows は同梱
```

### 2. 画面設計をコード化

1. `src/views/tk_main.py`

   * `Tk()` → `Listbox` でタスク一覧
   * 「追加」「編集」「削除」「完了」ボタン配置
2. `src/controllers/tk_controller.py`

   * ボタンクリック → Service 呼び出し → Listbox 再描画

### 3. GUI 用テスト

* **手動ユーザビリティテスト**：

  1. 初学者 5 名に触ってもらい、3 クリック以内で完了操作ができるか計測
  2. フィードバックを issue に記録

> *2025‑06‑22 までに β 版完成*

---

## Phase 4 ― PyQt + タグ／検索（MVVM）

1. `pip install pyqt6`
2. Qt Designer で UI プロトタイプを作る
3. `src/views/qt_main.py` で `QTableView`＋`QStandardItemModel`
4. タグ・優先度列をモデルに追加。検索バーの `textChanged` → `proxyModel.setFilterFixedString()`
5. **MVVM**：

   * Model = SQLAlchemy
   * View = .ui で生成したウィジェット
   * ViewModel = Signals/Slots でバインド
6. Phase 1 ～ 3 のテストをすべて動かしつつ、GUI 結合テストを追加（`pytest-qt` 推奨）

> *2025‑07‑20 までに完了*

---

## よくある質問（FAQ）― 詰まったら読む

| 疑問                       | ヒント                                                           |
| ------------------------ | ------------------------------------------------------------- |
| **JSON が文字化け**           | `open(path, "w", encoding="utf-8")` を忘れていないか確認                |
| **CLI で --due を省略したい**   | `argparse` の `default=None` + Service 側で `if due:` 判定         |
| **SQLite の日付が文字列になる**    | `Column(Date)` にして `date.fromisoformat()`/`isoformat()` を使う   |
| **Tkinter の日本語フォントが崩れる** | `root.option_add("*Font", "Yu Gothic 11")` を先頭で指定             |
| **PyQt の UI が重い**        | `QTableView.setModel()` 後に `resizeColumnsToContents()` を呼ぶと改善 |

---

## まとめと次のステップ

1. **Phase ごとに Git タグ**を切り、バージョン管理で学習過程を可視化
2. 公式ドキュメント → 小さなコード → 動くもの → テスト → リファクタリング、の**小さなループ**を回す
3. 余裕があれば **CI（GitHub Actions）** を追加してテスト自動化
4. クラウド同期・モバイル対応は Phase 4 完了後に別ブランチで着手

このロードマップを順に進めれば、\*\*「ローカル完結の CLI アプリ」→「GUI 付き本格 ToDo 管理ツール」\*\*まで、段階的に完成させながら Python／OOP／DB／GUI を一通り体験できます。
実装中に行き詰まったら *テストが赤くなる最小コード* を作って原因を切り分けるのがコツです。
それでは、**まずは Phase 1 の `Task` クラス作成から始めてみてください。Happy coding!**
