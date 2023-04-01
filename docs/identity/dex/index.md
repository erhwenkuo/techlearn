# Dex

![](./assets/dex-horizontal-color.png)

Dex 是一種身份服務，它使用 OpenID Connect 來驅動其他應用程序的身份驗證。

Dex 通過“連接器”充當其他身份提供者的門戶。這讓 Dex 將身份驗證推遲到 LDAP 服務器、SAML 提供商或已建立的身份提供商，如 GitHub、Google 和 Active Directory。客戶端編寫一次身份驗證邏輯來與 Dex 對話，然後 Dex 處理給定後端的協議。

