<!-- model：OpenAI o3 -->

### Phase 1 ― CLI × JSON（MVP）

――「**何を／どこに／どう書くか**」を一画面ずつ手取り足取り──

---

## A. 開発環境を 10 分で整える

1. **フォルダ作成 & 移動**

   ```bash
   mkdir todo_cli && cd todo_cli
   ```

2. **仮想環境**（自分だけの Python 空間）

   * mac / Linux

     ```bash
     python3 -m venv .venv && source .venv/bin/activate
     ```
   * Windows

     ```powershell
     py -m venv .venv; .\.venv\Scripts\activate
     ```

3. **必要ライブラリを一括インストール**

   ```bash
   pip install --upgrade pip pytest coverage pylint
   ```

4. **Git 初期化 & .gitignore**

   ```bash
   git init
   echo ".venv/\n__pycache__/\ntasks.json" > .gitignore
   ```

---

## B. プロジェクト骨格を 1 分で作る

```text
todo_cli/
├─ src/
│  ├─ models/
│  ├─ repositories/
│  ├─ controllers/
│  └─ cli/
└─ tests/
```

> **tip:** `mkdir -p src/{models,repositories,controllers,cli} tests`

---

## C. ステップ 1：Task クラス（モデル）を作る

`src/models/task.py`

```python
"""
タスク１件を表すクラス。
"""
from __future__ import annotations
from dataclasses import dataclass, field
from datetime import datetime
from uuid import uuid4


ISO = "%Y-%m-%dT%H:%M:%S%z"            # ISO8601 用フォーマット


@dataclass
class Task:
    title: str                          # やること
    due: str | None = None              # 期限 (YYYY-MM-DD) または None
    note: str | None = None             # メモ
    done: bool = False                  # 完了フラグ
    id: str = field(default_factory=lambda: str(uuid4()))
    created_at: str = field(default_factory=lambda: datetime.now().strftime(ISO))
    updated_at: str = field(default_factory=lambda: datetime.now().strftime(ISO))

    def toggle(self) -> None:
        """完了 ⇔ 未完了 を反転して更新日時を付ける"""
        self.done = not self.done
        self.updated_at = datetime.now().strftime(ISO)
```

*ポイント解説*

| 行                | 何をしている？             | 初学者向けメモ  |
| ---------------- | ------------------- | -------- |
| `@dataclass`     | 自動で `__init__` 等を生成 | 手書きコード激減 |
| `uuid4()`        | 一意 ID を作成           | 衝突しないキー  |
| `datetime.now()` | 現在時刻を取得             | 更新日時に使う  |

---

## D. ステップ 2：JSON リポジトリ（I/O）

`src/repositories/json_repo.py`

```python
"""
tasks.json を読み書きするリポジトリ。
"""
import json
from pathlib import Path
from typing import List
from models.task import Task


DATA_FILE = Path("tasks.json")


class JsonRepository:
    def __init__(self) -> None:
        self.tasks: list[Task] = []
        self.load()

    # ---------- public ----------
    def all(self) -> list[Task]:
        return self.tasks

    def save(self) -> None:
        DATA_FILE.write_text(
            json.dumps(
                {"version": 1, "tasks": [t.__dict__ for t in self.tasks]},
                ensure_ascii=False,
                indent=2,
            ),
            encoding="utf-8",
        )

    def load(self) -> None:
        if not DATA_FILE.exists():
            self.tasks = []
            return
        data = json.loads(DATA_FILE.read_text(encoding="utf-8"))
        self.tasks = [Task(**d) for d in data.get("tasks", [])]
```

*ポイント*

* `Path` を使うと OS を気にせずファイル操作できる
* `t.__dict__` で dataclass → dict 変換
* `ensure_ascii=False` で日本語もそのまま保存

---

## E. ステップ 3：Service（ビジネスロジック）

`src/controllers/service.py`

```python
"""
アプリの中心。バリデーションや並び替えをここで行う。
"""
from datetime import datetime
from typing import Optional
from models.task import Task
from repositories.json_repo import JsonRepository


class TaskService:
    def __init__(self) -> None:
        self.repo = JsonRepository()

    # ---- CRUD ----
    def add(self, title: str, due: Optional[str], note: Optional[str]) -> Task:
        if not title.strip():
            raise ValueError("タイトルは必須です")
        if due:
            try:
                datetime.strptime(due, "%Y-%m-%d")
            except ValueError:
                raise ValueError("期限は YYYY-MM-DD で入力してください")
        task = Task(title=title, due=due, note=note)
        self.repo.tasks.append(task)
        self.repo.save()
        return task

    def list(self) -> list[Task]:
        return sorted(self.repo.all(), key=lambda t: (t.done, t.due or "9999"))

    def delete(self, task_id: str) -> None:
        self.repo.tasks = [t for t in self.repo.tasks if t.id != task_id]
        self.repo.save()

    def edit(self, task_id: str, **kwargs) -> Task:
        task = self._find(task_id)
        for key, val in kwargs.items():
            if hasattr(task, key) and val is not None:
                setattr(task, key, val)
        task.updated_at = datetime.now().isoformat()
        self.repo.save()
        return task

    def toggle(self, task_id: str) -> Task:
        task = self._find(task_id)
        task.toggle()
        self.repo.save()
        return task

    # ---- internal ----
    def _find(self, task_id: str) -> Task:
        for t in self.repo.all():
            if t.id == task_id:
                return t
        raise KeyError("タスクが見つかりません")
```

---

## F. ステップ 4：CLI インターフェース

`src/cli/main.py`

```python
"""
argparse で CLI を作成。
使い方例:
  python -m src.cli.main add "牛乳を買う" --due 2025-05-10
"""
import argparse, textwrap
from controllers.service import TaskService

svc = TaskService()

def main() -> None:
    parser = argparse.ArgumentParser(
        prog="todo",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        description=textwrap.dedent(
            """
            シンプル ToDo CLI
            ----------------
            add    : タスク追加
            list   : 一覧表示
            done   : 完了／未完了切替
            edit   : 編集
            delete : 削除
            """
        ),
    )
    sub = parser.add_subparsers(dest="cmd", required=True)

    # add
    p_add = sub.add_parser("add")
    p_add.add_argument("title")
    p_add.add_argument("--due")
    p_add.add_argument("--note")

    # list
    sub.add_parser("list")

    # done
    p_done = sub.add_parser("done")
    p_done.add_argument("id")

    # edit
    p_edit = sub.add_parser("edit")
    p_edit.add_argument("id")
    p_edit.add_argument("--title")
    p_edit.add_argument("--due")
    p_edit.add_argument("--note")

    # delete
    p_del = sub.add_parser("delete")
    p_del.add_argument("id")

    args = parser.parse_args()
    dispatch(args)


def dispatch(a):
    if a.cmd == "add":
        task = svc.add(a.title, a.due, a.note)
        print("追加:", task.id, task.title)
    elif a.cmd == "list":
        for t in svc.list():
            flag = "✔" if t.done else " "
            print(f"[{flag}] {t.id[:8]}  {t.title}  {t.due or ''}")
    elif a.cmd == "done":
        svc.toggle(a.id)
        print("トグル完了")
    elif a.cmd == "edit":
        svc.edit(a.id, title=a.title, due=a.due, note=a.note)
        print("編集完了")
    elif a.cmd == "delete":
        svc.delete(a.id)
        print("削除完了")


if __name__ == "__main__":
    main()
```

> **ショートカットで実行したい?**
> `python -m src.cli.main` を毎回打つのが面倒なら、`todo` という**エイリアス**を作る：
>
> ```bash
> echo "alias todo='python -m src.cli.main'" >> ~/.zshrc  # bashなら ~/.bashrc
> source ~/.zshrc
> ```

---

## G. ステップ 5：単体テストを書く

`tests/test_service.py`

```python
from controllers.service import TaskService

def test_crud():
    svc = TaskService()
    # 追加
    t = svc.add("テスト", None, None)
    assert t.title == "テスト"
    # 編集
    svc.edit(t.id, title="修正")
    assert svc._find(t.id).title == "修正"
    # 完了切替
    svc.toggle(t.id)
    assert svc._find(t.id).done is True
    # 削除
    svc.delete(t.id)
    assert len(svc.repo.all()) == 0
```

実行：

```bash
pytest -q && coverage run -m pytest && coverage report
```

* **目標**：カバレッジ 80 % 以上（失敗ならテストを追加）

---

## H. ステップ 6：お試し実行

```bash
# 追加
todo add "牛乳を買う" --due 2025-05-10

# 一覧
todo list
[ ] 1a2b3c4d  牛乳を買う  2025-05-10

# 完了
todo done 1a2b3c4d
```

`tasks.json` が自動生成され、開くと以下のようになっていれば成功：

```json
{
  "version": 1,
  "tasks": [
    {
      "title": "牛乳を買う",
      "due": "2025-05-10",
      "note": null,
      "done": true,
      "id": "1a2b3c4d-...",
      "created_at": "...",
      "updated_at": "..."
    }
  ]
}
```

---

## I. ステップ 7：コード品質チェック

```bash
pylint src | tee pylint.log
```

* **スコア 9.0 以上**を目指して警告を潰す（変数名・docstring など）。
* 改善後はコミット：

  ```bash
  git add .
  git commit -m "Phase1: CLI + JSON MVP 完成"
  ```

---

## J. 完了チェックリスト（Phase 1）

* [x] `todo add/list/done/edit/delete` が動く
* [x] `tasks.json` でデータが永続化
* [x] `pytest` 緑 & Coverage ≥ 80 %
* [x] `pylint` 9.0+
* [x] リポジトリを GitHub 等へ Push

この７ステップを終えれば **Phase 1＝MVP** はクリアです。
次は **Phase 2（SQLite への置き換え）** へ進みましょう。Good luck!
