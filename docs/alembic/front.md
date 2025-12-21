# 關於 Alembic 專案

Alembic 是 SQLAlchemy 作者開發做為資料庫 Migration 的工具。

## Installation

雖然 Alembic 可以安裝到系統全域，但更常見的是將其安裝在虛擬環境的本機環境中，因為它使用的程式庫（例如 SQLAlchemy）和資料庫驅動程式更適合本地安裝。

以下文件僅介紹了一種為專案安裝 Alembic 的方法；安裝方法有很多種。以下文件僅適用於尚未選擇特定項目配置的使用者。

```
uv init alembic_tutorial

cd alembic_tutorial

uv venv
```

接下來，可以使用 `activate` 命令，以便將此新 Python 環境中的所有二進位檔案都放在本機路徑中：

```bash
source .venv/bin/activate
```

我們現在按以下步驟安裝 Alembic：

```bash
uv add alembic
```

安裝程式會將 `alembic` 指令新增至虛擬環境。之後，在該虛擬環境中對 Alembic 的所有操作都將透過使用此命令來完成，例如：

```bash
alembic init alembic
```

## Dependencies

Alembic 的安裝程序會確保安裝 **SQLAlchemy** 以及其他相依性。 Alembic 從 SQLAlchemy 1.4.0 版本開始即可與之相容。

```bash
uv tree
```

結果:


```bash
alembic-tutorial v0.1.0
└── alembic v1.17.2
    ├── mako v1.3.10
    │   └── markupsafe v3.0.3
    ├── sqlalchemy v2.0.45
    │   ├── greenlet v3.3.0
    │   └── typing-extensions v4.15.0
    └── typing-extensions v4.15.0

```

## Versioning Scheme

Alembic 的版本控制方案是基於 SQLAlchemy 的版本控制方案。需要特別注意的是，雖然 Alembic 使用的是三位數字版本控制方案，但它並沒有使用語意化版本控制（SemVer）。在 SQLAlchemy 和 Alembic 的方案中，中間的數字被視為「重要次要版本」（Significant Minor Release），其中可能包含移除先前已棄用的 API，在極少數情況下可能會導致不向後相容。

這表示版本「1.8.0」、「1.9.0」、「1.10.0」、「1.11.0」等均為重大次要版本，其中將包含新的 API 功能，並可能刪除或修改現有功能。

因此，在確定 Alembic 版本時，請鎖定 "major" 和 "minor"，以避免 API 變更。

真正的 "major" 版本更新，例如昇級到 "2.0"，將包括對基礎功能的完全重新設計/重新架構。