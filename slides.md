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

---

# Connect RPC

- **A mandatory proxy for translating between gRPC-Web requests and gRPC HTTP/2 responses.**
  - https://grpc.io/blog/state-of-grpc-web/
- `grpc-web`에서 일부 기능 미지원 (Client-side streaming, Bidrectional streaming)
  - https://github.com/grpc/grpc-web/blob/9856bfea5d9d301d3783d220cccf41e869dfbdf1/doc/streaming-roadmap.md
- Node.js 환경에서의 여러가지 이슈들
  - `grpc-web`은 `XMLHttpRequest`를 사용하는데 Node.js 환경에서는 XHR이 없음
    - https://github.com/yorkie-team/yorkie-js-sdk/issues/335
  - `grpc-web`의 `Uint8Array` 자체 구현으로 생기는 테스트 mock 이슈
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

# RPC in Yorkie

```proto
// yorkie.proto
service YorkieService {
  rpc ActivateClient (ActivateClientRequest) returns (ActivateClientResponse) {}
  // ...
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
    // ...
  },
};
```

---

# RPC in Yorkie

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

// call yorkie.v1.YorkieService.ActivateClient
this.rpcClient.activateClient(
  { clientKey: this.key },
  { headers: { 'x-shard-key': this.apiKey } },
)
```
