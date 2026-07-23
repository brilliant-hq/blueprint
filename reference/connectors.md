---
assumes: blueprint/core, blueprint/lines
dsl: [line, from, to, route, avoid, intent]
---
# Blueprint Connectors

`connect(...)` was removed. Use `line(...)` endpoint form for everything
connect did:

```
line(from(#a), to(#b)) intent(flow)
line(from(#a), to(#b), route(elbow), avoid(all)) cap(n,ar) st[...]
line(from(#a), to(#b, r, 0.25))            anchor 25% down b's right edge
line(from(#card_port), to(#b))             nested-child anchor on the parent
```

Endpoint-form lines also support coord endpoints (`from(x,y)`), which is
useful for callouts, fixed-point annotations, and any "draw between
these two specific points" need — not possible with connect().

Full reference: see `blueprint/lines`.
