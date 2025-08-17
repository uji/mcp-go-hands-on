# Go で MCP サーバーを作ろう

Go で MCP サーバーを作るワークショップの資料です

## 想定する参加者

- Goをほとんど or 全く書いたことがないがGoに興味があるかた
- 他のプログラミング言語での簡易的なプログラミング経験（if, for, 関数がだいたい分かる程度）がある方

# 環境構築

1. Go をインストール(最新バージョンでok) https://go.dev/doc/install
2. `mkdir` などで作業ディレクトリを作成
3. `go mod init mcp-go-hands-on`
4. `main.go` を作成し、https://go.dev/play/ のコードをコピーペースト
5. `go run .`

## mcp-go のインストール

作業ディレクトリで `go get github.com/mark3labs/mcp-go`

### mcp-go について

https://github.com/mark3labs/mcp-go

- 現状最も利用されているGoのMCP SDK
- GitHubの公式MCPでも利用されている https://github.com/github/github-mcp-server
- このライブラリを参考にGoチームが公式でSDKを開発中 https://github.com/modelcontextprotocol/go-sdk

# MCP (Model Context Protocol) 概要

- MCPは、Claudeの開発元であるAnthropicによりLLMとローカル環境を接続するための標準プロトコル
- この規格で実装されたMCPサーバーを使うことで、LLMにローカルファイルの読み取りやAPIへのアクセスなどの機能を拡張できる
- JSON-RPC の仕様が規格化されているだけなので、様々なプログラミング言語で実装可能
- Claude DesktopやCursor、Clineなどでは既にMCPサーバーの利用がサポートされている

[ドキュメント](https://modelcontextprotocol.io/docs/getting-started/intro)

# MCP に触れてみよう

Gemini CLIを使って、context7 を利用する例

- https://github.com/google-gemini/gemini-cli
- https://github.com/upstash/context7

`~/.gemini/settings.json` に以下を追記してください

```json:~/.gemini/settings.json
{
  "mcpServers": {
    "context7": {
      "httpUrl": "https://mcp.context7.com/mcp"
    }
  }
}
```

`mark3labs/mcp-go の基本的な使い方を教えて` など聞いてみる

# 作ってみる

## Hello, World!

```main.go
package main

import (
	"context"
	"fmt"

	"github.com/mark3labs/mcp-go/mcp"
	"github.com/mark3labs/mcp-go/server"
)

func main() {
	s := server.NewMCPServer(
		"uji mcp", // お好きな名前を
		"1.0.0",
	)

	tool := mcp.NewTool("HelloTool",
		mcp.WithDescription(("HelloTool is a tool that says hello")),
		mcp.WithString("your name", mcp.Description("Name of the person to greet")),
	)

	helloHandler := func(ctx context.Context, request mcp.CallToolRequest) (*mcp.CallToolResult, error) {
		name, err := request.RequireString("your name")
		if err != nil {
			return mcp.NewToolResultError(err.Error()), nil
		}

		return mcp.NewToolResultText(fmt.Sprintf("Hello, %s!", name)), nil
	}

	s.AddTool(tool, helloHandler)

	err := server.ServeStdio(s)
	if err != nil {
		panic(err)
	}
}
```

```json:~/.gemini/settings.json
{
  "mcpServers": {
    "context7": {
      "httpUrl": "https://mcp.context7.com/mcp"
    },
    "uji mcp": {
      "command": "go",
      "args": ["run", "/Users/uji/mcp-go-hands-on"]
    }
  }
}
```

`~/.gemini/GEMINI.md` に自分のニックネームを書いて gemini を起動し、`uji mcp の HelloTool を使ってください` と聞いてみてください

## 四則演算

2つの数値を受け取って四則演算を行う機能を実装してみましょう

### ヒント

- `mcp.NewTool` のオプションは可変長引数になっており、複数渡すことができる
- `mcp.NewTool` のオプションで 演算子(operator), 値2つを受け取るように定義する
- handler では `request.RequireInt("x")` とすると入力を整数型intとして受け取れる
- 計算ロジックは以下のように書ける
  ```
  var result int
  switch op {
  case "add":
      result = x + y
  case "subtract":
      result = x - y
  case "multiply":
      result = x * y
  case "divide":
      if y == 0 {
          return mcp.NewToolResultError("cannot divide by zero"), nil
      }
      result = x / y
  }
  ```

## テストを書いてみる

`github.com/mark3labs/mcp-go/mcptest` を使うことでテストコード上でサーバーの起動とリクエストが行えます

- tool, handler を他の関数からも呼び出せるように main関数の外の変数 `CalculateTool` ,関数 `CalculateHander` として実装

```go:main_test.go
package main

import (
	"fmt"
	"strings"
	"testing"

	handler を公開"github.com/mark3labs/mcp-go/mcp"
	"github.com/mark3labs/mcp-go/mcptest"
	"github.com/mark3labs/mcp-go/server"
)

func TestMain(t *testing.T) {
	s, err := mcptest.NewServer(t, server.ServerTool{
		Tool:    CalculateTool,
		Handler: CalculateHander,
	})
	if err != nil {
		t.Fatalf("Failed to create server: %v", err)
	}
	defer s.Close()

	req := mcp.CallToolRequest{
		Params: mcp.CallToolParams{
			Name: "CaluculateTool",
			Arguments: map[string]any{
				"operator": "add",
				"x":        5,
				"y":        10,
			},
		},
	}
	client := s.Client()
	result, err := client.CallTool(t.Context(), req)
	if err != nil {
		t.Fatalf("Failed to call tool: %v", err)
	}
	if result.IsError {
		t.Fatalf("Tool call returned error: %s", result.Content)
	}
	resultStr, err := resultToString(result)
	if err != nil {
		t.Fatalf("Failed to convert result to string: %v", err)
	}

	if resultStr != "The result is: 13" {
		t.Fatalf("Unexpected result: %s", result.Content)
	}
}

func resultToString(result *mcp.CallToolResult) (string, error) {
	var b strings.Builder

	for _, content := range result.Content {
		text, ok := content.(mcp.TextContent)
		if !ok {
			return "", fmt.Errorf("unsupported content type: %T", content)
		}
		b.WriteString(text.Text)
	}

	if result.IsError {
		return "", fmt.Errorf("%s", b.String())
	}

	return b.String(), nil
}
```