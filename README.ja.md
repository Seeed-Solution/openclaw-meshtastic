# MeshClaw: OpenClaw Meshtastic チャンネルプラグイン

<p align="center">
  <a href="https://www.npmjs.com/package/@seeed-studio/meshtastic">
    <img alt="npm version" src="https://img.shields.io/npm/v/@seeed-studio/meshtastic.svg" />
  </a>
  <a href="https://www.npmjs.com/package/@seeed-studio/meshtastic">
    <img alt="license" src="https://img.shields.io/npm/l/@seeed-studio/meshtastic.svg" />
  </a>
</p>

<!-- LANG_SWITCHER_START -->
<p align="center">
  <a href="README.md">English</a> | <a href="README.zh-CN.md">中文</a> | <b>日本語</b>
</p>
<!-- LANG_SWITCHER_END -->

<p align="center">
  <img src="media/GoMeshClaw.png" width="700" alt="Meshtastic LoRa ハードウェア" />
</p>

MeshClaw は、AI ゲートウェイと Meshtastic LoRa メッシュネットワークを Serial（USB）、HTTP（WiFi）、MQTT のいずれかで接続する OpenClaw チャンネルプラグインです。

> [!IMPORTANT]
> このリポジトリは **OpenClaw チャンネルプラグイン**です。スタンドアロンアプリではありません。
> 利用には動作中の [OpenClaw](https://github.com/openclaw/openclaw) ゲートウェイ（Node.js 22+）が必要です。

[Meshtastic ドキュメント][docs] · [バグを報告][issues] · [機能リクエスト][issues]

## 目次

- [前提条件](#前提条件)
- [クイックスタート](#クイックスタート)
- [動作原理](#動作原理)
- [主な機能](#主な機能)
- [トランスポートモード](#トランスポートモード)
- [アクセス制御](#アクセス制御)
- [設定](#設定)
- [デモ](#デモ)
- [推奨ハードウェア](#推奨ハードウェア)
- [トラブルシューティング](#トラブルシューティング)
- [制限事項](#制限事項)
- [開発](#開発)
- [コントリビューション](#コントリビューション)
- [ライセンス](#ライセンス)

## 前提条件

- OpenClaw ゲートウェイがインストール済みかつ動作中であること
- Node.js 22+
- Meshtastic 接続方法のいずれか:
  - Serial デバイス（USB 経由）、または
  - LAN 上で HTTP が有効な Meshtastic デバイス、または
  - MQTT broker へのアクセス（ローカルハードウェアは不要）

## クイックスタート

```bash
# 1) Install plugin from npm
openclaw plugins install @seeed-studio/meshtastic

# 2) Run guided setup
openclaw onboard

# 3) Verify channel status
openclaw channels status --probe
```

<p align="center">
  <img src="media/setup-screenshot.png" width="700" alt="OpenClaw セットアップウィザード" />
</p>

## 動作原理

```mermaid
flowchart LR
    subgraph mesh ["LoRa Mesh Network"]
        N["Meshtastic Nodes"]
    end
    subgraph gw ["OpenClaw Gateway"]
        P["MeshClaw Plugin"]
        AI["AI Agent"]
    end
    N -- "Serial (USB)" --> P
    N -- "HTTP (WiFi)" --> P
    N -. "MQTT (Broker)" .-> P
    P <--> AI
```

受信メッセージは AI Agent に到達する前に、DM / グループポリシーのチェックを通過します。
送信返信はプレーンテキストに変換され、無線送信向けにチャンク分割されます。

## 主な機能

- **3 種類のトランスポート**: Serial、HTTP、MQTT
- **DM ポリシー制御**: `pairing`、`open`、または `allowlist`
- **グループポリシー制御**: `disabled`、`open`、または `allowlist`
- **@mention ゲーティング**: メンションされた場合のみグループ内で返信（オプション）
- **マルチアカウント対応**: 複数の独立した Meshtastic 接続を実行可能
- **耐障害性のあるトランスポート処理**: 不安定なリンクでの再接続動作

## トランスポートモード

| モード | 用途 | 必須フィールド |
|---|---|---|
| `serial` | ローカルの USB 接続ノード | `transport`、`serialPort` |
| `http` | ローカルネットワーク到達可能なノード | `transport`、`httpAddress` |
| `mqtt` | ローカルハードウェアなし、共有 broker | `transport`、`mqtt.*`、`nodeName` |

備考:
- `serial` がデフォルトのトランスポートです。
- `mqtt` のデフォルト: broker `mqtt.meshtastic.org`、トピック `msh/US/2/json/#`。
- リージョン設定は Serial / HTTP に適用されます。MQTT はトピックからリージョンを導出します。

## アクセス制御

### DM ポリシー（`dmPolicy`）

| 値 | 動作 |
|---|---|
| `pairing`（デフォルト）| 新規ユーザーは DM チャット開始前に承認が必要です |
| `open` | 任意のノードから DM を受け付けます |
| `allowlist` | `allowFrom` に含まれる ID のみ DM を許可します |

### グループポリシー（`groupPolicy`）

| 値 | 動作 |
|---|---|
| `disabled`（デフォルト）| グループチャンネルを無視します |
| `open` | すべてのグループチャンネルで応答します |
| `allowlist` | 設定されたチャンネルのみで応答します |

チャンネルごとにメンションを必須にすることも可能（`requireMention`）で、明示的にメンションされた場合のみボットが返信します。

## 設定

ガイド付きセットアップには `openclaw onboard` を使用するか、`openclaw config edit` で手動で設定を編集してください。

### Serial（USB）

```yaml
channels:
  meshtastic:
    transport: serial
    serialPort: /dev/ttyUSB0
    nodeName: OpenClaw
```

### HTTP（WiFi）

```yaml
channels:
  meshtastic:
    transport: http
    httpAddress: meshtastic.local
    nodeName: OpenClaw
```

### MQTT（Broker）

```yaml
channels:
  meshtastic:
    transport: mqtt
    nodeName: OpenClaw
    mqtt:
      broker: mqtt.meshtastic.org
      username: meshdev
      password: large4cats
      topic: "msh/US/2/json/#"
```

### マルチアカウント

```yaml
channels:
  meshtastic:
    accounts:
      home:
        transport: serial
        serialPort: /dev/ttyUSB0
      remote:
        transport: mqtt
        mqtt:
          broker: mqtt.meshtastic.org
          topic: "msh/US/2/json/#"
```

<details>
<summary><b>設定リファレンス</b></summary>

| キー | 型 | デフォルト | 備考 |
|---|---|---|---|
| `transport` | `serial \| http \| mqtt` | `serial` | 基本トランスポート |
| `serialPort` | `string` | - | `serial` で必須 |
| `httpAddress` | `string` | `meshtastic.local` | `http` で必須 |
| `httpTls` | `boolean` | `false` | HTTP TLS |
| `mqtt.broker` | `string` | `mqtt.meshtastic.org` | MQTT broker ホスト |
| `mqtt.port` | `number` | `1883` | MQTT ポート |
| `mqtt.username` | `string` | `meshdev` | MQTT ユーザー名 |
| `mqtt.password` | `string` | `large4cats` | MQTT パスワード |
| `mqtt.topic` | `string` | `msh/US/2/json/#` | 購読トピック |
| `mqtt.publishTopic` | `string` | derived | オプションで上書き |
| `mqtt.tls` | `boolean` | `false` | MQTT TLS |
| `region` | enum | `UNSET` | Serial / HTTP のみ |
| `nodeName` | `string` | auto-detect | MQTT で必須 |
| `dmPolicy` | `open \| pairing \| allowlist` | `pairing` | DM アクセスポリシー |
| `allowFrom` | `string[]` | - | DM 許可リスト、例: `!aabbccdd` |
| `groupPolicy` | `open \| allowlist \| disabled` | `disabled` | グループチャンネルポリシー |
| `channels` | `Record<string, object>` | - | チャンネルごとの上書き設定 |
| `textChunkLimit` | `number` | `200` | 許可範囲: `50-500` |

</details>

<details>
<summary><b>環境変数による上書き</b></summary>

これらの変数はデフォルトアカウントのフィールドを上書きします。

| 変数 | 設定キー |
|---|---|
| `MESHTASTIC_TRANSPORT` | `transport` |
| `MESHTASTIC_SERIAL_PORT` | `serialPort` |
| `MESHTASTIC_HTTP_ADDRESS` | `httpAddress` |
| `MESHTASTIC_MQTT_BROKER` | `mqtt.broker` |
| `MESHTASTIC_MQTT_TOPIC` | `mqtt.topic` |

</details>

## デモ

<div align="center">

https://github.com/user-attachments/assets/837062d9-a5bb-4e0a-b7cf-298e4bdf2f7c

</div>

フォールバック: [media/demo.mp4](media/demo.mp4)

## 推奨ハードウェア

<p align="center">
  <img src="media/XIAOclaw.png" width="760" alt="Seeed XIAO モジュールを搭載した Meshtastic デバイス" />
</p>

| デバイス | 用途 | リンク |
|---|---|---|
| XIAO ESP32S3 + Wio-SX1262 キット | 入門用開発 | [購入][hw-xiao] |
| Wio Tracker L1 Pro | ポータブルフィールドゲートウェイ | [購入][hw-wio] |
| SenseCAP Card Tracker T1000-E | コンパクトトラッカー | [購入][hw-sensecap] |

Meshtastic 対応デバイスであればどれでも動作します。MQTT モードはローカルハードウェアなしで実行できます。

## トラブルシューティング

| 症状 | 確認事項 |
|---|---|
| Serial が接続できない | `serialPort` は正しいですか？ホストにデバイス権限はありますか？ |
| HTTP が接続できない | `httpAddress` に到達可能ですか？`httpTls` は正しく設定されていますか？ |
| MQTT でメッセージを受信しない | トピックのリージョンは正しいですか？broker 認証情報は有効ですか？ |
| DM 返信がない | `dmPolicy` と `allowFrom` を確認してください |
| グループ返信がない | `groupPolicy`、許可リスト、メンション要件を確認してください |

issue を作成する際は、トランスポートモード・編集済みの設定・`openclaw channels status --probe` の出力を添えてください。

## 制限事項

- LoRa メッセージは帯域幅が制限されています。返信はチャンク分割されます（`textChunkLimit`、デフォルト `200`）。
- リッチなマークダウンは無線デバイスへの送信前に除去されます。
- メッシュの品質・範囲・レイテンシは無線環境とネットワーク状況に依存します。

## 開発

```bash
git clone https://github.com/Seeed-Solution/openclaw-meshtastic.git
cd openclaw-meshtastic
npm install
openclaw plugins install -l ./openclaw-meshtastic
openclaw channels status --probe
```

ビルドステップは不要です。OpenClaw は `index.ts` から TypeScript ソースを直接読み込みます。

## コントリビューション

- [GitHub Issues][issues] から issue や機能リクエストを作成してください
- Pull Request を歓迎します
- 既存の TypeScript 規約に合わせて変更してください

## ライセンス

MIT

<!-- Reference-style links -->
[docs]: https://meshtastic.org/docs/
[issues]: https://github.com/Seeed-Solution/openclaw-meshtastic/issues
[hw-xiao]: https://www.seeedstudio.com/Wio-SX1262-with-XIAO-ESP32S3-p-5982.html
[hw-wio]: https://www.seeedstudio.com/Wio-Tracker-L1-Pro-p-6454.html
[hw-sensecap]: https://www.seeedstudio.com/SenseCAP-Card-Tracker-T1000-E-for-Meshtastic-p-5913.html