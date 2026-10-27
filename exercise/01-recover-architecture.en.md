**Language:** [ภาษาไทย](01-recover-architecture.md) | English

# Task 1: Recover the architecture (approximately 15 min)

> **Lecture link:** T7 architecture recovery, AICH2 (traceability), AICH3 (context), and the JSS finding that 93% of studies never validate AI output.
> **Question:** how much of an architecture can AI recover, and how much does it depend on the evidence it reads?

**Don't look at the original Online Boutique repository during this task:** its README shows the architecture diagram you are recovering.

For each run, start a **new chat or session** in your agent and send the message below. The agent follows [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) and writes its result to `exercise/outputs/1<run>-architecture.md`.

**Run A: interfaces only**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interfaces + deployment config**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

While the agent works, watch which files it opens. If it reads files outside the evidence you allowed, write that down in your prompt log.

## Results

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

Open each file with **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) to see the diagram.

## Verify Run B yourself (the important part)

Check **at least five** dependencies in Run B's table against the files. Open the cited file and find the evidence. You may also check the service's source code, e.g. `src/checkoutservice/main.go`.

| Caller → Callee | AI's evidence | Verified? ✅ / ❌ / ❓ | What you found |
|---|---|---|---|
| `frontend` → `checkoutservice` | `frontend.yaml` (`CHECKOUT_SERVICE_ADDR=checkoutservice:5050`) | ✅ | `src/frontend/handlers.go:354` calls `pb.NewCheckoutServiceClient(fe.checkoutSvcConn)` |
| `frontend` → `shippingservice` | `frontend.yaml` (`SHIPPING_SERVICE_ADDR`) | ✅ | `src/frontend/rpc.go:88` calls `NewShippingServiceClient(...).GetQuote` |
| `frontend` → `adservice` | `frontend.yaml` (`AD_SERVICE_ADDR=adservice:9555`) | ✅ | `src/frontend/rpc.go:123` calls `NewAdServiceClient(...).GetAds` |
| `checkoutservice` → `emailservice` | `checkoutservice.yaml` (`EMAIL_SERVICE_ADDR=emailservice:5000`) | ✅ | `src/checkoutservice/main.go:380` `NewEmailServiceClient(...).SendOrderConfirmation`; the client address is `emailservice:5000`, and the Service maps 5000 → pod port 8080, so the manifest is consistent |
| `cartservice` → `redis-cart` | `cartservice.yaml` (`REDIS_ADDR=redis-cart:6379`) | ✅ | `src/cartservice/src/Startup.cs:29` reads `REDIS_ADDR` and registers `RedisCartStore` |
| `recommendationservice` → `productcatalogservice` | `recommendationservice.yaml` (`PRODUCT_CATALOG_SERVICE_ADDR`) | ✅ | `src/recommendationservice/recommendation_server.py:131-133` reads the variable and fails if it is not set |
| `loadgenerator` → `frontend` | `loadgenerator.yaml` (`FRONTEND_ADDR=frontend:80`) | ❓ | I only confirmed the env var and the init-container check in the manifest, not the Locust code. Note that `loadgenerator` is not in `kustomization.yaml`, so it is probably a test tool, not part of the product |

- **Wrong or missing** in Run B (false dependencies, missing ones, containers to question):
  - **Missing container:** `src/shoppingassistantservice/` exists, and `src/frontend/handlers.go:464` calls it with `"http://" + fe.shoppingAssistantSvcAddr`. Run B saw the env var `SHOPPING_ASSISTANT_SERVICE_ADDR=shoppingassistantservice:80` but there is no manifest for it in the allowed folder. It put this under Gaps and left it out of the diagram, which is the correct behaviour with that evidence, but the diagram is still incomplete. It is also an **HTTP** dependency, unlike all the gRPC ones.
  - **Missing:** the tracing collector. `COLLECTOR_SERVICE_ADDR` is read by `frontend/main.go:178` and by `checkoutservice/main.go:164-165` (only when `ENABLE_TRACING=1`). It is not in the manifests Run B read.
  - **Questionable container:** `loadgenerator` is drawn inside the system boundary, although it is a test tool and not listed in `kustomization.yaml`.
  - **Protocol "gRPC" for every internal arrow** is inferred, not stated in the env vars. It happens to be right for those I checked.
  - **Env vars are not call sites:** Run B treats every `*_ADDR` as a real call. For the ones I checked this was true, but it is an assumption for the rest (I did not check `frontend` → `recommendationservice` and `frontend` → `currencyservice` in the source).

## Reflect (2–3 sentences each)

1. Compare A and B. What could the agent **not** know from the `.proto` file alone, and why?
   A `.proto` file only describes what each service **offers** (its server interface), so Run A found 9 services and zero dependencies. It could not know who calls whom, which port or protocol is used, that Redis exists behind `cartservice`, or that a frontend and a load generator exist. Those are deployment and client-side facts, which live in the manifests and the source, not in the interface definition.
2. Which **one** mistake would mislead a new team member most if the diagram were published without checking?
   Leaving out `shoppingassistantservice` (and the HTTP link from `frontend` to it). A new team member would believe that the diagram lists every service and every protocol and that all internal calls are gRPC, then be surprised by a service, with its own failure modes, that is called over plain HTTP and appears in no diagram.
