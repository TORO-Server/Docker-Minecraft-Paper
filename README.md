# Docker-Minecraft-Paper

PaperMC (Minecraft サーバー) の最小限の Docker イメージを自動ビルドおよび配信するプロジェクト。

## 特徴
- **自動ビルド & 公開**: GitHub Actions を利用し、毎日 PaperMC の新規ビルドを自動的に検出して GHCR (GitHub Container Registry) にパブリッシュする。
- **軽量・安全なイメージ**: ベースイメージは Google の `distroless/java-base` (シェルすら含まれない)。Java の実行環境は `jlink` で作り (モジュールは Temurin の JRE と同じ)、既定は管理者権限を持たない一般ユーザー (非 root、65532) で動く。
- **Java バージョンの自動追従**: PaperMC の jar ファイルを解析し、必要な Java バージョン (Java 21 等) を自動的に検出して Docker ベースイメージを切り替える。
- **マルチプラットフォーム対応**: `linux/amd64` および `linux/arm64` アーキテクチャに対応。

## 提供されるタグ
パブリッシュされるイメージには、以下のルールに従って複数のタグが割り当てられる：
- **ビルド特定タグ**: `${version}-${build}` (例: `1.21.4-56`)
- **マイナーバージョン最新タグ**: `${version}-latest` (例: `1.21.4-latest`)
- **全体最新タグ (PaperMC の最新ビルド時のみ)**:
  - `latest`
  - `latest-latest`
  - `latest-${build}`

> [!NOTE]
> 自動ビルドの対象は、Minecraft `1.21.4` 以上のバージョンと、`1.20.4` (最後のビルドだけ) です。対象の条件の正本は `updatePaper.py` です。

## 使い方

本イメージには Java の実行環境と `paper.jar` のみが含まれている。Minecraft サーバーを実行するには、ワールドデータや設定ファイルを保存するディレクトリを作業ディレクトリ `/app` にマウント (ホスト側のディレクトリをコンテナの中から見えるようにすること) し、`eula.txt` を配置する必要がある。

### 1. ディレクトリと eula.txt の準備
ホスト側 (サーバーを動かす PC 側) にデータを保持するディレクトリ（例: `./data`）を作成し、Minecraft の EULA (使用許諾契約) に同意する設定ファイルを作成する。

```bash
mkdir data
echo "eula=true" > data/eula.txt
```

### 2. コンテナの起動
作成したディレクトリをコンテナの `/app` にマウントして起動する。

```bash
docker run -d \
  --user "$(id -u):$(id -g)" \
  -p 25565:25565 \
  -v $(pwd)/data:/app \
  --name minecraft-server \
  ghcr.io/<owner>/docker-minecraft-paper:latest
```

## 構成スクリプト

### [updatePaper.py](/updatePaper.py)
GitHub Actions 自動処理でビルドプロセスを制御するための多機能 Python スクリプト。

#### コマンドラインの使用方法
- **ビルド対象の一覧 (マトリックス) の取得**
  まだ `ghcr.io` にパブリッシュされていない新しいビルド情報を PaperMC API から取得し、GitHub Actions のマトリックス形式 (JSON。データを文字で表す形式の 1 つ) で出力する。
  ```bash
  python updatePaper.py get-matrix
  ```
- **jar ファイルのダウンロードと SHA256 (ファイルの改ざんを確かめる値) 検証**
  指定した URL から jar ファイルをダウンロードし、チェックサムを検証する。
  ```bash
  python updatePaper.py download <保存パス> <ダウンロードURL> <期待されるSHA256>
  ```
- **Java バージョンの検出**
  jar ファイルに含まれるクラスファイルのバージョンヘッダから、必要な Java メジャーバージョンを自動的に判定して出力する。
  ```bash
  python updatePaper.py get-jar-version <jarファイルのパス>
  ```
- **Java バージョンの整合性検証**
  Dockerfile が提供する Java バージョンが、jar ファイルが要求する Java バージョン以上であることを確認する。
  ```bash
  python updatePaper.py verify-java <jarファイルのパス> <Dockerfileのパス>
  ```
