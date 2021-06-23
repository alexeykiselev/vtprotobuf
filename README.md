# `vtprotobuf`, the Vitess Protocol Buffers compiler

This repository provides the `protoc-gen-go-vtproto` plug-in for `protoc`, which is used by Vitess to generate optimized marshall & unmarshal code.

The code generated by this compiler is based on the optimized code generated by [`gogo/protobuf`](https://github.com/gogo/protobuf), although this package is not a fork of the original `gogo` compiler, as it has been implemented to support the new ProtoBuf APIv2 packages.

## Available features

`vtprotobuf` is implemented as a helper plug-in that must be run **alongside** the upstream `protoc-gen-go` generator, as it generates fully-compatible auxiliary code to speed up (de)serialization of Protocol Buffer messages.

The following features can be generated:

- `size`: generates a `func (p *YourProto) SizeVT() int` helper that behaves identically to calling `proto.Size(p)` on the message, except the size calculation is fully unrolled and does not use reflection. This helper function can be used directly, and it'll also be used by the `marshal` codegen to ensure the destination buffer is properly sized before ProtoBuf objects are marshalled to it.

- `marshal`: generates the following helper methods

    - `func (p *YourProto) MarshalVT() ([]byte, error)`: this function behaves identically to calling `proto.Marshal(p)`, except the actual marshalling has been fully unrolled and does not use reflection or allocate memory. This function simply allocates a properly sized buffer by calling `SizeVT` on the message and then uses `MarshalToSizedBufferVT` to marshal to it.

    - `func (p *YourProto) MarshalToVT(data []byte) (int, error)`: this function can be used to marshal a message to an existing buffer. The buffer must be large enough to hold the marshalled message, otherwise this function will panic. It returns the number of bytes marshalled. This function is useful e.g. when using memory pooling to re-use serialization buffers.

    - `func (p *YourProto) MarshalToSizedBufferVT(data []byte) (int, error)`: this function behaves like `MarshalTo` but expects that the input buffer has the exact size required to hold the message, otherwise it will panic.

- `unmarshal`: generates a `func (p *YourProto) UnmarshalVT(data []byte)` that behaves similarly to calling `proto.Unmarshal(data, p)` on the message, except the unmarshalling is performed by unrolled codegen without using reflection and allocating as little memory as possible. If the receiver `p` is **not** fully zeroed-out, the unmarshal call will actually behave like `proto.Merge(data, p)`. This is because the `proto.Unmarshal` in the ProtoBuf API is implemented by resetting the destionation message and then calling `proto.Merge` on it. To ensure proper `Unmarshal` semantics, ensure you've called `proto.Reset` on your message before calling `UnmarshalVT`, or that your message has been newly allocated.

- `pool`: generates the following helper methods

    - `func (p *YourProto) ResetVT()`: this function behaves similarly to `proto.Reset(p)`, except it keeps as much memory as possible available on the message, so that further calls to `UnmarshalVT` on the same message will need to allocate less memory. This an API meant to be used with memory pools and does not need to be used directly.

    - `func (p *YourProto) ReturnToVTPool()`: this function returns message `p` to a local memory pool so it can be reused later. It clears the object properly with `ResetVT` before storing it on the pool. This method should only be used on messages that were obtained from a memory pool by calling `YourProtoFromVTPool`. **Using `p` after calling this method will lead to undefined behavior**.

    - `func YourProtoFromVTPool() *YourProto`: this function returns a `YourProto` message from a local memory pool, or allocates a new one if the pool is currently empty. The returned message is always empty and ready to be used (e.g. by calling `UnmarshalVT` on it). Once the message has been processed, it must be returned to the memory pool by calling `ReturnToVTPool()` on it. Returning the message to the pool is not mandatory (it does not leak memory), but if you don't return it, that defeats the whole point of memory pooling.

## Usage

1. Install `protoc-gen-go-vtproto`:

```
go install github.com/planetscale/vtprotobuf/cmd/protoc-gen-go-vtproto
```

2. Ensure your project is already using the ProtoBuf v2 API (i.e. `google.golang.org/protobuf`). The `vtprotobuf` compiler is not compatible with APIv1 generated code.

2. Update your `protoc` generator to use the new plug-in. Example from Vitess:

```
for name in $(PROTO_SRC_NAMES); do \
    $(VTROOT)/bin/protoc \
    --go_out=. --plugin protoc-gen-go="${GOBIN}/protoc-gen-go" \
    --go-grpc_out=. --plugin protoc-gen-go-grpc="${GOBIN}/protoc-gen-go-grpc" \
    --go-vtproto_out=. --plugin protoc-gen-go-vtproto="${GOBIN}/protoc-gen-go-vtproto" \
    --go-vtproto_opt=features=marshal+unmarshal+size \
    proto/$${name}.proto; \
done
```

Note that the `vtproto` compiler runs like an auxiliary plug-in to the `protoc-gen-go` in APIv2, just like the new GRPC compiler plug-in, `protoc-gen-go-grpc`. You need to run it alongside the upstream generator, not as a replacement.

4. (Optional) Pass the features that you want to generate as `--go-vtproto_opt`. If no features are given, all the codegen steps will be performed.

5. Compile the `.proto` files in your project. You should see `_vtproto.pb.go` files next to the `.pb.go` and `_grpc.pb.go` files that were already being generated.

6. (Optional) Switch your RPC framework to use the optimized helpers (see following sections)

## Using the optimized code with RPC frameworks

The `protoc-gen-go-vtproto` compiler does not overwrite any of the default marshalling or unmarshalling code for your ProtoBuf objects. Instead, it generates helper methods that can be called explicitly to opt-in to faster (de)serialization.

### `vtprotobuf` with GRPC

To use `vtprotobuf` with the new versions of GRPC, you need to register the codec provided by the `github.com/planetscale/vtprotobuf/codec/grpc` package.

```go
package servenv

import (
    "github.com/planetscale/vtprotobuf/codec/grpc"
	"google.golang.org/grpc/encoding"
	_ "google.golang.org/grpc/encoding/proto"
)

func init() {
	encoding.RegisterCodec(grpc.Codec{})
}

```

Note that we perform a blank import `_ "google.golang.org/grpc/encoding/proto"` of the default `proto` coded that ships with GRPC to ensure it's being replaced by us afterwards. The provided Codec will serialize & deserialize all ProtoBuf messages using the optimized codegen.

#### Mixing ProtoBuf implementations with GRPC

If you're running a complex GRPC service, you may need to support serializing ProtoBuf messages from different sources, including from external packages that will not have optimized `vtprotobuf` marshalling code. This is perfectly doable by implementing a custom codec in your own project that serializes messages based on their type. The Vitess project [implements a custom codec](https://github.com/vitessio/vitess/blob/main/go/vt/servenv/grpc_codec.go) to support ProtoBuf messages from Vitess itself and those generated by the `etcd` API -- you can use it as a reference.

### Twirp

I actually have no idea of how to switch encoders in Twirp. Maybe it's not even possible.

### DRPC

To use `vtprotobuf` as a DRPC encoding, simply pass `github.com/planetscale/vtprotobuf/codec/drpc` as the `protolib` flag in your `protoc-gen-go-drpc` invocation.

Example:

```
protoc --go_out=. --go-vtproto_out=. --go-drpc_out=. --go-drpc_opt=protolib=github.com/planetscale/vtprotobuf/codec/drpc
```

