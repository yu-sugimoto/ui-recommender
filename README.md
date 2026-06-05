UI Recommender
======================

ユーザーがGitHubリポジトリのURLとデザイン指示を入力すると, AIが同時に最大5件分のUIデザイン改善を実装し, 動く成果物（Before/Afterスクリーンショット）を並べて見比べさせ, 選んだ案をそのままPull Requestにしてくれるツールです. UI改善案の良し悪しは「plan のテキスト」だけでは判断しづらく, 最終的には実装してみないと分からないという点が面倒だったため, 先に全案を実装してしまい, 具体物で比較できるようにしようと思い開発しました. なお, このプロジェクトは 2026/2/15-3/29 に行われた「Webアプリハッカソン」の成果物になります.

## 概要

<img width="1271" height="595" alt="image" src="https://github.com/user-attachments/assets/5b1f363d-db81-4f14-bbd3-76e719393cbd" />

<div>&nbsp;</div>

このツールでは, GitHubリポジトリの解析とデザインコンテキスト抽出（ui-ux-pro-max CLI）, Before/Afterスクリーンショット取得（Playwright MCP）, Claude Agent SDKによる最大5件のデザイン案生成と並列実装, S3/MinIOによる成果物保存, GitHubへのPR自動作成を実装しました. ユーザーがリポジトリURL・ベースブランチ・デザイン指示を入力すると, AIが最大5件分のデザイン案をすべて実装し, それぞれのafterスクリーンショットを成果物として返します. ユーザーは「コード変更前の plan」ではなく「実装済みの動く成果物」を見比べて1件を選び, 選ばれた案だけがPRとしてGitHubに作成されます.

### 提案生成・実装パイプライン（LangGraph + Claude Agent SDK ワーカー）の実装

各段階は LangGraph の StateGraph（analyze / implement / createpr の3グラフ）として実装され, ノードは「K8s Job 作成 → Job 完了待機 → S3 から結果取得」の3ステップで構成されます. 実際のAI処理（分析・実装・PR作成）は Kubernetes Job として起動されるワーカーPod内の Claude Agent SDK が担います.

1. 解析と plan 生成（analyze グラフ）  
リポジトリを clone し, `uiux-pro-max init` でデザインコンテキストを抽出, 開発サーバを起動して Playwright MCP で beforeスクリーンショット を取得. その後 Claude Agent SDK が コードは変更せず, 5件分の plan（タイトル・コンセプト・実装計画・対象ファイル）を JSON で生成し S3 に保存
2. 最大5件の並列実装（implement グラフ）  
analyze 完了直後にバックエンドが 最大5件すべてを並列に自動実装ジョブとして投入. 各ジョブで Claude Agent SDK が plan に従いコード変更 → 開発サーバー起動 → Playwright MCP で afterスクリーンショット 取得 → 累積パッチ（diff）を S3 に保存
3. 成果物の競い合いとユーザーによる選択  
フロントエンド（React）で5件分の after スクリーンショット・diff・ログを並べて比較. ユーザーが「実際に動く成果物」を見比べて採用案を1件選ぶ
4. PR 作成（createpr グラフ）  
選ばれた案の既存パッチを feature ブランチに適用して GitHub へ push, PR を作成し PR URL を返却
5. 反復（iterate）  
選択された案を出発点として次イテレーションを起動し, analyze → implement×5 のサイクルを繰り返す
6. 進捗の可視化  
各 K8s Job のログを SSE（`sse-starlette`）でフロントエンドにストリーミング表示

## デモ

https://github.com/user-attachments/assets/70bfb2ec-21a3-489a-95f6-c40c59a0def5

## 使用技術
#### フロントサイド
- TypeScript
- React 19
- Vite
- React Router v7

#### サーバーサイド
- Python 3.13
- FastAPI
- LangGraph（オーケストレーション層の StateGraph）
- Anthropic Claude Agent SDK（ワーカー内の AI 処理）
- SQLAlchemy + Alembic
- PostgreSQL 17
- Kubernetes（ワーカー Job 実行）
- Docker
- S3 / MinIO（boto3）
- Playwright MCP
- ui-ux-pro-max CLI

## プロジェクトの構成

インストールについては frontend/backend ディレクトリ内の README 参照

```
.
├─frontend : クライアントサイド（React + Vite）
├─backend  : APIサーバー / オーケストレーター（FastAPI + Postgres + LangGraph）
├─docker   : ワーカースクリプトと Dockerfile（analyze / implement / createpr）
└─k8s      : Kubernetes マニフェスト（Job テンプレート, Secret）
```
