# EveryJapan（EJ2）- 認証機能 担当パート

> チームプロジェクト「EveryJapan」における、私（崔考恩）が担当した認証・会員管理機能の抜粋です。

## 担当範囲

| 機能 | 主なファイル |
|------|------------|
| ログイン / ログアウト | `AuthController.java` `AuthService.java` |
| 会員登録 + メール認証 | `AuthService.java` `RegisterRequest.java` |
| パスワード再設定（2段階） | `AuthService.java` `PasswordResetRequest.java` `PasswordResetConfirmRequest.java` |
| パスワードハッシュ化 / トークン生成 | `PasswordUtil.java` |
| ユーザーDB操作 | `UserRepository.java` |

---

## 実装のポイント

### 1. タイミング攻撃への対策

ユーザーが存在しない場合でも、ダミーのBCrypt検証を実行することで  
応答時間を均一にし、IDの存在有無を外部から判断できないようにしました。

```java
if (user == null) {
    // タイミング攻撃防止: 存在しないユーザーでも同じ処理時間にする
    PasswordUtil.verifyPassword("dummy", "$2a$12$000000000000000000000uGTACuPSTOQRhMqaViHUXn4eOJyGkm");
    return new AuthResponse(false, "ユーザー名またはパスワードが正しくありません");
}
```

### 2. エラーメッセージの統一

「IDが存在しない」と「パスワードが違う」を分けると、攻撃者にID存在を教えてしまいます。  
どちらのケースでも同一メッセージを返すことで情報漏洩を防ぎました。

```java
// IDなし・パスワード不一致、どちらも同じメッセージ
return new AuthResponse(false, "ユーザー名またはパスワードが正しくありません");
```

### 3. BCrypt rounds=12 の選択理由

`PasswordUtil.java` では rounds を 12 に設定しています。

- rounds=10 → 約100ms（ブルートフォース攻撃に対して弱い）
- **rounds=12 → 約300ms（セキュリティと応答速度のバランスが最も良い）**
- rounds=14 → 約1秒（ログインが遅すぎる）

```java
private static final int BCRYPT_ROUNDS = 12;

public static String hashPassword(String plainPassword) {
    return BCrypt.hashpw(plainPassword, BCrypt.gensalt(BCRYPT_ROUNDS));
}
```

### 4. パスワード再設定 — 2段階フロー

```
メール入力 → UUID トークン生成（有効期限24h）→ トークン検証 → パスワード変更 → トークン削除
```

使用後にトークンを `null` にすることで、同じトークンの再利用を防止しています。

```java
// パスワード変更後、トークンを即座に削除
user.setResetToken(null);
user.setResetTokenExpiry(null);
userRepository.save(user);
```

### 5. DTO で入口を絞った設計

`User` エンティティを `@RequestBody` に直接使うと、  
クライアントから `role: SUPER_ADMIN` 等を送り込まれるリスクがあります。  
`LoginRequest` / `RegisterRequest` 等のDTOで受け取るフィールドを限定しました。

```java
// NG: エンティティをそのまま受け取る
public ResponseEntity<AuthResponse> login(@RequestBody User user) { ... }

// OK: DTOで必要なフィールドだけ受け取る
public ResponseEntity<AuthResponse> login(@RequestBody LoginRequest request) { ... }
```

### 6. セッション vs JWT

今回はセッション方式を採用しました。  
JWTはスケーラビリティが高い一方、発行済みトークンの即時無効化が困難です。  
単一サーバー環境で強制ログアウト機能が必要だったため、セッションが適切と判断しました。

```java
session.setAttribute("userId", response.getUser().getId());
session.setAttribute(
    HttpSessionSecurityContextRepository.SPRING_SECURITY_CONTEXT_KEY,
    SecurityContextHolder.getContext()
);
```

---

## 技術スタック

- **言語**: Java
- **フレームワーク**: Spring Framework
- **認証**: HttpSession / Spring Security
- **パスワード**: BCrypt（jBCrypt）
- **DB操作**: JPA（EntityManager）
- **トークン生成**: UUID

---

## プロジェクト全体について

このリポジトリは、チームプロジェクト「EveryJapan（EJ2）」の中から  
私が担当した認証・会員管理パートのみを抜粋したものです。

- チーム規模: 5名
- 開発期間: 1ヶ月
- 担当: ログイン・会員登録・パスワード再設定
