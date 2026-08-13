# Case Study: VHBB — a Complete App Built on vita2d

**VHBB** (`devnoname120/vhbb`, "Vita HomeBrew Browser") is a real, shipped app-store client for the
Vita: it browses the [VitaDB](https://vitadb.rinnegatamante.it/#/) homebrew catalog, shows
descriptions/screenshots, and downloads/installs homebrews — the same category of app as this
wiki's own motivating project. 424 GitHub stars, C/C++, built with VitaSDK + CMake. Read here as a
worked example of a full app built **without porting any desktop UI/rendering library** — see also
[vita2d vs vitaGL](07-vita2d-vs-vitagl.md) and the
[porting methodology section](../06-porting-opengl-libraries-to-vitagl/README.md) for the
alternative this app deliberately avoids.

![VHBB screenshots — browse, detail view, install](https://user-images.githubusercontent.com/2824100/79699878-794ac200-8292-11ea-855e-f4ed96dbe61f.png)
*(Hosted on GitHub's user-content CDN as part of `devnoname120/vhbb`'s own README; linked here, not
mirrored.)*

## Dependency stack

| Library | Role |
|---|---|
| `vita2d` | All rendering — no vitaGL, no OpenGL at all |
| `curl` + `curlpp` (C++ wrapper) + OpenSSL | HTTP(S) to the VitaDB API and file downloads |
| `yaml-cpp` | Parses the catalog, which VitaDB exposes as YAML |
| `freetype`, `libpng`, `libjpeg`, `zlib` | Font/image decoding (vita2d's own dependencies) |
| `minizip` (vendored) | Reads/writes zip — a `.vpk` *is* a zip file under the format |
| `pthread` | Background threads for downloads/installation |
| `ftpvita` | Dev-convenience FTP server |

Every one of these is either already prebuilt for Vita by VitaSDK, or a generic C/C++ library that
cross-compiles unmodified — none of it required the kind of rendering-API translation work covered
in the [porting methodology section](../06-porting-opengl-libraries-to-vitagl/README.md).

## The main loop

```c
void mainLoopTick(Input& input, Activity& activity)
{
    vita2d_start_drawing();
    vita2d_clear_screen();

    input.Get();

    activity.FlushQueue();
    activity.HandleInput(1, input);
    activity.Display();

    vita2d_end_drawing();
    vita2d_common_dialog_update();
    vita2d_swap_buffers();
    sceDisplayWaitVblankStart();
}
```

A direct instance of the [frame lifecycle](02-initialization-and-frame-lifecycle.md) documented
above — nothing app-specific is needed beyond calling into an `Activity` object between
`start_drawing`/`end_drawing`.

## UI architecture: a screen stack, not a UI framework

```c
class View
{
public:
    virtual ~View();
    virtual int HandleInput(int focus, const Input& input);
    virtual int Display();

    std::atomic_bool request_destroy = false;
    std::atomic_uint priority = 100;
};
```

Every screen (`Views/CategoryView`, `HomebrewView`, `ListView`, `ProgressView`, ...) subclasses this
~20-line base. An `Activity` object holds a stack/queue of active `View`s and routes input/draw
calls to the one with the highest `priority`. This is the entire "UI framework" — no imported
widget toolkit, just a small hand-rolled screen-stack pattern implemented directly with vita2d draw
calls inside each `View::Display()`.

## Data model: YAML into an STL vector, no database engine

```c
class Database : public Singleton<Database>
{
public:
    std::vector<Homebrew> homebrews;
    template<class UnaryPredicate> std::vector<Homebrew> Filter(UnaryPredicate pred);
    template<class UnaryPredicate> std::vector<Homebrew> Sort(UnaryPredicate pred);
private:
    const YAML::Node db;
};
```

Despite the filename (`database.h`), there's no SQLite or any real database engine — the catalog is
fetched once, parsed into a `YAML::Node`, materialized into a `std::vector<Homebrew>`, and
filtered/sorted with ordinary STL algorithm-style predicates.

## Background work: threads with a shared progress object

Icon downloads (`fetch_load_icons_thread.cpp`) and package installation
(`install_thread.cpp`) run on dedicated `pthread`s, reporting progress through a shared
`InfoProgress` object that the main loop polls every frame to draw a progress bar — the native-
thread analog of a background worker posting progress updates to a UI thread.

## What this confirms

Nothing here required writing a `RenderInterface`, translating indexed draws, or reasoning about
`sceGxm` shaders directly. The entire app is: vita2d for drawing, curl for networking, yaml-cpp for
data, a ~20-line hand-rolled `View` base class for screens, and `pthread` for background work — the
concrete existence proof that a complete, popular Vita homebrew doesn't require porting a desktop
UI/rendering library at all.
