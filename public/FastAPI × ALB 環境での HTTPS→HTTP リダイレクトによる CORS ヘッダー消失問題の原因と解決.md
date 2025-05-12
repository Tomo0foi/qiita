---
title: FastAPI × ALB 環境での HTTPS→HTTP リダイレクトによる CORS ヘッダー消失問題の原因と解決
tags:
  - FastAPI
  - AWS
  - Application-Load-Balancer
  - Uvicorn
  - CORS
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに
AWS の Application Load Balancer（ALB）で HTTPS 終端を行い、バックエンド（FastAPI／Uvicorn）へは HTTP でリクエストを流す構成において、CORS エラーが発生する事例がありました。  
原因は **HTTPS→HTTP へリダイレクトされる際に必要な `Access-Control-*` ヘッダーが欠落すること** にありました。この記事では、その原因と解決手順をまとめます。

## 問題の再現例

1. ブラウザがプリフライト（OPTIONS）リクエストを発行  
   ```bash
   OPTIONS https://request.example.com
   Origin: https://client.example.com
   ```

2. ALB（HTTPS→HTTP 終端） → Uvicorn（HTTP:80）へ転送

3. FastAPI がリダイレクトを返却  
   ```http
   HTTP/1.1 307 Temporary Redirect
   Location: http://request.example.com/
   ```

4. ブラウザが HTTP 側へ再プリフライト  

5. ALB の HTTP→HTTPS リダイレクトルール（301）で HTTPS に戻る

6. **一度も CORS ヘッダーが付与されず** ブラウザが以下を出力して失敗
   ```
   No ‘Access-Control-Allow-Origin’ header is present on the requested resource.
   ```  

## 原因の詳細

- **ALB が HTTPS 終端** → バックエンドへは HTTP で転送  
  この際、起動時の設定を行なっておらず、`X-Forwarded-Proto: https` が付与されていない
- **Uvicorn** はデフォルトで「127.0.0.1 からのプロキシのみ信頼」  
  → ALB からの `X-Forwarded-Proto` を無視 → `request.url.scheme` が `http` のまま  
- ブラウザは一度 HTTP エンドポイントへ飛び、CORS ミドルウェアを経由しないまま再リダイレクト  
- 結果的に **CORS ヘッダーが一切付与されず** リクエストがブロック

## 解決方法

### 1. Uvicorn 起動時にプロキシを全許可する

Dockerfile や起動スクリプトで、以下オプションを追加します。

```bash
uvicorn app.main:app --reload --proxy-headers --forwarded-allow-ips="*"
```

- `--proxy-headers`：X-Forwarded ヘッダーを処理  
- `--forwarded-allow-ips="*"`：任意のプロキシ IP を信頼

### 2. コード上で ProxyHeadersMiddleware を追加

CLI オプションが使えない場合は、`main.py` に以下を挿入します。

```python
from starlette.middleware.proxy_headers import ProxyHeadersMiddleware

app = FastAPI()
app.add_middleware(ProxyHeadersMiddleware, trusted_hosts="*")
```

## まとめ

1. **真の原因**：HTTPS→HTTP リダイレクト時に CORS ヘッダーが付かずブロック  
2. **対策**：Uvicorn に `--proxy-headers` と `--forwarded-allow-ips="*"` を追加する

以上の手順で、ALB 経由の CORS エラーは解消できます。ぜひお試しください！