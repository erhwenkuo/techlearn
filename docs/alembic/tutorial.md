# 教學

Alembic 是一款用於建立、管理和呼叫關聯式資料庫變更管理腳本的工具，它使用 SQLAlchemy 作為底層引擎。本教程將全面介紹該工具的理論和使用方法。

首先，請確保已安裝 Alembic；安裝部分介紹了一種在本機虛擬環境中安裝的常用方法。如該章節所示，最好將 Alembic 安裝在與目標專案相同的模組/Python 路徑下（通常使用 Python 虛擬環境），這樣，當執行 `alembic` 命令時，`alembic` 呼叫的 Python 腳本（即專案中的 `env.py` 腳本）就可以存取應用程式的模型。雖然這不是絕對必要的，但通常建議這樣做。

以下教學假設 `alembic` 命令列公用程式存在於本機路徑中，並且在呼叫時，將可以存取與目標專案相同的 Python 模組環境。

## 資料庫遷移環境

使用 Alembic 的第一步是建立遷移環境。遷移環境是一個腳本目錄，專門用於特定的應用程式。遷移環境只需創建一次，之後便與應用程式的原始程式碼一起維護。此環境使用 Alembic 的 `init` 命令創建，並可根據應用程式的特定需求進行自訂。

該環境的結構（包括一些產生的遷移腳本）如下所示：

```bash
yourproject/
    alembic.ini
    pyproject.toml
    alembic/
        env.py
        README
        script.py.mako
        versions/
            3512b954651e_add_account.py
            2b1ae634e5cd_add_order_id.py
            3adcc9a56557_rename_username_field.py
```

該目錄包含以下目錄/檔案：

- `alembic.ini` - 這是 Alembic 的主配置文件，所有範本都會產生此文件。稍後將在「編輯 .ini 檔案」部分詳細介紹此文件。
- `pyproject.toml` - 大多數現代 Python 專案都有一個 `pyproject.toml` 檔案。 Alembic 也可以選擇將專案相關的配置儲存在這個檔案中；若要使用 `pyproject.toml` 配置，請參閱「使用 pyproject.toml 進行設定」部分。
- `yourproject` - 這是應用程式原始碼的根目錄，或是其中的某個目錄。
- `alembic` - 此目錄位於應用程式的原始碼樹中，是遷移環境的所在地。它可以任意命名，使用多個資料庫的項目甚至可以擁有多個這樣的目錄。
- `env.py` - 這是一個 Python 腳本，每次呼叫 Alembic 遷移工具時都會執行。它至少包含配置和產生 SQLAlchemy 引擎、從該引擎取得連線和事務，然後使用該連線作為資料庫連線來源來呼叫遷移引擎的指令。
    `env.py` 腳本是產生環境的一部分，因此遷移的運作方式完全可自訂。連接的具體細節以及遷移環境的呼叫方式都包含在此。該腳本可以進行修改，以便操作多個引擎，將自訂參數傳遞給遷移環境，並載入和啟用特定於應用程式的程式庫和模型。

    Alembic 包含一組初始化模板，其中包含針對不同用例的各種 env.py 版本。
- `README` - 包含在各種環境模板中的，應該包含一些有用的信息。
- `script.py.mako` - 這是一個 [Mako](http://www.makotemplates.org/) 範本文件，用於產生新的遷移腳本。此文件中的所有內容都將用於在 `versions/` 目錄下產生新文件。此範本檔案支援腳本控制，因此可以控制每個遷移檔案的結構，包括每個檔案中包含的標準導入語句，以及 `upgrade()` 和 `downgrade()` 函數的結構變更。

- `versions/` - 此目錄存放各個版本腳本。使用其他移轉工具的使用者可能會注意到，這裡的檔案未使用升序整數，而是採用部分 GUID 的方式。在 Alembic 中，版本腳本的順序取決於腳本內部的指令，理論上可以將版本檔案「拼接」到其他版本檔案之間，從而允許合併來自不同分支的遷移序列，儘管這需要手動謹慎操作。

## 構建環境

在對環境有了基本的了解之後，我們可以使用 `alembic init` 來建立一個環境。這將使用 "generic" 模板建立一個環境：

```bash
cd /path/to/yourproject
source /path/to/yourproject/.venv/bin/activate   # assuming a local virtualenv
alembic init alembic
```

在上述步驟中，呼叫了 `init` 指令來產生一個名為 `alembic` 的遷移目錄：

```bash
Creating directory /path/to/yourproject/alembic...done
Creating directory /path/to/yourproject/alembic/versions...done
Generating /path/to/yourproject/alembic.ini...done
Generating /path/to/yourproject/alembic/env.py...done
Generating /path/to/yourproject/alembic/README...done
Generating /path/to/yourproject/alembic/script.py.mako...done
Please edit configuration/connection/logging settings in
'/path/to/yourproject/alembic.ini' before proceeding.
```

上述佈局是使用名為 "generic" 的佈局範本產生的。 Alembic 也包含其他環境範本。可以使用 `list_templates` 指令列出這些模板：

```bash
$ alembic list_templates

Available templates:

generic - Generic single-database configuration.
pyproject - pep-621 compliant configuration that includes pyproject.toml
async - Generic single-database configuration with an async dbapi.
multidb - Rudimentary multi-database configuration.

Templates are used via the 'init' command, e.g.:

  alembic init --template generic ./scripts
```

## 編輯 `.ini` 文件

Alembic 已在目前目錄中放置了一個名為 `alembic.ini` 的檔案。 Alembic 在執行任何其他命令時都會在目前目錄中尋找此檔案；若要指定其他位置，可以使用 `--config` 選項，或設定 `ALEMBIC_CONFIG` 環境變數。

使用 "generic" 範本所建立的 `.ini` 檔案如下圖所示：

```ini
# A generic, single database configuration.

[alembic]
# path to migration scripts.
# this is typically a path given in POSIX (e.g. forward slashes)
# format, relative to the token %(here)s which refers to the location of this
# ini file
script_location = %(here)s/alembic

# template used to generate migration file names; The default value is %%(rev)s_%%(slug)s
# Uncomment the line below if you want the files to be prepended with date and time
# file_template = %%(year)d_%%(month).2d_%%(day).2d_%%(hour).2d%%(minute).2d-%%(rev)s_%%(slug)s

# sys.path path, will be prepended to sys.path if present.
# defaults to the current working directory.
prepend_sys_path = .

# timezone to use when rendering the date within the migration file
# as well as the filename.
# If specified, requires the python>=3.9 or backports.zoneinfo library and tzdata library.
# Any required deps can installed by adding `alembic[tz]` to the pip requirements
# string value is passed to ZoneInfo()
# leave blank for localtime
# timezone =

# max length of characters to apply to the
# "slug" field
# truncate_slug_length = 40

# set to 'true' to run the environment during
# the 'revision' command, regardless of autogenerate
# revision_environment = false

# set to 'true' to allow .pyc and .pyo files without
# a source .py file to be detected as revisions in the
# versions/ directory
# sourceless = false

# version location specification; This defaults
# to <script_location>/versions.  When using multiple version
# directories, initial revisions must be specified with --version-path.
# the special token `%(here)s` is available which indicates the absolute path
# to this configuration file.
#
# The path separator used here should be the separator specified by "version_path_separator" below.
# version_locations = %(here)s/bar:%(here)s/bat:%(here)s/alembic/versions

# path_separator (New in Alembic 1.16.0, supersedes version_path_separator);
# This indicates what character is used to
# split lists of file paths, including version_locations and prepend_sys_path
# within configparser files such as alembic.ini.
#
# The default rendered in new alembic.ini files is "os", which uses os.pathsep
# to provide os-dependent path splitting.
#
# Note that in order to support legacy alembic.ini files, this default does NOT
# take place if path_separator is not present in alembic.ini.  If this
# option is omitted entirely, fallback logic is as follows:
#
# 1. Parsing of the version_locations option falls back to using the legacy
#    "version_path_separator" key, which if absent then falls back to the legacy
#    behavior of splitting on spaces and/or commas.
# 2. Parsing of the prepend_sys_path option falls back to the legacy
#    behavior of splitting on spaces, commas, or colons.
#
# Valid values for path_separator are:
#
# path_separator = :
# path_separator = ;
# path_separator = space
# path_separator = newline
#
# Use os.pathsep. Default configuration used for new projects.
path_separator = os

# set to 'true' to search source files recursively
# in each "version_locations" directory
# new in Alembic version 1.10
# recursive_version_locations = false

# the output encoding used when revision files
# are written from script.py.mako
# output_encoding = utf-8

# database URL.  This is consumed by the user-maintained env.py script only.
# other means of configuring database URLs may be customized within the env.py
# file.
# See notes in "escaping characters in ini files" for guidelines on
# passwords
sqlalchemy.url = driver://user:pass@localhost/dbname

# [post_write_hooks]
# This section defines scripts or Python functions that are run
# on newly generated revision scripts.  See the documentation for further
# detail and examples

# format using "black" - use the console_scripts runner,
# against the "black" entrypoint
# hooks = black
# black.type = console_scripts
# black.entrypoint = black
# black.options = -l 79 REVISION_SCRIPT_FILENAME

# lint with attempts to fix using "ruff" - use the module runner, against the "ruff" module
# hooks = ruff
# ruff.type = module
# ruff.module = ruff
# ruff.options = check --fix REVISION_SCRIPT_FILENAME

# Alternatively, use the exec runner to execute a binary found on your PATH
# hooks = ruff
# ruff.type = exec
# ruff.executable = ruff
# ruff.options = check --fix REVISION_SCRIPT_FILENAME

# Logging configuration.  This is also consumed by the user-maintained
# env.py script only.
[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARNING
handlers = console
qualname =

[logger_sqlalchemy]
level = WARNING
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
datefmt = %H:%M:%S
```

Alembic 使用 Python 的 `configparser.ConfigParser` 函式庫來解析 `alembic.ini` 檔案。這裡提供的 `%(here)s` 變數是一個替換值，其值為 `alembic.ini` 檔案本身的絕對路徑。這樣可以確保目錄和檔案的路徑名相對於設定檔所在的位置都是正確的。


如果只使用單一資料庫和通用配置，只需設定 SQLAlchemy URL 即可：

```ini
sqlalchemy.url = postgresql://scott:tiger@localhost/test

sqlalchemy.url = postgresql://dxlab:wistron888@localhost:542/test
```

## 建立遷移腳本

環境準備好後，我們可以使用 `alembic revision` 建立一個新的版本：

```bash
$ alembic revision -m "create account table"

Generating /path/to/yourproject/alembic/versions/1975ea83b712_create_accoun
t_table.py...done
```

產生了一個新檔案 `1975ea83b712_create_account_table.py`。查看該文件內容：

```python
"""create account table

Revision ID: 1975ea83b712
Revises:
Create Date: 2011-11-08 11:40:27.089406

"""

# revision identifiers, used by Alembic.
revision = '1975ea83b712'
down_revision = None
branch_labels = None

from alembic import op
import sqlalchemy as sa

def upgrade():
    pass

def downgrade():
    pass
```

該文件包含一些 header 資訊、當前版本和「降級」版本的識別碼、一些基本的 Alembic 指令導入，以及空的 `upgrade()` 和 `downgrade()` 函數。我們的任務是向 `upgrade()` 和 `downgrade()` 函數中填充指令，以便對資料庫應用一系列變更。通常情況下，`upgrade()` 是必需的，而 `downgrade()` 僅在需要降級功能時才需要，不過最好還是使用。

另一點要注意的是 `down_revision` 變數。 Alembic 正是透過這個變數來決定應用遷移的正確順序。當我們建立下一個版本時，新檔案的 `down_revision` 識別碼將指向目前版本：

每次 Alembic 對 `versions/` 目錄執行操作時，它都會讀取所有文件，並根據 `down_revision` 標識符之間的關聯方式構建一個列表，其中 `down_revision` 為 `None` 的文件代表第一個文件。理論上，如果遷移環境包含數千個遷移，這可能會增加啟動延遲，但實際上，專案無論如何都應該清理舊的遷移（有關如何做到這一點，同時保持完整建置目前資料庫的能力，請參閱 [Building an Up to Date Database from Scratch](https://alembic.sqlalchemy.org/en/latest/cookbook.html#building-uptodate) 部分。

然後我們可以在腳本中新增一些指令，例如新增一個新的表格 "account"：

```python
def upgrade():
    op.create_table(
        'account',
        sa.Column('id', sa.Integer, primary_key=True),
        sa.Column('name', sa.String(50), nullable=False),
        sa.Column('description', sa.Unicode(200)),
    )

def downgrade():
    op.drop_table('account')
```

`create_table()` 和 `drop_table()` 是 Alembic 指令。 Alembic 透過這些指令提供所有基本的資料庫遷移操作，這些指令的設計力求簡潔高效；大多數指令都不依賴現有的 table metadata。它們依賴於一個 global “context”，該上下文指示如何取得資料庫連接（如果有；遷移也可以將 SQL/DDL 指令轉儲到檔案中）以呼叫命令。與其他所有內容一樣，此全域上下文也在 `env.py` 腳本中設定。

所有 Alembic 指令的概述請參閱[操作參考](https://alembic.sqlalchemy.org/en/latest/ops.html#ops)。

## 運行首次遷移

現在我們要運行遷移。假設我們的資料庫完全乾淨，並且尚未進行版本控制。 `alembic upgrade` 指令將執行升級操作，從目前資料庫版本（本例中為 `None`）升級到指定的目標版本。我們可以指定 `1975ea83b712` 作為要升級到的版本，但在大多數情況下，直接指定「最新版本」（`head`）會更簡單：

```bash
$ alembic upgrade head

INFO  [alembic.context] Context class PostgresqlContext.
INFO  [alembic.context] Will assume transactional DDL.
INFO  [alembic.context] Running upgrade None -> 1975ea83b712
```

哇，太棒了！請注意，我們在螢幕上看到的資訊是 `alembic.ini` 中設定的日誌配置的結果——將 alembic 日誌記錄到控制台。

此處的處理過程包括：Alembic 首先檢查資料庫中是否存在名為 `alembic_version` 的表，如果不存在，則建立該表。它會在該表中尋找目前版本（如果有），然後計算從目前版本到請求版本（在本例中為 `head`，已知為 `1975ea83b712`）的路徑。最後，它會呼叫每個檔案中的 `upgrade()` 方法來更新到目標版本。

## 運行第二次遷移

我們再來一個，這樣我們就有更多東西可以嘗試了。我們再次建立一個修訂文件：

```bash
$ alembic revision -m "Add a column"

Generating /path/to/yourapp/alembic/versions/ae1027a6acf_add_a_column.py...
done
```

讓我們編輯這個文件，並在 "account" 表中新增一列：

```python
"""Add a column

Revision ID: ae1027a6acf
Revises: 1975ea83b712
Create Date: 2011-11-08 12:37:36.714947

"""

# revision identifiers, used by Alembic.
revision = 'ae1027a6acf'
down_revision = '1975ea83b712'

from alembic import op
import sqlalchemy as sa

def upgrade():
    op.add_column('account', sa.Column('last_transaction_date', sa.DateTime))

def downgrade():
    op.drop_column('account', 'last_transaction_date')
```

再次進行資料庫遷移 (run to `head`)：

```bash
$ alembic upgrade head

INFO  [alembic.context] Context class PostgresqlContext.
INFO  [alembic.context] Will assume transactional DDL.
INFO  [alembic.context] Running upgrade 1975ea83b712 -> ae1027a6acf
```

我們已將 `last_transaction_date` 欄位新增至資料庫。

## 獲取資訊

透過一些修訂，我們可以了解一下現狀。

首先，我們可以查看當前版本：

```bash
$ alembic current

INFO  [alembic.context] Context class PostgresqlContext.
INFO  [alembic.context] Will assume transactional DDL.
Current revision for postgresql://scott:XXXXX@localhost/test: 1975ea83b712 -> ae1027a6acf (head), Add a column
```

只有當此資料庫的修訂識別碼與主修訂版相符時，才會顯示主修訂版。

我們也可以使用 `alembic history` 查看歷史記錄；`--verbose` 選項（包括 `history`, `current`, `heads` 和 `branches` 等多個命令都接受此選項）將顯示有關每個版本的完整資訊：

```bash
$ alembic history --verbose

Rev: ae1027a6acf (head)
Parent: 1975ea83b712
Path: /path/to/yourproject/alembic/versions/ae1027a6acf_add_a_column.py

    add a column

    Revision ID: ae1027a6acf
    Revises: 1975ea83b712
    Create Date: 2014-11-20 13:02:54.849677

Rev: 1975ea83b712
Parent: <base>
Path: /path/to/yourproject/alembic/versions/1975ea83b712_add_account_table.py

    create account table

    Revision ID: 1975ea83b712
    Revises:
    Create Date: 2014-11-20 13:02:46.257104
```

## 降版

我們可以透過呼叫 `alembic downgrade` 返回初始狀態來示範降級到零的過程，在 Alembic 中，初始狀態稱為 `base`：

```bash
$ alembic downgrade base

INFO  [alembic.context] Context class PostgresqlContext.
INFO  [alembic.context] Will assume transactional DDL.
INFO  [alembic.context] Running downgrade ae1027a6acf -> 1975ea83b712
INFO  [alembic.context] Running downgrade 1975ea83b712 -> None
```

回到原點，然後再次升級版本：

```bash
$ alembic upgrade head

INFO  [alembic.context] Context class PostgresqlContext.
INFO  [alembic.context] Will assume transactional DDL.
INFO  [alembic.context] Running upgrade None -> 1975ea83b712
INFO  [alembic.context] Running upgrade 1975ea83b712 -> ae1027a6acf
```
