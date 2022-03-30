# Temporal Scylla Persistence Bug Repro Repository

## The problem
Child workflow can't be executed twice when scylla is used 
as temporal persistence backend. Also it seems like Temporal appears in a deadlock, 
because temporal process consumes nearly 200% cpu and web UI is not responsive (can't click through workflows).

## Steps to reproduce
* `docker compose up`. Wait for everything to start
* Start `worker`
* Start `starter`
* Navigate to `localhost:8088` and check that both parent and child workflow has completed
* Start `starter` again and update Web UI.

At this step it is expected that we would see 4 completed workflows 
(2 from previous step and 2 from the recent starter run). 
But actually what we see is 3 total workflows and one of which is a stuck parent workflow.

## Further confirmation steps

* Check top/htop or Activity Monitor on mac and confirm high CPU usage
* Check temporal logs and see this error:
```json
{
  "level":"error",
  "ts":"2022-03-30T17:20:42.225Z",
  "msg":"Persistent store operation Failure",
  "shard-id":1,
  "address":"127.0.0.1:7234",
  "wf-namespace-id":"1e16cd9f-097d-4e81-8969-05fce5865111",
  "wf-id":"ABC-SIMPLE-CHILD-WORKFLOW-ID",
  "wf-run-id":"e4b9f903-85dc-4b1f-ab83-77e3e321775c",
  "store-operation":"create-wf-execution",
  "error":"Encounter shard ownership lost, request range ID: 55456, actual range ID: 0",
  "logging-call-at":"transaction_impl.go:344",
  "stacktrace":"go.temporal.io/server/common/log.(*zapLogger).Error\n\t/temporal/common/log/zap_logger.go:142\ngo.temporal.io/server/service/history/workflow.createWorkflowExecutionWithRetry\n\t/temporal/service/history/workflow/transaction_impl.go:344\ngo.temporal.io/server/service/history/workflow.(*ContextImpl).CreateWorkflowExecution\n\t/temporal/service/history/workflow/context.go:343\ngo.temporal.io/server/service/history.(*historyEngineImpl).StartWorkflowExecution\n\t/temporal/service/history/historyEngine.go:580\ngo.temporal.io/server/service/history.(*Handler).StartWorkflowExecution\n\t/temporal/service/history/handler.go:527\ngo.temporal.io/server/api/historyservice/v1._HistoryService_StartWorkflowExecution_Handler.func1\n\t/temporal/api/historyservice/v1/service.pb.go:931\ngo.temporal.io/server/common/rpc/interceptor.(*RateLimitInterceptor).Intercept\n\t/temporal/common/rpc/interceptor/rate_limit.go:84\ngoogle.golang.org/grpc.chainUnaryInterceptors.func1.1\n\t/go/pkg/mod/google.golang.org/grpc@v1.44.0/server.go:1116\ngo.temporal.io/server/common/rpc/interceptor.(*TelemetryInterceptor).Intercept\n\t/temporal/common/rpc/interceptor/telemetry.go:108\ngoogle.golang.org/grpc.chainUnaryInterceptors.func1.1\n\t/go/pkg/mod/google.golang.org/grpc@v1.44.0/server.go:1119\ngo.temporal.io/server/common/metrics.NewServerMetricsTrailerPropagatorInterceptor.func1\n\t/temporal/common/metrics/grpc.go:113\ngoogle.golang.org/grpc.chainUnaryInterceptors.func1.1\n\t/go/pkg/mod/google.golang.org/grpc@v1.44.0/server.go:1119\ngo.temporal.io/server/common/metrics.NewServerMetricsContextInjectorInterceptor.func1\n\t/temporal/common/metrics/grpc.go:66\ngoogle.golang.org/grpc.chainUnaryInterceptors.func1.1\n\t/go/pkg/mod/google.golang.org/grpc@v1.44.0/server.go:1119\ngo.temporal.io/server/common/rpc.ServiceErrorInterceptor\n\t/temporal/common/rpc/grpc.go:131\ngoogle.golang.org/grpc.chainUnaryInterceptors.func1.1\n\t/go/pkg/mod/google.golang.org/grpc@v1.44.0/server.go:1119\ngoogle.golang.org/grpc.chainUnaryInterceptors.func1\n\t/go/pkg/mod/google.golang.org/grpc@v1.44.0/server.go:1121\ngo.temporal.io/server/api/historyservice/v1._HistoryService_StartWorkflowExecution_Handler\n\t/temporal/api/historyservice/v1/service.pb.go:933\ngoogle.golang.org/grpc.(*Server).processUnaryRPC\n\t/go/pkg/mod/google.golang.org/grpc@v1.44.0/server.go:1282\ngoogle.golang.org/grpc.(*Server).handleStream\n\t/go/pkg/mod/google.golang.org/grpc@v1.44.0/server.go:1616\ngoogle.golang.org/grpc.(*Server).serveStreams.func1.2\n\t/go/pkg/mod/google.golang.org/grpc@v1.44.0/server.go:921"
}
```