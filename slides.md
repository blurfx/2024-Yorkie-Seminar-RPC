---
theme: geist
class: text-center
highlighter: shiki
lineNumbers: true
title: Welcome to Slidev
colorSchema: light
mdc: true
fonts:
  mono: 'Fira Code'
layout: center
---

# Remote Procedure Call

---

# 서버와 통신하기

가장 고전적인 통신 방법: 소켓 통신

- 서버와 클라이언트가 계속 연결을 맺고 있어야 하고 네트워크가 최상의 상태를 유지하고 있어야 함
- 현실: 네트워크의 속도는 불안정하고, 중간에 손실이 있을 수 있으며 더 나쁘면 연결이 끊길 수도 있음
- -> 비즈니스 로직 외에도 네트워크 연결을 관리해야 함

---

# Remote Procedure Call

- 네트워크의 연결 상태를 신경쓰지 않고, 원격에 있는 함수를 호출하는 방법
- 말 그대로 방법을 말하는 것이라 표준이 딱히 없습니다.
- 일반적으로 Interface Definition Language이라는 서버-클라이언트 간의 인터페이스를 정의
- 웹 환경에서 사용할 수 있도록 XML-RPC, JSON-RPC 등이 등장함

```jsonc
// JSON-RPC 예시
Request  {"jsonrpc": "2.0", "method": "sum", "params": {"x": 10, "y": 20}, "id": 3}
Response {"jsonrpc": "2.0", "result": 30, "id": 3}
```

---

# REST(or HTTP) API

웹 환경에 조금 더 알맞고, 객체 지향적인 관점으로 API를 설계하기 시작함
  - `GET /proucts/[id]`
  - `POST /products`
  - `PUT /products/[id]`
  - `DELETE /products/[id]`

---

# gRPC

- 구글에서 만든, 가장 흔히 사용되는 RPC 프레임워크
- Protocol Buffers를 메시지 포맷/인터페이스로 사용함
  ```proto
  message Person {
    optional int32 id = 1;
    optional string name = 2;
    optional string email = 3;
  }
  ```
- Unary RPC, Server-side streaming, Client-side streaming, Bidirectional streaming 을 지원함

---

# Protocol Buffers

```proto
message Person {
  optional int32 id = 1;
  optional string name = 2;
  optional string email = 3;
}
```

```jsonc
// (47 byte)
{"name":"blurfx","id":666,"email":"iam@xo.dev"}
```

```js
// (23 byte)
0x089a051206626c757266781a0a69616d40786f2e64657
```

```jsonc
// (85 byte)
{"name":"blurfx","id":666,"emailllllllllllllllllllllllllllllllllllllll":"iam@xo.dev"}
```

---

# 현대의 RPC 프레임워크
- gRPC
- Apache Thrift
- Cap'n Proto
- ...
- Connect <- Yorkie가 씀

---

# Connect RPC

Connect is a family of libraries for building browser and **gRPC-compatible** APIs

왜 Yorkie는 사용하던 gRPC를 떠나 Connect을 사용하게 되었을까?

- https://github.com/yorkie-team/yorkie/issues/668
- grpc-web
  - **A mandatory proxy for translating between gRPC-Web requests and gRPC HTTP/2 responses.**

---

# Connect RPC

  - https://grpc.io/blog/state-of-grpc-web/
- `grpc-web`에서 일부 기능 미지원 (Client-side streaming, Bidrectional streaming)
  - https://github.com/grpc/grpc-web/blob/9856bfea5d9d301d3783d220cccf41e869dfbdf1/doc/streaming-roadmap.md
- Node.js 환경에서의 여러가지 이슈들
  - `grpc-web`은 `XMLHttpRequest`를 사용하는데 Node.js 환경에서는 XHR이 없음 (polyfill 필요함)
    - https://github.com/yorkie-team/yorkie-js-sdk/issues/335
  - `grpc-web`의 `Uint8Array` 자체 구현으로 생기는 테스트 mock 이슈 (역시 polyfill 필요함)
    - https://github.com/yorkie-team/yorkie-js-sdk/pull/655

---

# Connect RPC

Migrate RPC to ConnectRPC
  - https://github.com/yorkie-team/yorkie/pull/703

---

# RPC in Yorkie

- Yorkie
  - `api/yorkie/v1/*.proto`
- Yorkie-JS-SDK
  - `src/api/yorkie/v1/*.proto`

---

# RPC in Yorkie: Client

- Protobuf로 인터페이스 작성
```proto
// yorkie.proto
service YorkieService {
  rpc ActivateClient (ActivateClientRequest) returns (ActivateClientResponse) {}
}
```

- `buf`로 자바스크립트 코드 생성

```sh
npx buf generate
```

---

# RPC in Yorkie: Client

- Protobuf로 인터페이스 작성
```proto
// yorkie.proto
service YorkieService {
  rpc ActivateClient (ActivateClientRequest) returns (ActivateClientResponse) {}
}
```

```ts
// yorkie_connect.js (generated)
export const YorkieService = {
  typeName: "yorkie.v1.YorkieService",
  methods: {
    /**
      * @generated from rpc yorkie.v1.YorkieService.ActivateClient
      */
    activateClient: {
      name: "ActivateClient",
      I: ActivateClientRequest,
      O: ActivateClientResponse,
      kind: MethodKind.Unary,
    },
  },
};
```

---

# RPC in Yorkie: Client

- JS에서 RPC 클라이언트 생성
```ts
// create rpc client
this.rpcClient = createPromiseClient(
  YorkieService,
  createGrpcWebTransport({
    baseUrl: rpcAddr,
    interceptors: [
      createAuthInterceptor(opts.apiKey, opts.token),
      createMetricInterceptor(),
    ],
  }),
);
```

- 원격 함수 호출하기
```ts
// call yorkie.v1.YorkieService.ActivateClient
this.rpcClient.activateClient(
  { clientKey: this.key },
  { headers: { 'x-shard-key': this.apiKey } },
)
```

---

# RPC in Yorkie: Server

- RPC 함수 작성하기
```go
// server/rpc/yorkie_server.go
func (s *yorkieServer) ActivateClient(
	ctx context.Context,
	req *connect.Request[api.ActivateClientRequest],
) (*connect.Response[api.ActivateClientResponse], error) {
  // ...
	return connect.NewResponse(&api.ActivateClientResponse{
		ClientId: cli.ID.String(),
	}), nil
}
```

---

# RPC in Yorkie: Server

- RPC 이벤트 핸들러 연결하기
```go
// api/yorkie/v1/v1connect/yorkie.connect.go
func NewYorkieServiceHandler(svc YorkieServiceHandler, opts ...connect.HandlerOption) (string, http.Handler) {
	yorkieServiceActivateClientHandler := connect.NewUnaryHandler(
		YorkieServiceActivateClientProcedure,
		svc.ActivateClient,
		opts...,
	)
	// ...
	return "/yorkie.v1.YorkieService/", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		switch r.URL.Path {
		case YorkieServiceActivateClientProcedure:
			yorkieServiceActivateClientHandler.ServeHTTP(w, r)

			// ...
		}
	})
}
```

---

# RPC in Yorkie: Server

- HTTP 요청 멀티플렉서 생성하고 RPC 핸들러 연결하기
```go
// server/rpc/server.go
mux := http.NewServeMux()
mux.Handle(v1connect.NewYorkieServiceHandler(newYorkieServer(yorkieServiceCtx, be), opts...))
```

---

# Example (CodePair)

<img src="/req1.png" style="max-height:100px;">


<img src="/payload1.png" style="max-height:100px;">


payload (string)
```
\u0000\u0000\u0000\u00004\n\u001866d2a5d8f598806fa0013466\u0012\u001866cd994df598806fa0b0a887
```

hexadecimal
```
0x00000000340a183636643261356438663539383830366661303031333436361218363663643939346466353938383036666130623061383837
```
---

# Example (CodePair)

![](/result1.png)


```proto
message WatchDocumentRequest {
  string client_id = 1;
  string document_id = 2;
}
```

```json
{
  client_id: "66d2a5d8f598806fa0013466",
  document_id: "66cd994df598806fa0b0a887"
}
```
