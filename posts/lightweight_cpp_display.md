# Drawing a Face on Almost Nothing: Building Mochiro's Lightweight Display Driver

Mochiro is my little robot companion. I wanted it to have an expressive face — eyes that blink, squint, and go wide — running on a small single-board computer, with parts eventually pushed down onto even tinier microcontrollers. Animating a face on a device with the memory of a 1990s desktop is the whole story of this driver.

What follows are the C and C++ tricks that kept it fast and tiny. None are exotic; the craft is in *not doing the wasteful thing* in the places wasteful things hide.

## The one fact everything flows from

A face at 60 fps needs a fresh image sixty times a second, and the naive model — draw a full picture, hand it over, repeat — is fatal here: a single 480×320 frame in full color is ~460 KB, and the smallest chip I target has 264 KB of RAM *total*. The frame is too big to treat casually, so the whole design is built around never materializing it more than it must.

The driver is a C++ module (exposed to Python via pybind11) that renders with OpenGL ES 2.0 on SDL2 — the comfortable, Pi-class end of the spectrum. The interesting part is keeping that pipeline cheap, then asking what survives when you strip GL away for a bare microcontroller.

## First, the conventional stuff

Of course there are the more conventional optimizations you'd expect on constrained hardware, and Mochiro leans on all of them: the face is a small animated **mesh** rather than a folder of bitmaps, so memory never grows with the number of expressions and the GPU rasterizes it for free; the per-frame vertex interpolation writes into a buffer allocated once at startup, so nothing touches the heap inside the render loop; only the vertices that actually changed are streamed to a pre-allocated VBO with `glBufferSubData`, and the whole face draws in a single `glDrawElements` call; the shaders stay flat-colored and `mediump` to spare the little GPU's fragment throughput; and the binary is built with `-Os`, link-time optimization, and a `-mcpu` tuned to the exact chip. Those are table stakes. The decisions that actually *shaped* this driver — the ones worth writing down — were the less obvious ones.

## Keep the threads from waiting on each other

Rendering runs on its own thread, separate from the code that decides *what* to show. How does one tell the other "blink now" without the two tripping over each other?

A mutex works, but a lock means one thread can stall waiting on the other, and a stalled render thread is a visible stutter. For simple signals, atomics are enough: a change of expression is just an integer written by one thread and read by the other. Nobody blocks, and the worst case is the render thread noticing one frame late — which no eye catches.

```cpp
// Written by the brain thread, read by the render thread. No mutex needed.
static std::atomic<int>   g_target_expression{EXPR_NEUTRAL};
static std::atomic<bool>  g_blink_requested{false};

// In the render loop:
int want = g_target_expression.load(std::memory_order_relaxed);
if (want != g_current_expression) {
    begin_transition_to(want);          // sets up the 'b' pose for blend_pose
}
if (g_blink_requested.exchange(false)) {
    trigger_blink();
}
```

The rule that keeps this honest: the threads hand each other small atomic facts, never a complex structure they both edit. When the state *is* complex — a whole new face geometry loaded from disk — I build it off to the side and flip a single atomic pointer, so the render thread picks up the finished object whole, never a half-built one.

## Stream the face out cheaply, off the critical path

Mochiro streams its face as MJPEG so I can watch it from a laptop while debugging. Done on the render thread it wrecks the frame rate, because both of its steps are expensive: reading pixels back from the GPU, and JPEG-compressing them.

`glReadPixels` is the villain — it forces the GPU to finish everything before handing pixels over, stalling the pipeline. So capture is throttled to what a client actually needs, and compression runs on a *separate* encoder thread. The render thread just drops fresh pixels into a handoff buffer and gets back to drawing.

That handoff is a triple-buffer so grabber and encoder never block: three slots mean there's always a free one for each side.

```cpp
// Three slots: one being written, one most-recently-published, one being read.
static uint8_t g_slots[3][kFrameBytes];
static std::atomic<int> g_latest{-1};   // index of newest complete frame
static int g_writing = 0;               // grabber's current scratch slot

// Grabber side (throttled, not every frame):
void publish_frame() {
    glReadPixels(0, 0, W, H, GL_RGB, GL_UNSIGNED_BYTE, g_slots[g_writing]);
    int published = g_writing;
    g_writing = (g_writing + 1) % 3;
    if (g_writing == g_latest.load()) g_writing = (g_writing + 1) % 3;
    g_latest.store(published, std::memory_order_release);
}

// Encoder thread side:
int idx = g_latest.exchange(-1, std::memory_order_acquire);
if (idx >= 0) encode_and_send_jpeg(g_slots[idx]);
```

The HTTP server fronting it is hand-rolled on raw sockets — a full framework would be hundreds of KB to do what MJPEG barely needs: write a header, then loop emitting a boundary, a length, and a JPEG blob. Those few lines keep the binary small and the dependency list empty.

## Going smaller: what survives when you take the GPU away

Everything above assumes a board big enough for real OpenGL ES. A microcontroller like the Pico's RP2040 has no GPU, no Linux, no SDL, and that 264 KB of RAM — the full-color framebuffer doesn't even fit. So which tricks were *principles*, and which were just GPU conveniences?

Geometry-not-pixels carries over and matters more: with no GPU to rasterize, a cartoon face's large flat regions can be drawn as filled shapes straight into the framebuffer.

That framebuffer shrinks by switching to 16-bit **RGB565**, halving both its size and the bytes shipped to the panel. On the smallest devices you skip a full framebuffer entirely, driving a small SPI display (ST7789/ILI9341) and re-sending only the rectangles that changed — blink, and only the two rectangles around the eyes go out, clocked over SPI by **DMA** so the lone CPU core can keep thinking.

The principle that becomes critical is dropping floating-point. The main board has an FPU; a Cortex-M0+ doesn't, so every `float` multiply is a slow software routine. The interpolation gets rewritten in fixed-point — a value stored as an integer scaled by a fixed factor, a "multiply" done as an integer multiply and shift:

```c
// Fixed-point: 16.16 format. A "1.0" is (1 << 16). Integer math only —
// no FPU required, so this runs fast on an M0+ where floats would crawl.
typedef int32_t fix16_t;
#define FIX_ONE (1 << 16)

static inline fix16_t fix_lerp(fix16_t a, fix16_t b, fix16_t t) {
    // a + (b - a) * t, keeping everything in integer land
    return a + (fix16_t)(((int64_t)(b - a) * t) >> 16);
}
```

Same blend as before, wearing integer clothes — the shape didn't change, only the arithmetic.

## The C++ that earns its keep

A surprising amount of the speed came from the language itself — making it do work before the program runs, or shedding the overhead its abstractions usually smuggle in. These cut across both targets.

### Move the work into the compiler

A natural blink follows a *smoothstep* easing curve, not a straight line. Evaluating it per frame is harmless on the big board and painful on a chip with no FPU — so I precompute it as a 256-entry table, *in the compiler*, with `constexpr`:

```cpp
// Computed by the compiler, not the device. Because it's constexpr with
// static storage, it lands in .rodata (flash/ROM) and costs ZERO RAM.
struct EaseTable { int32_t v[256]; };   // 16.16 fixed-point entries

constexpr EaseTable make_ease_table() {
    EaseTable t{};
    for (int i = 0; i < 256; ++i) {
        double x = i / 255.0;
        double s = x * x * (3.0 - 2.0 * x);   // smoothstep, evaluated at compile time
        t.v[i] = static_cast<int32_t>(s * 65536.0);
    }
    return t;
}

constexpr EaseTable kEase = make_ease_table();   // exists before main() runs
```

The win is double: a lookup replaces a polynomial, and because `kEase` is `constexpr` it lands in read-only memory (flash) rather than RAM — found money where RAM is scarcest. (C++20's `consteval` forces the compile-time evaluation if you want the guarantee.)

### Pay for polymorphism only when you actually use it

The platform layer — SDL window vs. bare framebuffer — invites a `virtual` base class. But runtime polymorphism isn't free: a vtable pointer in every object (RAM), and an indirect call the compiler can't inline.

A firmware image only ever runs on one platform, known at compile time, so CRTP — a base templated on its own derived type — gives the same clean interface with every call resolved and inlined:

```cpp
// Compile-time polymorphism. No vtable, no per-object vptr, and every call
// inlines straight through to the implementation.
template <class Impl>
struct Platform {
    void swap_buffers() { static_cast<Impl*>(this)->swap_buffers_impl(); }
    bool poll_events()  { return static_cast<Impl*>(this)->poll_events_impl(); }
};

struct PlatformSDL : Platform<PlatformSDL> {
    void swap_buffers_impl() { SDL_GL_SwapWindow(window_); }
    bool poll_events_impl()  { /* drain SDL events */ return true; }
    SDL_Window* window_;
};
```

The trade is losing runtime platform selection, but you pick the platform by which template you instantiate — exactly when that decision is actually made.

### Own your memory by hand

Where does that off-thread geometry live if the heap is banned from the hot path? An **arena**: a fixed block I own, where allocating is a pointer bump and "freeing" is resetting the whole block at once — no fragmentation, deterministic timing.

```cpp
#include <new>        // placement new
#include <utility>    // std::forward

// A bump allocator over a static block. Allocation is a pointer increment;
// there's no per-object free, so no fragmentation and no malloc latency.
class Arena {
public:
    Arena(std::byte* base, std::size_t cap) : base_(base), cap_(cap) {}

    template <class T, class... Args>
    T* make(Args&&... args) {
        std::size_t off = align_up(used_, alignof(T));
        if (off + sizeof(T) > cap_) return nullptr;            // out of room
        T* p = new (base_ + off) T(std::forward<Args>(args)...); // placement new
        used_ = off + sizeof(T);
        return p;
    }
    void reset() { used_ = 0; }   // "free everything" in O(1)

private:
    static std::size_t align_up(std::size_t n, std::size_t a) {
        return (n + a - 1) & ~(a - 1);
    }
    std::byte* base_;
    std::size_t cap_, used_ = 0;
};

// Backed by static storage — the heap is never involved.
alignas(std::max_align_t) static std::byte g_pool[32 * 1024];
static Arena g_geometry_arena{g_pool, sizeof(g_pool)};
```

The key piece is *placement new* — `new (ptr) T(...)` constructs an object in raw bytes I already own, so the arena holds real objects, not just a slab. The catch it hands back is destruction: placement-new'd objects aren't destroyed automatically, so anything owning a resource needs a manual `~T()` before `reset()`. Mochiro's geometry is trivially destructible, so the reset alone suffices.

### Tell the compiler what your pointers won't do

Take the loop that interpolates vertices between two poses each frame. By default the compiler must assume the output array might overlap the inputs (*aliasing*), so it reloads defensively instead of vectorizing. `__restrict` promises no overlap:

```cpp
// '__restrict' promises these arrays don't alias, freeing the compiler to
// vectorize (e.g. ARM NEON) instead of reloading defensively each iteration.
static void blend_pose(const Vec2* __restrict a,
                       const Vec2* __restrict b,
                       Vec2*       __restrict out,
                       float t, int n) {
    for (int i = 0; i < n; ++i) {
        out[i].x = a[i].x + (b[i].x - a[i].x) * t;
        out[i].y = a[i].y + (b[i].y - a[i].y) * t;
    }
}
```

With an `alignas(16)` on the buffers so the vectorized loads stay aligned, it's a one-word annotation handing the optimizer a fact it can't deduce — and on a loop run sixty times a second, that's real.

### A type that costs you nothing

The earlier `fix16_t` typedef worked but read badly: `fix_lerp(a, b, t)` instead of `a + (b - a) * t`. Wrapping the integer in a small type with inlinable operators gives the readable math back for free:

```cpp
template <int FracBits>
struct Fixed {
    int32_t raw;
    static constexpr int32_t kOne = 1 << FracBits;

    constexpr Fixed() : raw(0) {}
    constexpr explicit Fixed(float f) : raw(int32_t(f * kOne)) {}  // compile-time constants!

    constexpr Fixed operator+(Fixed o) const { return from_raw(raw + o.raw); }
    constexpr Fixed operator-(Fixed o) const { return from_raw(raw - o.raw); }
    constexpr Fixed operator*(Fixed o) const {
        return from_raw(int32_t((int64_t(raw) * o.raw) >> FracBits));
    }
    static constexpr Fixed from_raw(int32_t r) { Fixed f{}; f.raw = r; return f; }
};
using fix16 = Fixed<16>;
```

"Zero-cost abstraction," literally: each operator is trivial, so the compiler unwraps the expression back into the exact integer multiply-and-shift I'd written by hand — and the `constexpr` constructor builds fixed-point constants at compile time, looping back to the first idea here.

### Turn off what you're not using

The driver signals failure with return codes and never asks an object its type at runtime, so I build `-fno-exceptions -fno-rtti`. Exceptions drag in unwinding tables; RTTI emits type records per polymorphic class. Dropping both trims real bytes off a flash-tight binary — it's just an all-or-nothing, whole-program choice.

## What it came down to

The one habit that mattered more than any trick: profile before believing anything — the slow part is almost never where intuition points. I'd fretted over the rendering, but the real cost centers were the GPU readback and the JPEG encode.

And if I had to compress the whole experience into one sentence, it would be this: **the fast path is the one where almost nothing happens.** No allocation, no copying, no locking, no recomputing what didn't change, no precision you don't need. Mochiro's face feels alive not because the driver does something clever sixty times a second, but because it has been carefully taught to do almost nothing at all — and to do that nothing very, very quickly.