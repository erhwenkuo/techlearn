# Generators

生成器負責生成參數，然後將其渲染到模板中：`ApplicationSet` 資源的字段。有關生成器如何使用模板創建 Argo CD `Application` 的示例，請參閱[簡介](../intro.md)。

生成器主要基於它們用來生成模板參數的數據源。例如：列表生成器從文字列表中提供一組參數，集群生成器使用 Argo CD 集群列表作為源，Git 生成器使用 Git 存儲庫中的文件/目錄，等等。

在撰寫本文時，有七種生成器：

- **List generator**: 列表生成器允許您根據集群名稱/URL 值的固定列表將 Argo CD 應用程序定位到集群。
  
- **Cluster generator**: 集群生成器允許您根據在 Argo CD 中定義（並由其管理）的集群列表（包括自動響應來自 Argo CD 的集群添加/刪除事件）將 Argo CD 應用程序定位到集群。
  
- **Git generator**: Git 生成器允許您基於 Git 存儲庫中的文件或基於 Git 存儲庫的目錄結構創建應用程序。

- **Matrix generator**: 矩陣生成器可用於組合兩個獨立生成器的生成參數。

- **Merge generator**: Merge 生成器可用於合併兩個或多個生成器的生成參數。其他生成器可以覆蓋基本生成器的值。

- **SCM Provider generator**: SCM 提供者生成器使用 SCM 提供者（例如 GitHub）的 API 來自動發現組織內的存儲庫。

- **Pull Request generator**: 拉取請求生成器使用 SCMaaS 提供者（例如 GitHub）的 API 來自動發現存儲庫中打開的拉取請求。

- **Cluster Decision Resource generator**: 集群決策資源生成器用於與 Kubernetes 自定義資源交互，這些資源使用自定義資源特定邏輯來決定要部署到哪一組 Argo CD 集群。

如果您不熟悉生成器，請從 List generator 和 Cluster generator 開始。有關更高級的用例，請參閱上面其餘生成器的文檔。

