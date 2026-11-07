---
name: ios-logos-tweak
description: Best practices for iOS jailbreak tweaks using Theos and Logos (%hook, %hookf, %new, %orig, %init). Prefer Logos and Substrate helpers (MSHookIvar) over manual hook/ivar runtime APIs. Covers firmware checks (IS_IOS_OR_NEWER vs @available vs respondsToSelector) and bracket messaging. Use when writing or reviewing .x/.xm tweak code, MobileSubstrate/libhooker hooks, swizzling, or Theos Tweak projects.
---

# iOS Logos Tweak Best Practices

## Core rule

Put hooking code in **Logos files** (`.x`, `.xm`, `.xi`) and express hooks with **Logos directives**. Let the Logos preprocessor emit `MSHookMessageEx`, `MSHookFunction`, `class_addMethod`, constructors, and class registration.

Do **not** hand-write Substrate hook boilerplate in tweak source unless Logos cannot express the case (rare; see [reference.md](reference.md)).

## File layout

**Default: `.x`.** Most tweaks use a single `Tweak.x` (Logos + Objective-C). Theos runs Logos on the file, then compiles the result as ObjC.

| Extension | Compiles as | When to use |
|-----------|-------------|-------------|
| **`.x`** | Objective-C | **Default.** `%hook`, `%hookf`, `%new`, `%orig`, `%property`, blocks, and normal Foundation/UIKit code. |
| **`.xm`** | Objective-C++ | Only when this **translation unit** needs C++ or Substrate’s C++ APIs (see below). |
| **`.xi` / `.xmi`** | Included fragments | Rare: Logos inside `#define` macros or `#include` chains; prefer splitting into separate `.x`/`.xm` files with `extern` instead ([Theos docs](https://theos.dev/docs/logos-file-extensions)). |

### Use `.xm` when this file contains

- **`MSHookIvar<T>`** or **`MSIvarHook`** (implemented as C++ templates in `<substrate.h>`).
- **C++** only valid in ObjC++: `std::`, `namespace`, templates, lambdas, C++ types in hook bodies.
- **`%orig`** forms that embed **block literals** in the hook implementation when your toolchain/setup requires ObjC++ for those expansions (uncommon; most hooks stay in `.x`).

### Stay on `.x` when

- Hooks are plain Objective-C (even with blocks, `@autoreleasepool`, etc.).
- You do not call `MSHookIvar` in that file.

If only one hook file needs ivar access, keep **`Tweak.x`** and add a small **`Something.xm`** (or move just those hooks to `.xm`) rather than renaming the whole project to `.xm` without reason.

Keep **interfaces and constants** in normal `.h` / `.m` when they are not hooks. Keep **all hooks** in Logos (`.x` / `.xm`, not raw `.m` with `MSHook*`).

Do not `#include <substrate.h>` in Logos files; Theos/Logos adds it in generated code.

## Objective-C methods: `%hook`

```logos
%hook YTPlayerViewController

- (void)layoutSubviews {
    %orig;
    // tweak logic
}

%end
```

- Match the **exact** selector signature (class vs instance `+`/`-`, types, argument names).
- Call the original with `%orig` or `%orig(arg1, arg2)`; never pass `self` or `_cmd`.
- If you use `%orig(...)`, pass **every** user argument (Logos fills `self`/`_cmd`).
- Always close with `%end`.

**Avoid:** `MSHookMessageEx`, `method_setImplementation`, `method_exchangeImplementations`, or IMP trampolines for methods Logos can hook.

## Objective-C messaging

Send messages with **`[receiver message]`** / **`[receiver foo:bar]`** syntax. Do not use `objc_msgSend`, `objc_msgSendSuper`, or similar runtime send helpers for calls you can express as normal Objective-C.

If a symbol is undeclared in the public SDK, add a **private header** or class/category `@interface` with the method, then call it with brackets. That keeps type checking and ARC rules; `objc_msgSend` needs manual casts, is easy to get wrong on arm64, and reads worse than idiomatic ObjC.

**Avoid:** `((void (*)(id, SEL, ...))objc_msgSend)(obj, @selector(foo), arg)` when `[obj foo:arg]` (after a proper declaration) works.

## C functions: `%hookf`

```logos
%hookf(BOOL, MGGetBoolAnswer, CFStringRef string) {
    if (CFEqual(string, CFSTR("SomeKey"))) return YES;
    return %orig;
}
```

- First argument is return type; second is the **symbol name**; rest are parameters (no implicit `self`/`_cmd`).
- For private symbols, resolve at init time with **`MSFindSymbol(MSGetImageByName(...), "_symbol")`**. Pass the **image that owns the symbol** as the first argument to `MSFindSymbol`; do not pass `NULL`.

```logos
%ctor {
    void *gestalt = MSGetImageByName("/usr/lib/libMobileGestalt.dylib");
    %init(MGGetBoolAnswer = MSFindSymbol(gestalt, "_MGGetBoolAnswer"));
}
```

- **`MSFindSymbol(NULL, name)`** searches every loaded image. That pattern is **discouraged**: some runtime modification / injection setups can crash or behave badly during global symbol lookup. Always **`MSGetImageByName("/path/to/Image.dylib")`** (framework binary path as Substrate expects) and pass that `MSImageRef` to `MSFindSymbol`.
- If lookup fails, guard before `%init` (skip the group or log) rather than passing a null pointer into `%init`.

**Avoid:** `MSHookFunction` + manual replacement when `%hookf` + `%init` suffices; **`MSFindSymbol(NULL, ...)`** for dynamic `%hookf` targets.

## New methods and storage: `%new` and `%property`

Add methods on an existing class inside `%hook`:

```logos
%hook MyTargetClass

%new(v@:@)
- (void)myTweak_didTap:(id)sender {
    // no %orig — method did not exist before
}

%property (nonatomic, strong) UIView *myTweak_overlay;

%end
```

- Use `%new` for any method that **does not exist** on the class (including helpers only your tweak calls).
- Always write **`%new(typeencoding)`** with the **exact** Objective-C method type encoding (e.g. `%new(v@:)` for `- (void)foo`, `%new(v@:@)` for `- (void)foo:(id)x`, `%new(@@:)` for `- (id)bar`). Bare `%new` without parentheses makes Logos **generate** an encoding from the declaration; that extra work is unnecessary when you already know the signature.
- Match the encoding to the method you declare; when unsure, compile a stub and read `@encode` / `method_getTypeEncoding`, or copy from a class dump for the same selector shape.
- Use `%property` for associated state on hooked classes instead of `objc_setAssociatedObject` in every hook (unless you need fine-grained control).

**Avoid:** `class_addMethod` + static C functions for tweak-only ObjC methods on hooked classes.

## Instance variables: `MSHookIvar`

In **tweak dylib code** linked against Cydia Substrate (Logos `.x`/`.xm`, or `.m`/`.xm` helpers in the tweak target), read and write the target app’s **existing** ivars with **`MSHookIvar`** from `<substrate.h>` (included automatically in Logos output). Do not reimplement access with `class_getInstanceVariable`, `ivar_getOffset`, manual `((char *)self + offset)`, `object_getIvar`, or `object_setIvar` unless Substrate is unavailable.

```logos
%hook SomeViewController

- (void)viewDidLoad {
    %orig;
    UIView *content = MSHookIvar<UIView *>(self, "_contentView");
    content.hidden = NO;
}

%end
```

Inside a hook body, the **`MSIvarHook(type, name)`** macro (Substrate) is equivalent to a typed reference for an ivar whose **name matches** the stringized token (e.g. `MSIvarHook(UIView *, _contentView)`).

- Use **`.xm`** for this file if it uses **`MSHookIvar`** / **`MSIvarHook`**; otherwise keep ivar-free hooks in **`.x`**.
- For **state your tweak adds**, prefer **`%property`** on the hooked class, not reading/writing foreign ivars you invent.
- In targets **without** Substrate (preference bundles, standalone tools, libraries not injected into processes), use the Objective-C runtime ivar APIs or public accessors; do not assume `MSHookIvar` is linked.

**Avoid:** offset hunting duplicated in every call site when the tweak already links Substrate.

## When hooks run: `%group`, `%ctor`, `%init`

- Ungrouped `%hook` / `%hookf` → default group; Logos emits a constructor that calls `%init` automatically.
- Use `%group` when hooks must be **conditional** or **delayed**:

```logos
#import <version.h>

%group iOS16
%hook SomeNewClass
- (void)foo { %orig; }
%end
%end

%ctor {
    if (IS_IOS_OR_NEWER(iOS_16_0)) %init(iOS16);
}
```

- Call `%init` or `%init(GroupName)` from `%ctor` (or another function) for grouped hooks; do not rely on auto-init for grouped code.
- **One `%init` per `%group` in the whole file.** Logos records initialization at preprocess time. A second `%init(ThatGroup)` anywhere (including the other branch of `if`/`else`) is a **compile error** (`re-%init of %group`). Branches do not make two inits “conditional” at runtime; both directives are still seen.

  **OK:** different groups, one init each:

  ```logos
  %ctor {
      if (IS_IOS_OR_NEWER(iOS_9_0)) {
          %init(iOS9);
      } else {
          %init(iOS8);
      }
  }
  ```

  **Not OK:** same group twice:

  ```logos
  %ctor {
      if (useLegacy) {
          %init(SharedHooks, Foo = objc_getClass("LegacyFoo"));
      } else {
          %init(SharedHooks, Foo = objc_getClass("ModernFoo"));  // re-%init error
      }
  }
  ```

  **Fix:** use separate `%group`s, or pick the `ClassName=expr` arguments once and call **`%init(SharedHooks, Foo = chosenClass)`** a single time after computing `chosenClass` in ordinary C/ObjC.

- Use `%init(ClassName = objc_getClass("Swift.Module.Class"))` for Swift/mangled or awkward class names.

## Firmware and API availability

Pick **one** primary strategy per code path. Mixing `IS_IOS_OR_NEWER`, `@available`, `%group`, and `respondsToSelector` on the same feature adds noise without extra safety.

### Hook groups and private behavior: `IS_IOS_OR_NEWER`

Default for **`%ctor` / `%init`**, private SpringBoard/app internals, and point-release or range checks. `#import <version.h>` (Theos `vendor/include`) and compare `kCFCoreFoundationVersionNumber`:

```objc
if (IS_IOS_OR_NEWER(iOS_16_0)) { ... }
if (IS_IOS_BETWEEN(iOS_14_0, iOS_14_8)) { ... }
```

`@available` tracks marketing OS version and SDK availability annotations. Those do not describe private classes. A high `IPHONEOS_DEPLOYMENT_VERSION` can also **compile away** `@available` checks.

If `version.h` has no constant yet, **add** `kCFCoreFoundationVersionNumber_iOS_X_Y` and keep using the macros. Do not switch the whole tweak to `@available` just because one new OS is missing from the table.

### Public SDK APIs: `@available`

Use `@available(iOS 16, *)` at **call sites** for APIs the SDK marks with availability (compiler warnings, weak linking). Do not use it as the gate for Logos `%group`s of private hooks.

```objc
if (@available(iOS 16, *)) {
    [view setPreferredFrameRateRange:...];
}
```

### When the deployment floor already includes the API

If the tweak only runs on iOS versions where the selector **always exists** (package minimum, or hooks live in a `%group` initialized only on that OS), **call it directly**. Do not wrap every use in `respondsToSelector:` / `instancesRespondToSelector:`.

Version gating at init (`%group` + `%init` with `IS_IOS_OR_NEWER`) is enough; inside those hooks, treat the method as present.

### Optional selectors without a version table: `respondsToSelector:`

Use when:

- The tweak supports a **wide OS range** with **few** explicit version checks, and
- A selector appears only on some releases (mid-cycle add, private API drift, optional helpers), and
- You are calling across classes you do not hook (not choosing whether to install a Logos hook).

Prefer **class** checks (`+[SomeClass instancesRespondToSelector:]`) when the method is on the class, not per instance.

```objc
if ([obj respondsToSelector:@selector(newStyleLayout)]) {
    [obj newStyleLayout];
} else {
    [obj legacyLayout];
}
```

### What to avoid

- `@available` in `%ctor` to decide `%init` of private/internal hook groups; use `IS_IOS_OR_NEWER` / `IS_IOS_BETWEEN`.
- `respondsToSelector:` before every send when `%init(iOS16)` or the package minimum already guarantees the selector.
- Stacking `respondsToSelector:`, `@available`, and `IS_IOS_OR_NEWER` on the same branch unless they guard different risks.
- Using runtime checks instead of `%group` for **entire hook sets** that only apply on one OS.

## Other Logos tools (prefer over manual runtime)

| Need | Logos / Substrate |
|------|--------|
| Runtime class reference | `%c(ClassName)` or `%c(+ClassName)` for metaclass |
| Log hook calls | `%log` in a method body |
| Subclass at runtime | `%subclass` + `%property` + `%c()` to allocate |
| Target class private ivar | `MSHookIvar` / `MSIvarHook` in tweak code (not raw runtime offset math) |
| Generator backend | `%config(generator=MobileSubstrate)` (default), `libhooker`, or `internal` |

Note: `%hookf` is **not** supported with `generator=internal`. Pick the generator to match the target jailbreak environment.

## Code organization

- One `%hook ClassName` block per class; multiple methods inside one block.
- Split large tweaks with `%group` in separate Logos files when using [hook splitting](https://theos.dev/docs/logos-hook-splitting) (`%init` from a shared constructor); use `.x` unless that file needs `.xm` for C++/`MSHookIvar`.
- Declare classes you hook in headers (`@interface`) or private headers; Logos does not replace the need for correct types at compile time.
- Use `logos_interface` / empty `@interface` in `.x` when you need typed calls to `%new` APIs from plain `.m` files.

## Filter plist (`.plist`)

Keep the tweak **filter plist** (e.g. `MyTweak.plist`) in the repo as **XML** (`plutil` “xml1”), not binary bplist.

- **Readability:** `Bundles`, `Classes`, and `Executables` filters are easy to review in git diffs.
- **Release builds:** `make package FINALPACKAGE=1` already **converts plists to binary** in the packaged `.deb`; you do not need to commit binary plists for production.

To convert an existing binary plist back to XML:

```bash
plutil -convert xml1 MyTweak.plist
```

Commit the XML source; let Theos optimize at package time.

## Common mistakes

1. **Hooking in `.m` with Substrate** while the rest of the project uses Logos — move hooks into **`Tweak.x`** (or `.xm` only if that file needs ObjC++).
2. **Missing `%end`** or nesting `%hook` inside a method (invalid; Logos errors).
3. **Calling `%orig` in a `%new` method** — unsupported; call super or another selector explicitly if needed.
4. **Forgetting `%init` for `%group`** — hooks never run.
5. **Two `%init`s for the same `%group`** (e.g. one per `if`/`else` branch) — Logos rejects `re-%init`; use separate groups or one `%init` with computed `Class=expr` args.
6. **Duplicate hooks** for the same class/method in one group — undefined behavior; merge blocks.
7. **Using `%hookf` inside `%hook`** — `%hookf` is top-level (except inside `%group`).
8. **`objc_msgSend` instead of `[obj msg]`** — declare the API and use bracket syntax.
9. **`ivar_getOffset` / `object_getIvar` in tweak code** — use `MSHookIvar` when Substrate is linked.
10. **Bare `%new` without type encoding** — use `%new(v@:)` (etc.) matching the method signature.
11. **`MSFindSymbol(NULL, …)`** — use `MSFindSymbol(MSGetImageByName("/path/to/dylib"), "_symbol")`.
12. **Binary filter `.plist` in git** — use XML in the repo; `FINALPACKAGE=1` bplist-converts on release.
13. **`@available` to `%init` private hook groups** — use `IS_IOS_OR_NEWER` from `<version.h>`.

## Review checklist

When writing or reviewing tweak code:

- [ ] Hooks live in `.x`/`.xm` with `%hook` / `%hookf`, not raw `MSHook*`
- [ ] New tweak methods use `%new(typeencoding)` with exact encodings; ivar-like state uses `%property` where appropriate
- [ ] `%orig` usage is correct; each `%group` has exactly **one** `%init` in the file (not split across `if`/`else` for the same group)
- [ ] Dynamic `%hookf` symbols resolved with **`MSGetImageByName` + `MSFindSymbol(image, …)`**, never `MSFindSymbol(NULL, …)`
- [ ] Default **`Tweak.x`** unless the file uses C++ or `MSHookIvar`; do not use `.xm` “just because”
- [ ] Firmware / `%group` gates use **`IS_IOS_OR_NEWER`** (or `IS_IOS_BETWEEN`); `@available` only at public SDK call sites
- [ ] No redundant `respondsToSelector:` where OS floor or `%group` already guarantees the API; runtime checks only where version checks are intentionally light
- [ ] No `objc_msgSend` for normal calls; use `[receiver …]` with headers for private selectors
- [ ] Private ivar access in the tweak dylib uses `MSHookIvar`, not manual runtime offset/`object_getIvar` (runtime OK only in non-Substrate targets)
- [ ] Filter **`.plist`** is XML in source; release binary conversion comes from **`make package FINALPACKAGE=1`**

## Additional resources

- Directive details and edge cases: [reference.md](reference.md)
- Official syntax: https://theos.dev/docs/logos-syntax
