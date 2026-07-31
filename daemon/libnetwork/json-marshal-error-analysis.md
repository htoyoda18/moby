# JSONマーシャルエラー処理の欠如 - 詳細分析

## 📋 問題の全体像

`daemon/libnetwork`配下で**10箇所のJSONマーシャルエラーが意図的に無視**されており、linterで警告されている。

---

## 🔍 根本原因の分析

### なぜ`Marshal` → `Unmarshal`という奇妙なパターンが使われているのか？

これらのコードは**UnmarshalJSON関数内**で、永続化されたデータを復元する際に使用されている：

```go
func (ep *Endpoint) UnmarshalJSON(b []byte) (err error) {
    var epMap map[string]any
    json.Unmarshal(b, &epMap)  // まず全体をmapに展開
    
    // map[string]any から構造体へ変換するために、
    // 一度JSONにマーシャルしてから、再度アンマーシャル
    ib, _ := json.Marshal(epMap["ep_iface"])  // ← エラー無視
    _ = json.Unmarshal(ib, &ep.iface)         // ← エラー無視
}
```

### なぜこのパターンが必要なのか？

1. **後方互換性**: 古いバージョンのdaemonが保存したデータを読めるようにするため
2. **フィールドの柔軟な処理**: `map[string]any`として一度受け取り、存在チェックをしてから型変換
3. **ネストした構造体の変換**: `any`型から具体的な構造体型への変換を、JSONを経由して行う

---

## 🚨 重要な発見：開発者のコメント

### [endpoint.go:143-147](daemon/libnetwork/endpoint.go#L143-L147)
```go
// TODO(cpuguy83): So yeah, this isn't checking any errors anywhere.
// Seems like we should be checking errors even because of memory related issues that can arise.
// Alas it seems like given the nature of this data we could introduce problems if we start checking these errors.
//
// If anyone ever comes here and figures out one way or another if we can/should be checking these errors 
// and it turns out we can't... then please document *why*
```

**作者**: Brian Goff (cpuguy83)  
**日付**: 2021年5月28日  
**コミット**: `4b981436fe2` - "Fixup libnetwork lint errors"

### [endpoint_info.go:502-506](daemon/libnetwork/endpoint_info.go#L502-L506)
```go
// TODO(cpuguy83): Linter caught that we aren't checking errors here
// I don't know why we aren't other than potentially the data is not always expected to be right?
// This is why I'm not adding the error check.
//
// In any case for posterity please if you figure this out document it or check the error
```

---

## 📊 エラー無視箇所の分類

### **パターン1: endpoint.go の UnmarshalJSON（8箇所）**

| 行 | フィールド | データ型 | 影響 |
|-----|-----------|---------|------|
| 149 | `ep_iface` | `*EndpointInterface` | ネットワークインターフェース情報の欠損 |
| 152 | `joinInfo` | `*endpointJoinInfo` | ゲートウェイ・ルーティング情報の欠損 |
| 155 | `exposed_ports` | `[]types.TransportPort` | 公開ポート情報の欠損 |
| 160 | `sandbox` | `string` (sandboxID) | コンテナとの紐付け情報の欠損 |
| 238 | `svcAliases` | `[]string` | サービスエイリアスの欠損 |
| 243 | `ingressPorts` | `[]*PortConfig` | Ingressポート設定の欠損 |
| 248 | `myAliases` | `[]string` | DNS別名の欠損 |
| 253 | `dnsNames` | `[]string` | DNS名の欠損 |

**共通点**:
- すべて`UnmarshalJSON`内のデシリアライズ処理
- `map[string]any` → 構造体への変換
- Marshal失敗 = ゼロ値が設定される（暗黙的なフォールバック）

### **パターン2: network.go:188（1箇所）**

```go
func (nwi *NetworkInfo) UnmarshalJSON(b []byte) error {
    // ...
    if v, ok := m["Meta"]; ok {
        b, _ := json.Marshal(v)  // FIXME
        json.Unmarshal(b, &i.Meta)
    }
}
```

**影響**: ネットワークメタデータの欠損

### **パターン3: endpoint_info.go:507（1箇所）**

```go
func (epj *endpointJoinInfo) UnmarshalJSON(b []byte) error {
    if v, ok := epMap["StaticRoutes"]; ok {
        tb, _ := json.Marshal(v)  // FIXME
        _ = json.Unmarshal(tb, &tStaticRoute)
    }
}
```

**影響**: スタティックルート情報の欠損

### **パターン4: driverapi/ipamdata.go:57（1箇所）**

```go
func (i *IPAMData) UnmarshalJSON(b []byte) error {
    if v, ok := m["AuxAddresses"]; ok {
        b, _ := json.Marshal(v)  // FIXME: unsafe type interface{}
        json.Unmarshal(b, &am)
    }
}
```

**影響**: IPAM補助アドレスの欠損

---

## 🔬 エラーが発生する可能性

### `json.Marshal`が失敗するケース

1. **循環参照**: 構造体が自己参照している（libnetworkでは起こりにくい）
2. **マーシャル不可能な型**: `chan`, `func`, `complex128`など
3. **メモリ不足**: 大量データのマーシャル時（開発者が懸念している点）
4. **カスタムMarshalJSONの失敗**: エンベッドされた型がエラーを返す

### 現実的なリスク評価

**発生頻度**: 🟢 **極めて低い**
- データは永続化済みの構造化データ
- すでに一度JSONとしてパース成功している
- 基本的な型（string, int, slice）のみ

**影響度**: 🟡 **中程度**
- 失敗時、フィールドがゼロ値になる
- ネットワーク設定の一部が欠損 → コンテナ起動失敗の可能性
- エラーがサイレント → デバッグ困難

---

## ✅ 同一ファイル内の正しい実装例

### endpoint.go:173-183（エラーをチェックしている）

```go
bytes, err := json.Marshal(tmp)
if err != nil {
    log.G(context.TODO()).Error(err)
    break
}
err = json.Unmarshal(bytes, &pb)
if err != nil {
    log.G(context.TODO()).Error(err)
    break
}
```

**違い**: 
- `generic`フィールド内の動的データ（プラグインから来る）
- より不確実なデータ源のため、エラーチェックを実装

### endpoint.go:387-391（Value()メソッド）

```go
func (ep *Endpoint) Value() []byte {
    b, err := json.Marshal(ep)
    if err != nil {
        return nil  // エラー時はnilを返す
    }
    return b
}
```

**違い**: 
- エンコード方向（Marshal）
- 呼び出し元がエラーハンドリング可能

---

## 🎯 推奨される解決策

### 戦略1: ログ出力（最小限の変更）

**メリット**:
- リスク最小
- 既存の動作を変えない
- デバッグ可能性の向上

**実装例**:
```go
ib, err := json.Marshal(epMap["ep_iface"])
if err != nil {
    log.G(context.TODO()).WithError(err).Warn("failed to marshal ep_iface, using zero value")
}
_ = json.Unmarshal(ib, &ep.iface)
```

### 戦略2: エラー伝播（推奨）

**メリット**:
- データ破損を防ぐ
- 明示的なエラーハンドリング
- 上位レイヤーで対処可能

**実装例**:
```go
ib, err := json.Marshal(epMap["ep_iface"])
if err != nil {
    return fmt.Errorf("failed to marshal ep_iface: %w", err)
}
if err := json.Unmarshal(ib, &ep.iface); err != nil {
    return fmt.Errorf("failed to unmarshal ep_iface: %w", err)
}
```

**影響**: `UnmarshalJSON`がエラーを返すため、呼び出し元でのハンドリングが必要

### 戦略3: 中間構造体を使った正攻法（最良だが大規模）

**最近の正しい例**: コミット `33fc45e5c5` (2025年10月)

```go
type endpointInterfaceJSON struct {
    Address     string `json:"address,omitempty"`
    AddressIPv6 string `json:"addressIPv6,omitempty"`
    MacAddress  string `json:"macAddress,omitempty"`
}

func (epi *EndpointInterface) UnmarshalJSON(data []byte) error {
    var tmp endpointInterfaceJSON
    if err := json.Unmarshal(data, &tmp); err != nil {
        return err
    }
    // 型変換ロジック
    return nil
}
```

**メリット**:
- Marshal/Unmarshalの往復が不要
- 型安全
- パフォーマンス向上

**デメリット**:
- 大規模なリファクタリング
- テストが必要

---

## 📈 優先度評価（再評価）

### 影響度: 🟡 **中程度**
- 通常運用では問題なし
- メモリ不足など極端な状況でサイレント失敗
- デバッグ困難性

### 解決の容易さ: ✅ **容易（戦略1）** ⚠️ **中程度（戦略2）** ❌ **困難（戦略3）**

### 推奨アクション:

**短期（即座に実施可能）**:
1. ✅ 戦略1を全10箇所に適用（1-2時間）
2. ✅ ログレベルはWarnで統一

**中期（次のメジャーバージョン）**:
1. ⚠️ 戦略2でエラー伝播を実装
2. ⚠️ データストアからの復元失敗時のリカバリー戦略を検討

**長期（リファクタリング）**:
1. 🔍 戦略3で中間構造体に移行（EndpointInterface方式）

---

## 🔗 関連情報

- **linter警告**: `errchkjson` - Go 1.16+で追加されたチェック
- **コミット履歴**:
  - `4b981436fe2` (2021-05): linterエラー対応でTODOコメント追加
  - `33fc45e5c5` (2025-10): EndpointInterfaceの正しいリファクタリング例
- **影響を受けるコンポーネント**:
  - Endpoint復元
  - Network復元
  - IPAM設定復元
  - Swarmサービスディスカバリー

---

## 💡 結論

**開発者も迷っている部分**であり、2021年から未解決のまま。
最近の `EndpointInterface` リファクタリング（2025年10月）が、正しいアプローチの参考になる。

**まず戦略1（ログ追加）で可視化し、実際のエラー発生頻度を観察してから、戦略2or3を判断すべき。**
