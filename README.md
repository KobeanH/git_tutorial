# Google Forms + サービスアカウント認証アーキテクチャ

## 概要

Google Formsでファイルをアップロードした際に、サービスアカウントを使用してファイルの所有者を取得し、所有者のドメインからProvider IDを自動判定する仕組み。

---

## システムフロー図

```mermaid
flowchart TD
    Start[顧客がGoogle Formに回答] --> Upload[PDFファイルをアップロード]
    Upload --> Save[顧客のGoogle Driveに保存]
    Save --> Trigger[GAS onFormSubmitトリガー起動]

    Trigger --> AddPerm[サービスアカウントに閲覧権限を付与]
    AddPerm --> SendEvent[Event Listenerにfile_idを送信]

    SendEvent --> Receive[Event Listenerが受信]
    Receive --> GetFile[サービスアカウントでファイル情報を取得]

    GetFile --> GetOwner[ファイル所有者を取得]
    GetOwner --> ExtractDomain[メールアドレスからドメインを抽出]

    ExtractDomain --> Mapping{ドメインマッピング}

    Mapping -->|customer-a.com| ProviderA[Provider ID: provider_a]
    Mapping -->|customer-b.com| ProviderB[Provider ID: provider_b]
    Mapping -->|未登録| Error[エラー: 未登録のドメイン]

    ProviderA --> CallAPI[API Gatewayに送信]
    ProviderB --> CallAPI

    CallAPI --> CreateContract[Scalebaseに契約作成]

    style Start fill:#e1f5ff
    style Upload fill:#e1f5ff
    style GetOwner fill:#fff4e1
    style ExtractDomain fill:#fff4e1
    style Mapping fill:#ffe1e1
    style ProviderA fill:#e1ffe1
    style ProviderB fill:#e1ffe1
    style CreateContract fill:#e1ffe1
```

---

## シーケンス図（正常フロー）

```mermaid
sequenceDiagram
    actor User as 顧客A<br/>(user@customer-a.com)
    participant Form as Google Form
    participant Drive as Google Drive
    participant GAS as Google Apps Script
    participant EL as Event Listener
    participant SA as サービスアカウント<br/>(Scalebase)
    participant API as API Gateway
    participant SB as Scalebase<br/>(provider_a)

    User->>Form: フォームに回答・ファイルアップロード
    Form->>Drive: ファイルを保存
    Note over Drive: 所有者: user@customer-a.com

    Form->>GAS: onFormSubmitトリガー
    GAS->>Drive: サービスアカウントに閲覧権限を付与
    Note over Drive: 権限リスト:<br/>- user@customer-a.com (owner)<br/>- service-account@... (reader)

    GAS->>EL: file_id を送信
    Note over GAS,EL: ペイロードのprovider_idは無視

    EL->>SA: サービスアカウントで認証
    SA->>Drive: files().get(fileId, fields='owners')
    Drive-->>SA: ファイルメタデータ
    Note over Drive,SA: owners: [{<br/>  emailAddress: "user@customer-a.com",<br/>  displayName: "User A"<br/>}]

    SA->>EL: ファイル所有者情報を返す

    EL->>EL: ドメイン抽出<br/>(customer-a.com)
    EL->>EL: マッピング確認<br/>customer-a.com → provider_a

    EL->>API: provider_id=provider_a で送信
    API->>SB: 契約作成リクエスト
    SB-->>API: 契約作成完了
    API-->>EL: 成功レスポンス
    EL-->>GAS: 成功レスポンス
```

---

## 攻撃シナリオ（改ざん防止）

```mermaid
sequenceDiagram
    actor Attacker as 悪意のある顧客A<br/>(user@customer-a.com)
    participant GAS as Google Apps Script<br/>(改ざん可能)
    participant EL as Event Listener
    participant Drive as Google Drive API
    participant Mapping as ドメインマッピング

    Note over Attacker,GAS: 攻撃: GASコードを改ざんして<br/>provider_idをBに書き換え

    Attacker->>GAS: ファイルをアップロード
    GAS->>GAS: コード改ざん<br/>provider_id = "provider_b"

    GAS->>EL: 送信<br/>{file_id: "xxx", provider_id: "provider_b"}

    Note over EL: ペイロードのprovider_idは無視！

    EL->>Drive: files().get(fileId, fields='owners')
    Drive-->>EL: owners: [{emailAddress: "user@customer-a.com"}]

    Note over EL: 所有者情報は改ざん不可能<br/>（Googleが管理）

    EL->>EL: ドメイン抽出: customer-a.com
    EL->>Mapping: customer-a.com を検索
    Mapping-->>EL: provider_a

    Note over EL: 結果: provider_a に決定<br/>（改ざんされたprovider_bは無視）

    EL->>EL: provider_aで処理を継続

    Note over Attacker,EL: ✅ 攻撃失敗<br/>顧客Aは顧客Bになりすませない
```

---

## データフロー図

```mermaid
flowchart LR
    subgraph Customer["顧客環境（編集可能）"]
        Form[Google Form]
        GAS[Google Apps Script]
        Payload["ペイロード<br/>{file_id, provider_id}"]
    end

    subgraph Google["Google（信頼できる）"]
        Drive[Google Drive API]
        FileData["ファイルデータ<br/>owners: [{<br/>  emailAddress,<br/>  displayName<br/>}]"]
    end

    subgraph Scalebase["Scalebase（検証）"]
        EL[Event Listener]
        SA[サービスアカウント]
        Mapping["ドメインマッピング<br/>customer-a.com → provider_a<br/>customer-b.com → provider_b"]
        API[API Gateway]
    end

    Form --> GAS
    GAS --> Payload
    Payload -.->|送信| EL

    EL --> SA
    SA --> Drive
    Drive --> FileData
    FileData -->|所有者情報| EL

    EL --> Mapping
    Mapping -->|Provider ID決定| API

    style Payload fill:#ffe1e1
    style FileData fill:#e1ffe1
    style Mapping fill:#fff4e1

    classDef untrusted fill:#ffe1e1,stroke:#ff0000,stroke-width:2px
    classDef trusted fill:#e1ffe1,stroke:#00ff00,stroke-width:2px

    class Payload untrusted
    class FileData,Mapping trusted
```

---

## セキュリティモデル

```mermaid
graph TB
    subgraph Untrusted["信頼できない領域"]
        GAS[Google Apps Script<br/>顧客が編集可能]
        Payload[ペイロード<br/>provider_id など]
    end

    subgraph TrustBoundary["信頼境界"]
        EL[Event Listener]
    end

    subgraph Trusted["信頼できる領域"]
        DriveAPI[Google Drive API]
        Owner[ファイル所有者情報<br/>改ざん不可能]
        Mapping[ドメインマッピング<br/>Scalebase管理]
    end

    GAS -->|❌ 無視| Payload
    Payload -.->|送信| EL

    EL -->|✅ 使用| DriveAPI
    DriveAPI --> Owner
    Owner --> EL

    EL -->|✅ 使用| Mapping
    Mapping -->|Provider ID決定| EL

    style Untrusted fill:#ffe1e1
    style TrustBoundary fill:#fff4e1
    style Trusted fill:#e1ffe1
```

---

## ドメインマッピングテーブル

| ドメイン | Provider ID | 備考 |
|---------|------------|------|
| customer-a.com | provider_a | 企業ドメイン |
| customer-b.com | provider_b | 企業ドメイン |
| example.com | provider_example | 企業ドメイン |
| gmail.com | ❌ 判定不可 | 個人アカウント |

**個人アカウント（gmail.com）の場合:**
- ドメインベースの判定は不可
- メールアドレス全体でマッピング必要
- 例: `user@gmail.com` → `provider_xxx`

---

## コンポーネント詳細

### 1. Google Apps Script（顧客環境）

**役割:**
- フォーム送信イベントをトリガー
- サービスアカウントに閲覧権限を付与
- Event Listenerにfile_idを送信

**セキュリティ:**
- ❌ 顧客が編集可能
- ❌ ペイロードは信頼できない
- ✅ サービスアカウントへの権限付与は検証可能

### 2. Event Listener（Scalebase）

**役割:**
- file_idを受信
- サービスアカウントでファイル所有者を取得
- ドメインからProvider IDを判定
- API Gatewayに送信

**セキュリティ:**
- ✅ ペイロードのprovider_idは無視
- ✅ Google Drive APIから直接所有者情報を取得
- ✅ ドメインマッピングはScalebase側で管理

### 3. サービスアカウント（Scalebase）

**役割:**
- Google Drive APIにアクセス
- ファイルの所有者情報を取得

**権限:**
- `reader`（閲覧のみ）で十分
- ファイルの編集・削除は不可

### 4. Google Drive API（Google）

**役割:**
- ファイルのメタデータを提供
- 所有者情報を返す

**セキュリティ:**
- ✅ 所有者情報は改ざん不可能
- ✅ Googleが管理

---

## 重要なセキュリティ特性

### ✅ 安全な点

1. **所有者情報は改ざん不可能**
   - Googleが管理
   - GASから変更できない

2. **所有者は必ず1人**
   - 複数の所有者は存在しない
   - 一意に判定可能

3. **ドメインマッピングはScalebase管理**
   - 顧客は変更できない
   - Event Listener側で管理

4. **ペイロードは無視**
   - GASから送信されたprovider_idは使用しない
   - 所有者情報のみを信頼

### ⚠️ 制約

1. **個人アカウント（gmail.com）は判定不可**
   - ドメインが同じため区別できない
   - メールアドレス全体でマッピング必要

2. **権限リストは取得不可**
   - reader権限では403エラー
   - しかし所有者情報だけで判定可能

3. **サービスアカウントへの権限付与が必要**
   - GAS側で必ず実行する必要がある
   - 忘れるとファイルにアクセスできない

---

## まとめ

この仕組みは：
- ✅ 顧客がGASコードを編集しても安全
- ✅ なりすましは不可能
- ✅ Slackと同等のセキュリティレベル
- ✅ シンプルで実装しやすい

**鍵となるポイント:**
ファイルの所有者情報は、Googleが管理する改ざん不可能な情報であり、これを使ってProvider IDを判定することで、顧客が何を送信しても正しいProvider IDに処理される。
