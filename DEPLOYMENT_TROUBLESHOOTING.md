# Azure App Service デプロイ トラブルシューティング記録

## 概要
FastAPIアプリケーションをAzure App Serviceにデプロイする際に発生したエラーと解決策の記録

**デプロイ先**: tech0-gen-11-step3-2-py-45.azurewebsites.net
**日付**: 2025年12月6日

---

## エラー1: 504 Gateway Timeout

### 症状
アプリケーションURLにアクセスすると504 Gateway Timeoutエラーが発生
```json
{"message": "FastAPI top page!"}
```
が表示されない

### 原因
- アプリケーションが正しく起動していなかった
- スタートアップコマンドの設定が不適切だった

### 試した解決策

#### ❌ 失敗: gunicornでの起動
`startup.txt`ファイルを作成し、以下のコマンドを設定:
```bash
gunicorn -w 4 -k uvicorn.workers.UvicornWorker app:app --bind=0.0.0.0:8000 --timeout 600
```
→ デプロイエラーが発生したため、アプローチを変更

#### ✅ 成功: uvicornでの自動検出
- `startup.txt`を削除
- Azure App ServiceがFastAPIアプリを自動検出して`uvicorn`で起動
- Azure Portalの「スタートアップコマンド」欄は空白のまま

### 確認した設定
- Azure Portal → 構成 → 全般設定
  - SCM基本認証: オン
  - FTP基本認証: 確認済み
- 環境変数の設定完了

---

## エラー2: ZIP Deploy失敗

### 症状
```
Error: Failed to deploy web package to App Service.
Error: Deployment Failed, Package deployment using ZIP Deploy failed. Refer logs for more details.
```

### 原因
GitHub Actionsのデプロイジョブに必要な権限設定が不足していた

### 解決策
`.github/workflows/main_tech0-gen-11-step3-2-py-45.yml`のデプロイジョブに以下を追加:

```yaml
deploy:
  runs-on: ubuntu-latest
  needs: build
  permissions:
    contents: none
  environment:
    name: 'Production'
    url: ${{ steps.deploy-to-webapp.outputs.webapp-url }}
```

**変更箇所**: L53-57

---

## エラー3: デプロイが40分以上ハングする

### 症状
```
Starting LocalZipHandler
Cleaning up temp folders from previous zip deployments and extracting pushed zip file
/tmp/zipdeploy/daf6a6de-34ed-4314-a933-2f2d9b667a18.zip (55.73 MB) to /tmp/zipdeploy/extracted
```
上記のログで40分以上停止

### 原因
**`.backend/`フォルダ（仮想環境）**がGitリポジトリに含まれていた
- ファイル数: 10,023個
- サイズ: 25MB
- これらは本番環境では不要

### 解決策

#### 1. `.gitignore`に仮想環境を追加
```gitignore
# Python
__pycache__/
*.pyc
*.pyo
*.pyd
.env
venv/
.backend/
```

#### 2. Gitから仮想環境ファイルを削除
```bash
git rm -r --cached .backend
git rm -r --cached venv
```

#### 3. 変更をコミット＆プッシュ
```bash
git add .gitignore
git commit -m "削除"
git push
```

**結果**: 10,023ファイルが削除され、デプロイサイズが大幅に削減
**デプロイ時間**: 40分以上 → 5〜10分程度に短縮

---

## SSL証明書について

### ファイル
`DigiCertGlobalRootG2.crt.pem`

### 用途
Azure Database for MySQL用のSSL証明書

### 今回のエラーとの関連性
**関係なし**

このファイルは:
- MySQL接続時に使用される
- デプロイプロセス自体には影響しない
- アプリ起動後のDB接続で必要

---

## 最終的な成功要因まとめ

1. ✅ 不要な仮想環境ファイル（`.backend/` 10,023個）の削除
2. ✅ GitHub Actionsワークフローへの`permissions`と`environment`追加
3. ✅ Azure Portalでの基本認証設定確認（SCM基本認証: オン）
4. ✅ 環境変数の正しい設定
5. ✅ uvicornによる自動検出（startup.txtは不要）

---

## 今後の注意点

### Gitに含めてはいけないもの
- `venv/` - Python仮想環境
- `.backend/` - 別の仮想環境
- `node_modules/` - Node.jsパッケージ
- `__pycache__/` - Pythonキャッシュファイル
- `.env` - 環境変数ファイル

### 必ず`.gitignore`に追加すること

### デプロイ時の確認事項
1. GitHub Actionsのワークフローが正しく設定されているか
2. Azure Portalで環境変数が設定されているか
3. 基本認証が有効になっているか
4. デプロイログでエラーがないか確認

---

## 参考リンク

- GitHub Actions ワークフロー: `.github/workflows/main_tech0-gen-11-step3-2-py-45.yml`
- GitIgnore設定: `.gitignore`
- アプリケーションURL: https://tech0-gen-11-step3-2-py-45.azurewebsites.net/
- GitHubリポジトリ: https://github.com/ryotsunekawa/Tech0_Step3-2_backend

---

**デプロイ成功日**: 2025年12月6日
