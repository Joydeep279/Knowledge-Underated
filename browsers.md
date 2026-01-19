# The Web Browser: A Deep Systems-Level Analysis

## Executive Overview

A web browser is a **multi-process, event-driven runtime** that orchestrates dozens of subsystems: network protocols, parsers, rendering pipelines, JavaScript VMs, GPU compositing, and sandboxed execution contexts. At its core, it's a distributed operating system managing hostile code from arbitrary origins.

Let me dissect this from the silicon up.

---

## I. Architecture: The Multi-Process Model

### 1.1 Process Taxonomy (Chromium Reference)

Modern browsers abandoned single-process designs after Firefox's Quantum and Chrome's Site Isolation projects. The architecture resembles a microkernel OS:

```
┌─────────────────────────────────────────────────────────────┐
│                      BROWSER PROCESS                         │
│  - UI thread (tabs, omnibox, bookmarks)                     │
│  - IO thread (IPC broker, network service dispatcher)       │
│  - File thread (safe browsing, download manager)            │
└────────────────────┬────────────────────────────────────────┘
                     │ IPC (Mojo/Chrome, IPDL/Firefox)
        ┌────────────┼────────────┬─────────────┐
        │            │            │             │
┌───────▼──────┐ ┌──▼─────┐ ┌────▼────┐ ┌──────▼──────┐
│   RENDERER   │ │RENDERER│ │  GPU     │ │  NETWORK    │
│   PROCESS    │ │PROCESS │ │ PROCESS  │ │  SERVICE    │
│              │ │        │ │          │ │             │
│ - Blink/Gecko│ │ (per   │ │ - GL cmds│ │ - HTTP/3    │
│ - V8/Spider  │ │  site) │ │ - Skia   │ │ - DNS       │
│ - Layout tree│ │        │ │ - Raster │ │ - TLS 1.3   │
└──────────────┘ └────────┘ └──────────┘ └─────────────┘
```

**Source Reference** (`chromium/content/browser/browser_main_loop.cc`):
```cpp
void BrowserMainLoop::CreateStartupTasks() {
  // Network service runs in isolated process
  content::GetNetworkService();

  // GPU process for all rendering
  content::GpuProcessHost::Get();

  // Renderer process per site (after Site Isolation)
  content::RenderProcessHostImpl::CreateRenderProcessHost();
}
```

### 1.2 Why Multi-Process?

**Security**: Each renderer is sandboxed (via `seccomp-bpf` on Linux, Job Objects on Windows). Compromised renderer cannot access:
- File system (no `open()` syscalls)
- Network sockets (network service is separate)
- Other site's memory (ASLR + process isolation)

**Stability**: One tab crash doesn't kill the browser. The kernel's process accounting (`task_struct` in Linux) provides fault isolation.

**Performance**: True parallelism across CPU cores. Renderer processes scale with `sched_setaffinity()` pinning to prevent cache thrashing.

---

## II. The Critical Path: URL to Pixels

### 2.1 Navigation Initiation

When you type `https://example.com`:

**Step 1: Browser Process → Network Service IPC**
```cpp
// chromium/content/browser/frame_host/navigation_request.cc
void NavigationRequest::BeginNavigation() {
  network::ResourceRequest request;
  request.url = url_;
  request.method = "GET";
  request.headers.SetHeader("User-Agent", GetUserAgent());

  // Mojo IPC to network service
  url_loader_factory_->CreateLoaderAndStart(
      loader.BindNewPipeAndPassReceiver(),
      request,
      base::BindOnce(&NavigationRequest::OnResponseStarted));
}
```

### 2.2 Network Stack Deep Dive

The network service is a **state machine** managing:

#### DNS Resolution
```
User input → DNS cache (chrome://net-internals/#dns)
          ↓ (miss)
       Platform resolver (getaddrinfo/WinHTTP)
          ↓
       DNS-over-HTTPS (if enabled)
          ↓
       UDP:53 → authoritative nameserver
```

**Implementation** (`net/dns/host_resolver_manager.cc`):
```cpp
HostResolverManager::Job::Start() {
  // 1. Check in-memory cache
  if (auto* cached = cache_.Lookup(hostname))
    return OnComplete(cached->addresses);

  // 2. Check OS resolver
  if (use_system_resolver_)
    return StartSystemResolve();

  // 3. Use async DNS (DnsTransaction)
  dns_task_ = std::make_unique<DnsTask>(
      dns_client_, hostname_,
      base::BindOnce(&Job::OnDnsTaskComplete));
}
```

#### TCP Connection (HTTP/1.1) or QUIC (HTTP/3)

For HTTPS with HTTP/2:
```
DNS complete → Socket pool allocates endpoint
            ↓
         TCP SYN → SYN-ACK (kernel TCP state machine)
            ↓
         TLS 1.3 ClientHello (with SNI, ALPN:h2)
            ↓
         ServerHello (session resumption via PSK if cached)
            ↓
         Certificate verification (trust store check)
            ↓
         Application data over TLS record layer
```

**Code Path** (`net/socket/ssl_client_socket_impl.cc`):
```cpp
int SSLClientSocketImpl::DoHandshake() {
  // OpenSSL/BoringSSL state machine
  int rv = SSL_do_handshake(ssl_.get());

  if (rv == 1) {
    // Handshake complete - extract negotiated protocol
    const uint8_t* proto;
    unsigned proto_len;
    SSL_get0_alpn_selected(ssl_.get(), &proto, &proto_len);

    if (proto_len == 2 && memcmp(proto, "h2", 2) == 0)
      negotiated_protocol_ = kProtoHTTP2;
  }
  return rv;
}
```

#### HTTP/2 Framing

HTTP/2 multiplexes requests over single TCP connection using binary frames:

```
HEADERS frame (stream 1): GET /index.html
HEADERS frame (stream 3): GET /style.css
DATA frame (stream 1): <html>...
DATA frame (stream 3): body { margin: 0 }
```

**Implementation** (`net/spdy/spdy_session.cc`):
```cpp
void SpdySession::SendRequest(SpdyStreamRequest* request) {
  spdy::SpdySerializedFrame frame =
      buffered_spdy_framer_->CreateHeaders(
          stream_id,
          priority,
          spdy::SpdyHeaderBlock(request->headers()));

  EnqueueWrite(priority, spdy::HEADERS,
               std::move(frame), stream->GetWeakPtr());
}
```

### 2.3 Response Handling & Renderer Selection

Network service receives HTTP 200:
```
HTTP/2 200 OK
content-type: text/html; charset=utf-8
content-length: 4521
x-frame-options: SAMEORIGIN
```

**Browser process decision tree**:
```cpp
// content/browser/loader/navigation_url_loader_impl.cc
void NavigationURLLoaderImpl::OnReceiveResponse() {
  // 1. Check content-type
  if (response->mime_type == "text/html") {
    // 2. Determine site for process isolation
    SiteInstance* site = SiteInstance::CreateForURL(url);

    // 3. Allocate or reuse renderer
    RenderProcessHost* rph = site->GetProcess();

    // 4. Send response to renderer's IO thread
    rph->Send(new FrameMsg_CommitNavigation(
        routing_id, response, body_stream));
  }
}
```

---

## III. Rendering Engine: Blink/Gecko Internals

### 3.1 Parsing HTML → DOM Construction

The renderer receives HTML bytes:
```html
<!DOCTYPE html>
<html>
  <head><title>Test</title></head>
  <body><div id="app">Hello</div></body>
</html>
```

**Tokenization** (`third_party/blink/renderer/core/html/parser/html_tokenizer.cc`):

The HTML tokenizer is a **state machine** defined by WHATWG spec:
```cpp
bool HTMLTokenizer::NextToken(HTMLToken& token) {
  while (true) {
    switch (state_) {
      case HTMLTokenizer::kDataState:
        if (cc == '<') {
          state_ = kTagOpenState;
        } else {
          token.AppendToCharacter(cc);
        }
        break;

      case HTMLTokenizer::kTagOpenState:
        if (cc == '/') {
          state_ = kEndTagOpenState;
        } else if (IsASCIIAlpha(cc)) {
          token.BeginStartTag(ToLowerCaseIfAlpha(cc));
          state_ = kTagNameState;
        }
        break;

      // ... 80+ states total
    }
  }
}
```

**Tree Construction** (`html_tree_builder.cc`):

Tokens → DOM nodes using insertion modes:
```cpp
void HTMLTreeBuilder::ProcessToken(HTMLToken& token) {
  // Example: "initial" → "before html" → "before head" modes
  switch (insertion_mode_) {
    case kInBodyMode:
      if (token.GetType() == HTMLToken::kStartTag) {
        if (token.GetName() == html_names::kDivTag) {
          HTMLElement* element = CreateElement(token);
          tree_.OpenElements().Push(element);
        }
      }
      break;
  }
}
```

**Result**: DOM tree in memory:
```
Document
 └─ HTMLHtmlElement
     ├─ HTMLHeadElement
     │   └─ HTMLTitleElement
     │       └─ Text("Test")
     └─ HTMLBodyElement
         └─ HTMLDivElement (id="app")
             └─ Text("Hello")
```

Each node is a C++ object inheriting from `Node`:
```cpp
class Element : public ContainerNode {
  QualifiedName tag_name_;
  ElementData* element_data_;  // attributes
  ComputedStyle* computed_style_;  // CSS resolved values
  LayoutObject* layout_object_;  // box model
};
```

### 3.2 CSS Parsing & Style Resolution

**Parsing** (`css_parser_impl.cc`):
```css
#app { color: red; font-size: 16px; }
```

Tokenizer produces:
```
HASH_TOKEN("app")
LEFT_BRACE_TOKEN
IDENT_TOKEN("color")
COLON_TOKEN
IDENT_TOKEN("red")
SEMICOLON_TOKEN
...
```

Parser builds CSSOM:
```cpp
StyleRule* CSSParserImpl::ConsumeQualifiedRule() {
  // Parse selector: #app
  CSSSelectorList* selector_list =
      CSSSelectorParser::ParseSelector(...);

  // Parse declarations: { color: red; ... }
  CSSPropertyValueSet* properties =
      ConsumeDeclarationList(...);

  return StyleRule::Create(selector_list, properties);
}
```

**Style Recalc** (`style_engine.cc`):

This is where browser speed lives or dies. For each DOM element:

1. **Selector Matching**:
```cpp
ElementRuleCollector collector(element, style_resolver);
collector.MatchAllRules();  // O(rules × elements) without optimization

// Optimization: Bloom filter for ID/class/tag
if (!ancestor_bloom_filter_.MayContain(selector.tag))
  return;  // Skip expensive subtree walk
```

2. **Cascade & Inheritance**:
```cpp
ComputedStyle* StyleResolver::ResolveStyle(Element* element) {
  ComputedStyleBuilder builder;

  // 1. User-agent stylesheet (default styles)
  ApplyMatchedProperties(ua_rules, builder);

  // 2. Author styles (your CSS)
  ApplyMatchedProperties(author_rules, builder);

  // 3. Inheritance (font-size, color, etc.)
  if (element->ParentComputedStyle())
    builder.Inherit(*element->ParentComputedStyle());

  return builder.TakeStyle();
}
```

**Result**: Each element has `ComputedStyle` with resolved values:
```cpp
ComputedStyle {
  color: RGBA(255, 0, 0, 255)  // "red" → absolute value
  font_size: 16px              // relative units resolved
  display: block
  position: static
  // ... 400+ properties
}
```

### 3.3 Layout (Reflow)

**Layout Tree Construction**:

Not all DOM nodes get layout objects (e.g., `display: none`, `<script>`, `<head>`):

```cpp
LayoutObject* LayoutTreeBuilderForElement::CreateLayoutObject() {
  if (style.Display() == EDisplay::kNone)
    return nullptr;

  if (element.IsHTMLDivElement())
    return MakeGarbageCollected<LayoutBlockFlow>();

  if (element.IsHTMLSpanElement())
    return MakeGarbageCollected<LayoutInline>();
}
```

**Layout Algorithm** (`layout_block_flow.cc`):

This is CSS box model implementation:

```cpp
void LayoutBlockFlow::UpdateLayout() {
  // 1. Calculate constraints from parent
  ConstraintSpace space = CreateConstraintSpace();

  // 2. Layout children
  for (LayoutBox* child : Children()) {
    LayoutResult result = child->Layout(space);

    // 3. Position child (block direction stacking)
    LogicalOffset offset(0, current_offset);
    child->SetOffset(offset);
    current_offset += result.BlockSize();
  }

  // 4. Calculate self size
  SetLogicalWidth(space.AvailableSize().inline_size);
  SetLogicalHeight(current_offset);
}
```

**Flexbox Example** (`layout_flexible_box.cc`):
```cpp
void LayoutFlexibleBox::LayoutFlexItems() {
  // 1. Determine main axis (row/column)
  bool is_horizontal = IsHorizontalFlow();

  // 2. Calculate flex basis for each item
  for (FlexItem& item : items) {
    item.flex_base_content_size =
        ComputeFlexBasisForChild(item.box);
  }

  // 3. Resolve flexible lengths (flex-grow/shrink)
  ResolveFlexibleLengths(items);

  // 4. Align items (align-items, justify-content)
  AlignFlexItems(items);
}
```

**Output**: Layout tree with geometry:
```
LayoutBlockFlow (body)
  offset: (0, 0)
  size: (800px, 600px)

  LayoutBlockFlow (div#app)
    offset: (0, 0)
    size: (800px, 20px)

    LayoutText ("Hello")
      offset: (0, 0)
      size: (40px, 16px)
```

### 3.4 Paint (Rasterization Preparation)

**Paint Property Trees**:

Modern browsers use **layer trees** + **property trees** (transform, clip, effect):

```cpp
// paint/paint_property_tree_builder.cc
void PaintPropertyTreeBuilder::UpdateForSelf() {
  // Transform tree for CSS transforms
  if (NeedsTransformNode()) {
    transform_node_ = TransformPaintPropertyNode::Create(
        parent_transform_,
        TransformationMatrix().Translate(offset.x, offset.y));
  }

  // Effect tree for opacity, filters
  if (NeedsEffectNode()) {
    effect_node_ = EffectPaintPropertyNode::Create(
        parent_effect_,
        transform_node_,
        opacity_value_);
  }
}
```

**Display List Generation**:

```cpp
// paint/box_painter.cc
void BoxPainter::Paint() {
  GraphicsContext& context = paint_info.context;

  // 1. Background
  PaintBackground(context, layout_box_.StyleRef());

  // 2. Border
  PaintBorder(context, layout_box_.StyleRef());

  // 3. Children
  for (LayoutObject* child : layout_box_.Children()) {
    child->Paint(paint_info);
  }
}
```

Each paint operation is recorded as **display items**:
```
DrawingDisplayItem {
  type: kBoxDecorationBackground
  bounds: (0, 0, 800, 600)
  color: rgb(255, 255, 255)
}
DrawingDisplayItem {
  type: kForeground
  bounds: (0, 0, 40, 16)
  text: "Hello"
  font: Arial 16px
}
```

### 3.5 Compositing & GPU Acceleration

**Compositor Thread** (`cc/trees/layer_tree_host_impl.cc`):

Certain layers get promoted to **compositor layers** (separate GPU textures):

**Promotion Criteria**:
- `will-change: transform`
- 3D transforms (`translateZ(0)`)
- `<video>`, `<canvas>`, `<iframe>`
- Overlapping fixed/sticky elements

```cpp
bool CompositingReasonFinder::RequiresCompositingForTransform(
    const LayoutObject& layout_object) {
  // Promotes to GPU if has 3D transform
  if (layout_object.HasTransformRelatedProperty()) {
    const TransformationMatrix& matrix =
        layout_object.StyleRef().Transform();
    if (matrix.IsAffine() == false)  // has Z component
      return true;
  }
  return false;
}
```

**Rasterization** (`cc/raster/gpu_raster_buffer_provider.cc`):

Display lists → GPU commands:

```cpp
void GpuRasterBufferProvider::PlaybackToTexture() {
  // 1. Convert display list to Skia commands
  SkCanvas* canvas = CreateGpuCanvas(texture);

  // 2. Skia records to GPU command buffer
  for (DisplayItem* item : display_list) {
    if (item->IsDrawing()) {
      SkPaint paint = item->CreatePaint();
      canvas->drawRect(item->Bounds(), paint);
    }
  }

  // 3. Submit to GPU
  canvas->flush();  // GL calls: glDrawArrays, etc.
}
```

**Draw Quad Generation** (`cc/layers/picture_layer_impl.cc`):

For each visible tile, generate draw quad:
```cpp
void PictureLayerImpl::AppendQuads(QuadSink* quad_sink) {
  for (Tile* tile : visible_tiles) {
    TileDrawQuad* quad = TileDrawQuad::Create();
    quad->SetNew(
        shared_quad_state,
        tile->content_rect(),
        tile->opaque_rect(),
        tile->resource_id());
    quad_sink->Append(quad);
  }
}
```

**Compositor Frame**:

```
CompositorFrame {
  render_pass_list: [
    RenderPass {
      quad_list: [
        TileDrawQuad { texture_id: 42, rect: (0,0,256,256) }
        TileDrawQuad { texture_id: 43, rect: (256,0,256,256) }
      ]
      transform: translate(0, scroll_offset)
    }
  ]
}
```

Sent via IPC to GPU process → OpenGL/Vulkan rendering:

```cpp
// viz/service/display/gl_renderer.cc
void GLRenderer::DrawRenderPass(RenderPass* pass) {
  for (DrawQuad* quad : pass->quad_list) {
    if (quad->material == DrawQuad::TILED_CONTENT) {
      TileDrawQuad* tile = static_cast<TileDrawQuad*>(quad);

      // Bind texture
      glBindTexture(GL_TEXTURE_2D, tile->resource_id());

      // Draw two triangles (quad)
      glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_SHORT, indices);
    }
  }
}
```

---

## IV. JavaScript Engine (V8 Deep Dive)

### 4.1 Parsing & Abstract Syntax Tree

JavaScript arrives:
```javascript
function add(a, b) {
  return a + b;
}
add(5, 3);
```

**Scanner** (`v8/src/parsing/scanner.cc`):
```cpp
Token::Value Scanner::ScanSingleToken() {
  while (c0_ == ' ' || c0_ == '\t') Advance();  // Skip whitespace

  if (c0_ == 'f') {
    return ScanKeyword();  // FUNCTION token
  }

  if (IsDecimalDigit(c0_)) {
    return ScanNumber();  // NUMBER token
  }
}
```

**Parser** (`parser.cc`):

Recursive descent parser builds AST:
```cpp
FunctionLiteral* Parser::ParseFunctionLiteral() {
  Expect(Token::FUNCTION);
  Identifier* name = ParseIdentifier();

  // Parameters
  Expect(Token::LPAREN);
  ZonePtrList<Expression>* params = ParseFormalParameterList();
  Expect(Token::RPAREN);

  // Body
  ZonePtrList<Statement>* body = ParseBlock();

  return factory()->NewFunctionLiteral(name, params, body);
}
```

**AST**:
```
FunctionLiteral: add
  params: [a, b]
  body:
    ReturnStatement
      BinaryOperation: +
        left: Variable(a)
        right: Variable(b)
```

### 4.2 Bytecode Generation (Ignition)

**Bytecode Compiler** (`bytecode-generator.cc`):

```cpp
void BytecodeGenerator::VisitFunctionLiteral(FunctionLiteral* expr) {
  // Generate bytecode for function body
  Register return_value = register_allocator()->NewRegister();

  // Load 'a' and 'b' parameters
  builder()->LoadAccumulatorWithRegister(
      register_allocator()->Parameter(0));  // a

  builder()->BinaryOperation(
      Token::ADD,
      register_allocator()->Parameter(1));  // b

  builder()->Return();
}
```

**Bytecode Output**:
```
// Function add
LdaNamedProperty a0, [0]  // Load 'a' into accumulator
Add r1                     // Add with 'b' (in r1)
Return                     // Return accumulator
```

### 4.3 Execution: Interpreter → Optimizing Compiler

**Ignition Interpreter Loop** (`interpreter/interpreter.cc`):

```cpp
void Interpreter::DoAdd(InterpreterAssembler* assembler) {
  TNode<Object> left = GetAccumulator();
  TNode<Object> right = LoadRegister(bytecode_operand);

  // Type feedback: track what types we see
  TNode<Object> result = CallBuiltin(
      Builtins::kAdd,
      GetContext(), left, right, slot_index);

  SetAccumulator(result);
  Dispatch();  // Next bytecode
}
```

**TurboFan Optimization** (`compiler/pipeline.cc`):

After function runs ~100 times (hot threshold):

```cpp
void Pipeline::OptimizeGraph() {
  // 1. Build optimization graph (SSA form)
  GraphBuilder builder(bytecode);
  Graph* graph = builder.CreateGraph();

  // 2. Type inference from feedback
  Typer typer(graph, type_feedback);
  typer.Run();

  // 3. Optimize
  InliningOptimizer inliner(graph);
  inliner.Inline();  // Inline small functions

  RedundancyElimination eliminator(graph);
  eliminator.Run();  // Remove dead code

  // 4. Generate machine code
  InstructionSelector selector(graph);
  MachineCode* code = selector.SelectInstructions();

  // 5. Register allocation
  RegisterAllocator allocator(code);
  allocator.Allocate();

  // 6. Emit assembly
  Assembler assembler;
  assembler.Emit(code);
}
```

**Optimized Assembly** (x64):
```asm
;; function add(a, b)
mov rax, [rbp+0x10]    ; Load a
mov rcx, [rbp+0x18]    ; Load b
test rax, 0x1          ; Check if Smi (small integer)
jz deopt               ; Deoptimize if not
test rcx, 0x1
jz deopt
sar rax, 1             ; Untag Smi
sar rcx, 1
add rax, rcx           ; Integer add
jo deopt               ; Overflow check
sal rax, 1             ; Tag result
ret
```

**Deoptimization**:

If assumptions fail (e.g., receives float instead of int), **bailout** to interpreter:

```cpp
void Deoptimizer::DeoptimizeFunction(JSFunction* function) {
  // 1. Reconstruct interpreter state from optimized state
  FrameDescription* output_frame =
      CreateInputFrame(optimized_code);

  // 2. Restore interpreter registers/stack
  TranslateOptimizedFrame(output_frame, bytecode_offset);

  // 3. Resume in interpreter
  SetProgramCounter(output_frame, bytecode_offset);
}
```

### 4.4 Memory Management: Garbage Collection

**Heap Layout**:
```
┌──────────────────────────────────────┐
│         New Space (Semi-space)       │
│  ┌────────────┬─────────────┐        │
│  │  From Space │  To Space  │        │  ← Young gen (Scavenger)
│  └────────────┴─────────────┘        │
├──────────────────────────────────────┤
│           Old Space                  │  ← Tenured objects
│  (Mark-Compact)                      │
├──────────────────────────────────────┤
│           Large Object Space         │  ← >256KB objects
└──────────────────────────────────────┘
```

**Scavenger (Young Gen GC)** (`heap/scavenger.cc`):

Cheney's semi-space algorithm:
```cpp
void Scavenger::ScavengeObject(HeapObject* object) {
  // Copy live objects from From-space → To-space
  if (IsInFromSpace(object)) {
    Address new_address = AllocateInToSpace(object->Size());
    MemCopy(new_address, object->address(), object->Size());

    // Leave forwarding pointer
    object->set_map_word(MapWord::FromForwardingAddress(new_address));

    // Update all pointers
    UpdatePointersToNewAddress(object, new_address);
  }
}
```

**Mark-Compact (Old Gen GC)** (`heap/mark-compact.cc`):

Tri-color marking:
```cpp
void MarkCompactCollector::MarkLiveObjects() {
  // White = unvisited, Gray = marked but unscanned, Black = done

  // 1. Mark roots (stack, globals)
  MarkRoots();

  // 2. Drain marking worklist
  while (!marking_worklist_.IsEmpty()) {
    HeapObject* object = marking_worklist_.Pop();
    MarkingVisitor visitor;
    object->Iterate(&visitor);  // Mark all referenced objects
    SetBlack(object);
  }

  // 3. Sweep: reclaim white objects
  SweepSpaces();
}
```

**Write Barriers** (for incremental GC):

When old object points to new object:
```cpp
void Heap::WriteBarrier(HeapObject* host, Object* value) {
  if (InNewSpace(value) && !InNewSpace(host)) {
    // Record in remembered set for next young gen GC
    RememberedSet::Insert(host, value);
  }
}
```

---

## V. Event Loop & Asynchronous Execution

### 5.1 The Main Thread Event Loop

**Architecture**:
```
┌─────────────────────────────────────────┐
│           Task Queue                    │
│  ┌────┬────┬────┬────┬────┐            │
│  │T1  │T2  │T3  │T4  │... │            │
│  └────┴────┴────┴────┴────┘            │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│       Microtask Queue                   │
│  ┌────┬────┬────┐                       │
│  │MT1 │MT2 │... │  (Promises, MutationOb)
│  └────┴────┴────┘                       │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│      Rendering Pipeline                 │
│   rAF → Style → Layout → Paint          │
└─────────────────────────────────────────┘
```

**Implementation** (`third_party/blink/renderer/platform/scheduler/main_thread/main_thread_scheduler_impl.cc`):

```cpp
void MainThreadSchedulerImpl::ExecuteTaskQueue() {
  while (true) {
    // 1. Run one macrotask
    Task* task = task_queue_.TakeTask();
    if (task) {
      task->Run();
    }

    // 2. Run ALL microtasks
    while (!microtask_queue_.IsEmpty()) {
      Microtask* microtask = microtask_queue_.TakeTask();
      microtask->Run();
    }

    // 3. Rendering opportunity
    if (ShouldRender()) {
      RunAnimationFrameCallbacks();
      UpdateRendering();
    }

    // 4. Idle tasks (requestIdleCallback)
    if (HasIdleTime()) {
      RunIdleTasks();
    }
  }
}
```

### 5.2 Promises & Microtasks

When you create a promise:
```javascript
Promise.resolve(42).then(x => console.log(x));
console.log('sync');
// Output: "sync" then "42"
```

**V8 Integration** (`builtins/builtins-promise.cc`):

```cpp
BUILTIN(PromisePrototypeThen) {
  // 'this' is the promise object
  JSPromise* promise = JSPromise::cast(receiver);

  // Create reaction (callback + result promise)
  PromiseReaction* reaction =
      factory()->NewPromiseReaction(on_fulfilled, on_rejected);

  if (promise->status() == PromiseStatus::kFulfilled) {
    // Already resolved - queue microtask immediately
    isolate->EnqueueMicrotask(
        factory()->NewPromiseResolveThenableJobTask(
            reaction, promise->result()));
  } else {
    // Pending - attach reaction
    promise->reactions()->Add(reaction);
  }
}
```

### 5.3 Task Priorities

**Scheduler Task Types** (ordered by priority):

1. **Immediate**: User input (click, keypress)
2. **High**: Parsing, resource loading
3. **Default**: setTimeout, postMessage
4. **Low**: requestIdleCallback

```cpp
// scheduler/task_queue_impl.cc
void TaskQueueImpl::EnqueueTask(Task* task, TaskPriority priority) {
  switch (priority) {
    case TaskPriority::kImmediate:
      immediate_queue_.push(task);
      break;
    case TaskPriority::kHigh:
      high_priority_queue_.push(task);
      break;
    case TaskPriority::kDefault:
      normal_queue_.push(task);
      break;
  }
}
```

---

## VI. Storage Mechanisms

### 6.1 IndexedDB

**Architecture**: Embedded database (LevelDB in Chromium, LMDB-like in Firefox)

```cpp
// content/browser/indexed_db/indexed_db_database.cc
void IndexedDBDatabase::Put(Transaction* transaction,
                             int64_t object_store_id,
                             const IndexedDBValue& value,
                             const IndexedDBKey& key) {
  // 1. Serialize value
  std::string serialized_value;
  V8ScriptValueSerializer serializer(value);
  serializer.Serialize(&serialized_value);

  // 2. Write to LevelDB
  leveldb::WriteBatch batch;
  std::string db_key = EncodeKey(object_store_id, key);
  batch.Put(db_key, serialized_value);

  // 3. Update indexes
  for (IndexMetadata& index : object_store->indexes) {
    IndexedDBKey index_key = ExtractIndexKey(value, index);
    batch.Put(EncodeIndexKey(index.id, index_key), db_key);
  }

  backing_store_->CommitTransaction(batch);
}
```

**On-Disk Format**:
```
~/.config/chromium/Default/IndexedDB/
  ├─ https_example.com_0.indexeddb.leveldb/
  │   ├─ 000003.log         (write-ahead log)
  │   ├─ CURRENT            (current manifest)
  │   ├─ MANIFEST-000002
  │   └─ *.ldb/*.sst        (sorted string tables)
```

### 6.2 Cache API (Service Worker)

**Cache Storage** (`content/browser/cache_storage/cache_storage_manager.cc`):

```cpp
void CacheStorage::Match(const CacheQueryOptions& options,
                          MatchCallback callback) {
  // 1. Iterate caches in order
  for (Cache* cache : caches_) {
    // 2. Check if request matches
    if (cache->Match(request, options)) {
      // 3. Read response from disk
      CacheStorageCache::ReadResponse(cache_entry, callback);
      return;
    }
  }

  // No match
  std::move(callback).Run(CacheStorageError::kErrorNotFound, nullptr);
}
```

**Storage Layout**:
```
CacheStorage/
  ├─ index.txt              (cache names)
  ├─ cache_0/
  │   ├─ index              (LevelDB index)
  │   └─ blobs/
  │       └─ response_body_123.bin
```

---

## VII. Security Model

### 7.1 Same-Origin Policy

**Origin Definition**: `(scheme, host, port)`

```cpp
// url/origin.cc
bool Origin::IsSameOriginWith(const Origin& other) const {
  return scheme_ == other.scheme_ &&
         host_ == other.host_ &&
         port_ == other.port_;
}
```

**Enforcement Points**:

1. **DOM Access**:
```cpp
// bindings/core/v8/v8_window.cc
void V8Window::ParentAttributeGetterCallback() {
  if (!BindingSecurity::ShouldAllowAccessTo(
          current_world, window, exception_state)) {
    // Cross-origin: return opaque WindowProxy
    return opaqueWindow;
  }
  return window->parent();
}
```

2. **XMLHttpRequest**:
```cpp
// xhr/xml_http_request.cc
void XMLHttpRequest::Send() {
  if (!GetExecutionContext()->GetSecurityOrigin()->
          CanRequest(url_)) {
    // CORS preflight required
    SendCORSPreflightRequest();
  }
}
```

### 7.2 Content Security Policy

**Parser** (`csp/content_security_policy.cc`):

```cpp
void ContentSecurityPolicy::ParseDirective(const String& directive) {
  // "script-src 'self' https://cdn.example.com"
  Vector<String> parts;
  directive.Split(' ', parts);

  DirectiveType type = GetDirectiveType(parts[0]);  // script-src

  for (size_t i = 1; i < parts.size(); ++i) {
    if (parts[i] == "'self'") {
      directive_sources.push_back(
          CSPSource(GetSecurityOrigin()));
    } else {
      directive_sources.push_back(
          CSPSource::Parse(parts[i]));
    }
  }
}
```

**Enforcement**:
```cpp
bool ContentSecurityPolicy::AllowScriptFromSource(const KURL& url) {
  for (CSPSource& source : script_src_sources_) {
    if (source.Matches(url))
      return true;
  }

  // Blocked - report violation
  ReportViolation("script-src", url);
  return false;
}
```

### 7.3 Site Isolation

**Process Assignment** (`site_instance.cc`):

```cpp
SiteInstance* SiteInstanceImpl::GetSiteInstanceForURL(const KURL& url) {
  // Extract site (scheme + eTLD+1)
  SiteInfo site_info = SiteInfo::Create(url);

  // Find or create dedicated process
  RenderProcessHost* process =
      RenderProcessHostImpl::GetProcessHostForSite(site_info);

  return new SiteInstanceImpl(process, site_info);
}
```

**Cross-Origin Read Blocking (CORB)**:

```cpp
// services/network/cross_origin_read_blocking.cc
bool CrossOriginReadBlocking::ShouldBlockResponse(
    const network::ResourceRequest& request,
    const network::ResourceResponseHead& response) {
  // Block if:
  // 1. Cross-origin
  if (request.request_initiator->IsSameOriginWith(response.url))
    return false;

  // 2. MIME type is sensitive (HTML, JSON, XML)
  if (response.mime_type == "text/html" ||
      response.mime_type == "application/json") {

    // 3. No CORS headers
    if (!response.headers->HasHeader("Access-Control-Allow-Origin")) {
      return true;  // BLOCK
    }
  }

  return false;
}
```

---

## VIII. Performance Optimizations

### 8.1 Preload Scanner

While parser is blocked on `<script>`:

```cpp
// html/parser/html_preload_scanner.cc
void HTMLPreloadScanner::Scan() {
  // Lookahead tokenize HTML without building DOM
  while (tokenizer_.NextToken(token)) {
    if (token.GetType() == HTMLToken::kStartTag) {
      if (token.GetName() == html_names::kLinkTag) {
        String rel = token.GetAttribute(html_names::kRelAttr);
        if (rel == "stylesheet" || rel == "preload") {
          KURL url = token.GetAttribute(html_names::kHrefAttr);

          // Emit early resource hint
          resource_preloader_->Preload(PreloadRequest(url, kCSSResource));
        }
      }
    }
  }
}
```

### 8.2 Paint Invalidation

Minimize repaints:

```cpp
// paint/paint_invalidator.cc
void PaintInvalidator::InvalidatePaint(const LayoutObject& object) {
  // 1. Compute what changed
  PaintInvalidationReason reason =
      ComputePaintInvalidationReason(object);

  if (reason == PaintInvalidationReason::kNone)
    return;  // Early out

  // 2. Invalidate only affected rect
  if (reason == PaintInvalidationReason::kBounds) {
    LayoutRect old_bounds = old_paint_offset_;
    LayoutRect new_bounds = object.VisualOverflowRect();
    LayoutRect damage = UnionRect(old_bounds, new_bounds);

    SetPaintAsDirty(damage);
  }
}
```

### 8.3 Resource Prioritization

**Fetch Priority** (`loader/resource_fetcher.cc`):

```cpp
ResourceLoadPriority ComputeLoadPriority(Resource* resource) {
  if (resource->GetType() == ResourceType::kMainResource)
    return ResourceLoadPriority::kVeryHigh;

  if (resource->GetType() == ResourceType::kCSSStyleSheet) {
    if (IsRenderBlocking(resource))
      return ResourceLoadPriority::kVeryHigh;
    return ResourceLoadPriority::kHigh;
  }

  if (resource->GetType() == ResourceType::kScript) {
    if (resource->IsAsync())
      return ResourceLoadPriority::kLow;
    return ResourceLoadPriority::kHigh;
  }

  // Images: based on viewport position
  if (resource->GetType() == ResourceType::kImage) {
    return IsInViewport(resource) ?
        ResourceLoadPriority::kMedium : ResourceLoadPriority::kLow;
  }
}
```

---

## IX. DevTools Protocol

**Remote Debugging** (`core/inspector/inspector_session.cc`):

```cpp
void InspectorSession::DispatchProtocolMessage(const String& message) {
  // Parse CDP (Chrome DevTools Protocol) JSON
  // {"id": 1, "method": "Runtime.evaluate", "params": {"expression": "1+1"}}

  std::unique_ptr<protocol::DictionaryValue> command =
      protocol::StringUtil::parseJSON(message);

  String method = command->getString("method");

  // Route to domain handler
  if (method.StartsWith("Runtime.")) {
    runtime_agent_->Dispatch(command);
  } else if (method.StartsWith("Debugger.")) {
    debugger_agent_->Dispatch(command);
  }
}
```

**Runtime.evaluate**:
```cpp
void InspectorRuntimeAgent::Evaluate(const String& expression) {
  // Execute JavaScript in page context
  v8::Local<v8::Value> result =
      V8ScriptRunner::CompileAndRunInternalScript(
          v8::String::NewFromUtf8(isolate_, expression));

  // Serialize result back to DevTools
  std::unique_ptr<protocol::Runtime::RemoteObject> remote_object =
      CreateRemoteObject(result);

  callback->sendSuccess(std::move(remote_object));
}
```

---

## X. Advanced Topics

### 10.1 WebAssembly Execution

**Instantiation** (`wasm/wasm-module-builder.cc`):

```cpp
MaybeHandle<WasmInstanceObject> InstantiateModule(
    Isolate* isolate,
    Handle<WasmModuleObject> module_object) {
  // 1. Allocate WebAssembly memory
  Handle<WasmMemoryObject> memory =
      WasmMemoryObject::New(isolate, initial_pages, maximum_pages);

  // 2. Compile to machine code (or use cached)
  wasm::NativeModule* native_module =
      module_object->native_module();

  for (uint32_t func_index = 0; func_index < num_functions; ++func_index) {
    WasmCode* code = native_module->GetCode(func_index);
    if (!code) {
      // TurboFan compile WASM → machine code
      code = CompileWasmFunction(isolate, func_index);
      native_module->PublishCode(code);
    }
  }

  // 3. Link imports
  LinkImports(instance, import_object);

  return instance;
}
```

**Execution**: Direct machine code call from JavaScript:

```cpp
// When calling WASM function from JS:
RUNTIME_FUNCTION(Runtime_WasmFunctionCall) {
  WasmInstanceObject* instance = ...;
  uint32_t function_index = ...;

  // Get compiled machine code
  wasm::WasmCode* code =
      instance->module_object()->native_module()->GetCode(function_index);

  // Direct call (no interpreter)
  Address entry_point = code->instruction_start();
  using WasmFunctionPtr = int32_t (*)(Address);
  WasmFunctionPtr wasm_func = reinterpret_cast<WasmFunctionPtr>(entry_point);

  int32_t result = wasm_func(instance_address);
  return Smi::FromInt(result);
}
```

### 10.2 Service Worker Lifecycle

**Registration** (`service_worker/service_worker_container.cc`):

```cpp
void ServiceWorkerContainer::Register(const KURL& script_url) {
  // 1. Start download & parse
  ServiceWorkerContextCore* context = provider_->context();

  context->RegisterServiceWorker(
      script_url,
      base::BindOnce(&OnRegistrationComplete));
}

void OnRegistrationComplete(ServiceWorkerRegistration* registration) {
  // 2. Install event
  registration->installing_version()->RunAfterStartWorker(
      base::BindOnce(&DispatchInstallEvent));
}

void DispatchInstallEvent(ServiceWorkerVersion* version) {
  // 3. Fire 'install' event in worker context
  version->StartWorker(ServiceWorkerMetrics::EventType::INSTALL,
                       base::BindOnce(&OnInstallEventFinished));
}
```

**Fetch Interception** (`service_worker_main_resource_loader.cc`):

```cpp
void ServiceWorkerMainResourceLoader::StartRequest() {
  // 1. Check for active service worker
  ServiceWorkerVersion* active_version =
      context_->GetLiveVersion(registration_->active_version_id());

  if (active_version) {
    // 2. Dispatch fetch event to worker
    active_version->DispatchFetchEvent(
        fetch_event_id_,
        request_,
        base::BindOnce(&OnFetchEventFinished));
  } else {
    // 3. Fallback to network
    StartNetworkRequest();
  }
}
```

---

## XI. Debugging & Profiling Internals

### 11.1 Chrome Tracing

**Trace Events** (tracing/trace_event.h):

```cpp
// Record performance trace
TRACE_EVENT0("blink.rendering", "PaintLayerCompositor::UpdateIfNeeded");
TRACE_EVENT_BEGIN0("v8", "V8.Optimize");
// ... optimization code ...
TRACE_EVENT_END0("v8", "V8.Optimize");

// With arguments
TRACE_EVENT2("network", "ResourceFetch",
             "url", url.GetString(),
             "priority", priority);
```

Outputs JSON to `chrome://tracing`:
```json
{
  "name": "PaintLayerCompositor::UpdateIfNeeded",
  "cat": "blink.rendering",
  "ph": "X",  // Complete event
  "ts": 1234567890,
  "dur": 1500,
  "pid": 12345,
  "tid": 1
}
```

### 11.2 Memory Profiling

**Heap Snapshot** (`profiler/heap-snapshot-generator.cc`):

```cpp
void HeapSnapshotGenerator::GenerateSnapshot() {
  // 1. Enumerate all HeapObjects
  HeapIterator iterator(heap_);
  for (HeapObject* obj = iterator.Next();
       obj != nullptr;
       obj = iterator.Next()) {

    // 2. Create snapshot node
    HeapEntry* entry = CreateEntry(obj);

    // 3. Record edges (references)
    obj->Iterate([&](Object* field) {
      HeapEntry* field_entry = CreateEntry(field);
      entry->AddEdge(field_entry);
    });
  }

  // 4. Compute retained sizes (DFS from GC roots)
  CalculateRetainedSizes();
}
```

Output: Directed graph for DevTools Memory panel

---

## XII. Emerging Architectures

### 12.1 Origin Isolation

**Speculation**: Future process-per-origin:

```cpp
// Hypothetical
bool ShouldIsolateOrigin(const url::Origin& origin) {
  // Opt-in via COOP/COEP headers
  if (origin.HasCrossOriginOpenerPolicy() &&
      origin.HasCrossOriginEmbedderPolicy()) {
    return true;
  }

  // Or automatic isolation for sensitive APIs
  if (origin.UsesSharedArrayBuffer() || origin.AccessesDeviceAPIs()) {
    return true;
  }

  return false;
}
```

### 12.2 Shared Element Transitions

**View Transition API** (`view_transitions/view_transition.cc`):

```cpp
void ViewTransition::Start() {
  // 1. Capture old state
  for (Element* el : transitioning_elements_) {
    CapturedElement snapshot;
    snapshot.bounds = el->BoundingBox();
    snapshot.pixels = RasterizeElement(el);
    old_state_[el->GetIdAttribute()] = snapshot;
  }

  // 2. Apply DOM changes
  update_dom_callback_.Run();

  // 3. Capture new state
  for (Element* el : transitioning_elements_) {
    new_state_[el->GetIdAttribute()] = CaptureElement(el);
  }

  // 4. Animate between states
  CreateTransitionAnimation(old_state_, new_state_);
}
```

---

## Conclusion: The Browser as Operating System

A modern browser is:

- **Network Stack**: HTTP/3, TLS 1.3, DNS-over-HTTPS, QUIC congestion control
- **Parser Combinator**: HTML/CSS/JS parsers with error recovery
- **Virtual Machine**: JIT compiler (TurboFan), GC (Orinoco), bytecode interpreter
- **Graphics Engine**: GPU-accelerated compositor, Skia/Core Graphics, display lists
- **Security Kernel**: Sandboxing (seccomp-bpf), site isolation, CSP enforcement
- **Distributed System**: Multi-process with IPC (Mojo), shared memory buffers
- **Database**: IndexedDB (LevelDB), Cache API, ServiceWorker storage
- **Scheduler**: Priority queues, event loop, microtasks, rendering pipeline

**Performance Critical Paths**:
1. **Input → Paint**: 16ms budget for 60 FPS
2. **Parse → Execute**: Streaming parsers + preload scanner
3. **Network → Cache**: HTTP cache, Service Worker intercept, prefetch

**Source Code Reference Points**:
- Chromium: `chromium/src/` (~25M LOC)
  - Rendering: `third_party/blink/`
  - V8: `v8/src/`
  - Network: `net/`, `services/network/`
- Firefox: `mozilla-central/`
  - Gecko: `layout/`, `dom/`
  - SpiderMonkey: `js/src/`

The browser doesn't just *run* web applications—it **is** the platform. Every pixel you see traversed this entire stack, from TCP handshake to GPU rasterization, orchestrated by thousands of cooperating threads across isolated processes.

That's how the web works. From syscalls to shaders.
