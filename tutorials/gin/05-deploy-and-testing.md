# Go 05 — Containerizing, Deploying to AWS (ECS and Lambda), Bedrock from Go

## Tiny container with a multi-stage build

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.23 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download                       # cached layer
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -trimpath -ldflags="-s -w" -o /out/api ./cmd/api

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/api /api
COPY migrations /migrations
EXPOSE 8080
USER nonroot:nonroot
ENTRYPOINT ["/api"]
```
Result: ~15–25 MB image, no shell/package manager (small attack surface), runs as non-root. `CGO_ENABLED=0` yields a static binary. Cross-compile for Graviton: `GOARCH=arm64`.

## ECS Fargate
Same steps as [../aws/04](../aws/04-ecs-fargate-and-ecr.md): push to ECR, task definition (port 8080, health check `/health`), service behind ALB. Go advantages: tiny CPU/memory footprint (e.g. 0.25 vCPU/512 MB is often plenty), instant start, fast scale-out, clean SIGTERM drain via `srv.Shutdown`. Set the ECS `stopTimeout` ≥ your shutdown timeout.

## Lambda

### Option A: native handler (smallest/fastest)
See the handler in [../aws/03](../aws/03-lambda-and-api-gateway.md). Build:

```bash
GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build -tags lambda.norpc -o bootstrap ./cmd/lambda
zip function.zip bootstrap
```
Runtime `provided.al2023`, handler `bootstrap`.

### Option B: run Gin unchanged with an adapter

```go
import ginadapter "github.com/awslabs/aws-lambda-go-api-proxy/gin"

var adapter *ginadapter.GinLambdaV2

func init() { adapter = ginadapter.NewV2(newRouter()) }           // cold-start work happens once

func handler(ctx context.Context, req events.APIGatewayV2HTTPRequest) (events.APIGatewayV2HTTPResponse, error) {
	return adapter.ProxyWithContext(ctx, req)
}
func main() { lambda.Start(handler) }
```
Cold starts for Go are typically tens of ms, one of Go's biggest Lambda advantages over Python/Node.

## Calling Bedrock from Go (AWS SDK v2)

```go
go get github.com/aws/aws-sdk-go-v2/config github.com/aws/aws-sdk-go-v2/service/bedrockruntime
```

```go
import (
	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/service/bedrockruntime"
	"github.com/aws/aws-sdk-go-v2/service/bedrockruntime/types"
	"github.com/aws/aws-sdk-go-v2/aws"
)

func Summarize(ctx context.Context, client *bedrockruntime.Client, modelID, prompt string) (string, error) {
	out, err := client.Converse(ctx, &bedrockruntime.ConverseInput{
		ModelId: aws.String(modelID),
		Messages: []types.Message{{
			Role:    types.ConversationRoleUser,
			Content: []types.ContentBlock{&types.ContentBlockMemberText{Value: prompt}},
		}},
		InferenceConfig: &types.InferenceConfiguration{MaxTokens: aws.Int32(500), Temperature: aws.Float32(0.2)},
	})
	if err != nil { return "", err }
	msg, ok := out.Output.(*types.ConverseOutputMemberMessage)
	if !ok || len(msg.Value.Content) == 0 { return "", errors.New("empty response") }
	if t, ok := msg.Value.Content[0].(*types.ContentBlockMemberText); ok { return t.Value, nil }
	return "", errors.New("unexpected content type")
}

// create once:
// cfg, _ := config.LoadDefaultConfig(ctx); client := bedrockruntime.NewFromConfig(cfg)
```
(Type names follow the SDK's union-type style; verify against the installed SDK version.) Wrap the call in a timeout context and bounded retries (SDK has built-in retry config).

## Test, lint, CI

```bash
go vet ./...
staticcheck ./...                      # or golangci-lint run
go test -race -coverprofile=coverage.out ./...
govulncheck ./...                      # known vulnerable deps
```

```yaml
# .github/workflows/go.yml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version-file: api-gin/go.mod, cache-dependency-path: api-gin/go.sum }
      - run: go vet ./... && go test -race -coverprofile=coverage.out ./...
        working-directory: api-gin
      - uses: golangci/golangci-lint-action@v6
        with: { working-directory: api-gin }
```
SonarQube reads `coverage.out` via `sonar.go.coverage.reportPaths`. Deploy steps: identical to [../aws/08](../aws/08-cicd-observability-and-cost.md).

## Profiling and performance
`import _ "net/http/pprof"` on an internal port, then `go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30` (CPU), `/heap`, `/goroutine`. Benchmarks: `go test -bench . -benchmem`. Load test: `hey`, `k6`, `vegeta`. Use `GOMEMLIMIT` and `GOMAXPROCS` awareness in containers (Go 1.25+ is container-aware for CPU limits; otherwise use `automaxprocs`).

## Exercise
Build the multi-stage image, compare image size and cold-start/p95 against the FastAPI container ([../fastapi/05](../fastapi/05-deploy-to-aws.md)) under the same `k6` script; write the findings as an ADR.

## Interview Q&A
- **Why is the Go image so small?** Static binary on distroless/scratch; no runtime needed.
- **Why Go on Lambda?** Fast cold starts, low memory, single binary.
- **How do you find a memory leak?** `pprof` heap diffs, goroutine profile, check unbounded maps/caches/goroutines.
- **How do you do graceful shutdown on ECS?** Trap SIGTERM → `Shutdown(ctx)`; ALB deregistration delay.
- **Static analysis in Go?** `go vet`, `staticcheck`/`golangci-lint`, `govulncheck`, `-race`.
