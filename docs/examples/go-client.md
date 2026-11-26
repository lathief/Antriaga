# Contoh integrasi Go dengan `net/http`

Contoh ini membuat job `email.send` melalui HTTP lalu membaca status-nya. Menggunakan **Go standard library**, bukan SDK Antriaga resmi. Contoh **belum diuji terhadap runtime Antriaga**; perlu menyesuaikan DTO jika response final berubah.

## Environment

Sediakan `ANTRIAGA_URL` (HTTPS di luar local development) dan `ANTRIAGA_API_KEY` (dari secret manager). Jangan menaruh key literal di source code.

## Contoh Go

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strings"
	"time"
)

type CreateResponse struct {
	Data struct {
		ID     string `json:"id"`
		Status string `json:"status"`
	} `json:"data"`
}

func main() {
	baseURL := strings.TrimRight(os.Getenv("ANTRIAGA_URL"), "/")
	apiKey := os.Getenv("ANTRIAGA_API_KEY")
	if baseURL == "" || apiKey == "" {
		panic("ANTRIAGA_URL and ANTRIAGA_API_KEY must be set")
	}

	client := &http.Client{Timeout: 10 * time.Second}
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	payload := map[string]any{
		"type":            "email.send",
		"payload_version": 1,
		"payload": map[string]string{
			"to":       "sample@example.com",
			"subject":  "Welcome",
			"template": "welcome",
		},
		"max_attempts": 5,
	}
	body, err := json.Marshal(payload)
	if err != nil {
		panic(err)
	}

	req, err := http.NewRequestWithContext(ctx, http.MethodPost,
		baseURL+"/api/v1/jobs", bytes.NewReader(body))
	if err != nil {
		panic(err)
	}
	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("X-API-Key", apiKey)
	req.Header.Set("Idempotency-Key", "go-demo-welcome-001")

	resp, err := client.Do(req)
	if err != nil {
		panic(err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusAccepted {
		// Hindari mencetak response mentah jika mungkin mengandung informasi sensitif.
		_, _ = io.Copy(io.Discard, io.LimitReader(resp.Body, 4096))
		panic(fmt.Errorf("Antriaga returned HTTP %d", resp.StatusCode))
	}
	var out CreateResponse
	if err := json.NewDecoder(io.LimitReader(resp.Body, 1<<20)).Decode(&out); err != nil {
		panic(err)
	}
	if out.Data.ID == "" {
		panic(errors.New("missing job ID"))
	}
	fmt.Printf("job accepted: id=%s status=%s\n", out.Data.ID, out.Data.Status)

	// Gunakan GET /api/v1/jobs/:id dari service terpisah untuk polling.
	// HTTP 202 tidak berarti job sudah selesai.
}
```

Untuk retry network error ketika POST, ulangi **body yang identik dan `Idempotency-Key` yang sama**. Jangan otomatis retry pada `401`, `403`, `409`, `422` atau rate-limit tanpa backoff. Untuk production, gunakan structured client telemetry, bounded retry, dan secret redaction.

Baca [Quickstart](../getting-started/quickstart.md), [Errors](../reference/errors.md), dan [API](../reference/api.md).

[Kembali ke indeks dokumentasi](../README.md)
