# Building Custom Python Function Models with MLflow

MLflow 提供了廣泛的預定義模型風格，但在某些情況下，您希望超越這些模型並根據您的需求自訂一些模型。這就是自訂 PyFunc 派上用場的地方。

本教程包含哪些內容？

本指南旨在引導您了解 PyFuncs 的複雜性，說明原因、內容和方式：

- **Named Model Flavors**: 在我們深入自訂領域之前，有必要了解 MLflow 中現有的預定義模型風格。這些預先定義的風格簡化了模型追蹤和部署，但它們可能無法涵蓋所有用例。
- **Custom PyFuncs Demystified**: 自訂 PyFunc 到底是什麼？它與預定義 named flavors 有何不同，您什麼時候想客制 flavor？我們將介紹：
    - **Pre/Post Processing**: 將預處理或後處理步驟集成為模型預測推論 pipeline 的一部分。
    - **Unsupported Libraries**: 也許您正在使用 MLflow 尚不支援的 ML 套件或函式庫。不用擔心，自訂 PyFunc 可以滿足您的需求。
    - **External References**: 透過外部化引用來避免序列化問題並簡化模型部署。
- **Getting Started with Custom PyFuncs**: 我們將從最簡單的範例開始，闡明自訂 PyFunc 所需的核心元件和抽象方法。
- **Tackling Unsupported Libraries**: 為了提高複雜性，我們將示範如何使用自訂 PyFunc 將模型從不受支援的套件或函式庫整合到 MLflow 中。
- **Overriding Default Inference Methods**: 有時，預設設定並不是您想要的。我們將向您展示如何覆蓋模型的 `predict` 方法，例如，使用 `predict_proba` 而不是 `predict`。

在本教程結束時，您將清楚地了解如何利用 MLflow 中的自訂 PyFunc 來滿足特殊需求，確保靈活性而不影響易用性。

