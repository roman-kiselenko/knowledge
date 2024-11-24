---
title: Go Gets New Reflection-Based Protocol Buffers API
source: https://www.infoq.com/news/2020/03/go-protobuf-apiv2/
clipped: 2023-09-04
published: 
category: development
tags:
  - grpc
  - protobuf
read:
---

Go's new bindings for [protocol buffers](https://developers.google.com/protocol-buffers), Google's language-neutral data interchange format that aims to replace JSON for high-performance applications, aim to [merge the protocol buffer type system within Go's type system](https://blog.golang.org/a-new-go-api-for-protocol-buffers) and enable its manipulation at runtime.

Protocol buffers provide a way to specify the schema used for transmitting structured data. This schema is usually translated into a language-specific representation, called bindings, which makes it easier to work with protobuf messages using a higher-level interface.

According to Joe Tsai, Damien Neil, and Herbie Ong, authors of the new [`protobuf` module version](https://pkg.go.dev/mod/google.golang.org/protobuf), the [previous `protobuf` implementation](https://github.com/golang/protobuf) was not up to Go developers' expectations anymore. Specifically, it provided a restricted view of the protobuf type system mapped on to Go's less rich type system.

A consequence of this was, for example, the loss of protobuf annotations, which are not part of Go type system. This made it impossible, for example, to process a message log and remove all sensitive information.

Another limitation of the older `protobuf` module was its reliance on static bindings, which barred the possibility to work with dynamic messages whose type was not entirely know at compile time.

The approach taken by the developers of the new `protobuf` module, versioned as APIv2, relies on the assumption that a `protobuf` `Message` has to fully specify the behaviour of a message and on the use of reflection to provide a full view of protobuf types. Cornerstone to Go protobuf APIv2 is the new `proto.Message` interface, which is common to all generated message types and provides the means to access a message content. This includes any protobuf fields, which can be iterated on using the [`protoreflect.Message.Range`](https://pkg.go.dev/google.golang.org/protobuf/reflect/protoreflect?tab=doc#Message.Range) method. This approach enables both working with dynamic messages as well accessing message options. The following is an example of how you can process a message to clear all sensitive information it contains before further processing:

```

func Redact(pb proto.Message) {
    m := pb.ProtoReflect()
    m.Range(func(fd protoreflect.FieldDescriptor, v protoreflect.Value) bool {
        opts := fd.Options().(*descriptorpb.FieldOptions)
        if proto.GetExtension(opts, policypb.E_NonSensitive).(bool) {
            return true
        }
        m.Clear(fd)
        return true
    })
}
```

Versioning for the Go protobuf APIv2 is [kind of confusing, as several Hacker News commenters pointed out](https://news.ycombinator.com/item?id=22468494). Indeed, the developers decided to tie the new version to a specific repository, instead of extending the existing repository with the new version and tag it as `v2`. Damien Neil explained the reasoning that was behind this decision in the following terms:

> We could tag the new API v2:
> 
> -   Makes the "v1" and "v2" distinction very clear in the import path.
>     
> -   Confusing: google.golang.org/protobuf@v1 doesn't exist, but v2 does.
>     
> -   In ten years, hopefully nobody cares about the old github.com/golang/protobuf and the confusion is gone.
>     
> 
> We could tag the new API v1:
> 
> -   Less visually distinct in the import path.
>     
> -   Seems to make sense for the first version of google.golang.org/protobuf to be a v1.
>     
> -   If we decide it was a terrible idea, it's easier to go from v1 to v2 than to roll back from v2 to v1.
>     

Additionally, Go protobuf APIv2 starts at version 1.20. This was aimed to avoid any ambiguities with bug reports mentioning overlapping versions, as Neil also explained, counting on the fact the Go protobuf APIv1 will likely never reach version 1.20.

As a final note, APIv1 is not obsoleted by APIv2 and will be maintained forever. As a matter of fact, its latest implementation, version 1.4, is actually implemented on top of APIv2.