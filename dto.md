# DTO Pattern

DTOパターンを図で説明しますね。DTOパターンを図にしました。重要なポイントを説明します：

## DTOパターンの目的

**DTO (Data Transfer Object)** は、異なる層の間でデータを運ぶための専用オブジェクトです。

## 主な特徴

**1. データの分離**
- **Entity**: データベースの構造を表現（内部情報を含む）
- **DTO**: 外部に公開する情報のみ含む（パスワードなど秘密情報を除外）

**2. 利点**
- セキュリティ向上（機密情報の漏洩防止）
- 疎結合（内部構造の変更が外部に影響しない）
- ネットワーク効率化（必要なデータのみ転送）
- API契約の明確化

**3. 使用例**

```text
Entity（内部）: 
- password, createdAt, updatedAt など全てのフィールド

DTO（外部公開）: 
- id, name, email など公開して良いフィールドのみ
```

この分離により、データベース構造を変更してもAPIの互換性を保ちやすくなります。

```mermaid
graph TB
    subgraph "Handler層 (Presentation)"
        Handler[Handler<br/>http.HandlerFunc]
        UserDTO[UserDTO struct<br/>- ID string json:id<br/>- Name string json:name<br/>- Email string json:email]
        CreateDTO[CreateUserDTO<br/>- Name string json:name<br/>- Email string json:email<br/>- Password string json:password]
    end
    
    subgraph "Service層 (Business Logic)"
        Service[UserService]
        Mapper[Mapper関数<br/>EntToDTO / DTOToEnt]
    end
    
    subgraph "Ent ORM層"
        Client[*ent.Client]
        Schema[User Schema<br/>schema/user.go]
        EntModel[*ent.User<br/>自動生成モデル<br/>- ID int<br/>- Name string<br/>- Email string<br/>- Password string<br/>- CreatedAt time.Time<br/>- UpdatedAt time.Time<br/>- Edges...]
        Query[User Query Builder<br/>client.User.Query]
    end
    
    subgraph "Database"
        DB[(PostgreSQL/MySQL)]
    end
    
    Handler -->|1. リクエスト受信| CreateDTO
    CreateDTO -->|2. DTOを渡す| Service
    Service -->|3. バリデーション| Mapper
    Mapper -->|4. Ent用に変換| Client
    Client -->|5. Create/Update| EntModel
    EntModel -->|6. SQL生成| Query
    Query -->|7. SQL実行| DB
    
    DB -->|8. データ取得| Query
    Query -->|9. *ent.User生成| EntModel
    EntModel -->|10. Entityを返す| Client
    Client -->|11. *ent.User| Service
    Service -->|12. DTOに変換| Mapper
    Mapper -->|13. UserDTO生成| UserDTO
    UserDTO -->|14. JSON返却| Handler
    
    Schema -.->|entc generate| EntModel
    Schema -.->|entc generate| Query
    
    style UserDTO fill:#e1f5ff
    style CreateDTO fill:#e1f5ff
    style EntModel fill:#fff4e1
    style Schema fill:#ffe1f5
    style Handler fill:#f0f0f0
    style DB fill:#ffe1e1
    style Mapper fill:#e1ffe1
```

Go EntでのDTOパターンの実装例を図にしますね。Go EntでのDTOパターンを図にしました！

## Go Entの特徴

**1. Entの自動生成**
- `schema/user.go`でスキーマ定義
- `entc generate`で`*ent.User`モデルと Query Builder を自動生成
- 型安全なクエリビルダーが使える

**2. DTOとの使い分け**

```go
// スキーマ定義 (schema/user.go)
type User struct {
    ent.Schema
}

// 自動生成される Ent モデル
type User struct {
    ID        int
    Name      string
    Email     string
    Password  string  // 🔒 外部に公開したくない
    CreatedAt time.Time
    UpdatedAt time.Time
}

// 手動で作成する DTO
type UserDTO struct {
    ID    string `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
    // Passwordは含めない！
}
```

**3. Mapper関数の例**

```go
// Ent → DTO
func EntToDTO(u *ent.User) UserDTO {
    return UserDTO{
        ID:    strconv.Itoa(u.ID),
        Name:  u.Name,
        Email: u.Email,
    }
}

// DTO → Ent (Create時)
func CreateUser(ctx context.Context, dto CreateUserDTO) (*ent.User, error) {
    return client.User.Create().
        SetName(dto.Name).
        SetEmail(dto.Email).
        SetPassword(hashPassword(dto.Password)).
        Save(ctx)
}
```

Entは内部のデータ管理に特化し、DTOは外部とのやり取りに特化させることで、安全で保守性の高いコードになります。
