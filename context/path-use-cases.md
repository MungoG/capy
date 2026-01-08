# Use Cases for capy::path

This document describes concrete use cases for `capy::path` in the beast2 and http
libraries, with before/after code examples.

## 1. Static File Serving - Path Concatenation

The `serve_static` handler builds filesystem paths from a document root and HTTP
request paths. Currently this requires manual separator handling with platform
ifdefs.

**Current code** (serve_static.cpp):
```cpp
static void
path_cat(
    std::string& result,
    core::string_view prefix,
    core::string_view suffix)
{
    result = prefix;

#ifdef BOOST_MSVC
    char constexpr path_separator = '\\';
#else
    char constexpr path_separator = '/';
#endif
    if(result.back() == path_separator)
        result.resize(result.size() - 1);
#ifdef BOOST_MSVC
    for(auto& c : result)
        if(c == '/')
            c = path_separator;
#endif
    for(auto const& c : suffix)
    {
        if(c == '/')
            result.push_back(path_separator);
        else
            result.push_back(c);
    }
}

// Usage:
std::string path;
path_cat(path, impl_->path, p.path);
```

**With std::filesystem::path**:

```cpp
// Convert UTF-8 to path - but how?
std::string utf8_path = p.path;  // UTF-8 from HTTP request

// WRONG: On Windows, this interprets utf8_path using ANSI codepage
std::filesystem::path filepath = impl_->root / utf8_path;

// Correct but verbose (C++20):
std::filesystem::path filepath = impl_->root / std::filesystem::path(
    std::u8string(utf8_path.begin(), utf8_path.end()));

// Note: std::filesystem::u8path(utf8_path) would be cleaner, but it was
// deprecated in C++20 and removed in C++26.

// No validation - malicious paths like "../../../etc/passwd" are accepted
```

**With capy::path**:

Since `p.path` comes from an HTTP request (untrusted input), it must be validated
before use. The `path_view` constructor validates and throws on invalid input:

```cpp
// Throwing version - validates p.path
capy::path filepath = impl_->root / path_view(p.path);
```

Or use the non-throwing API for explicit error handling:

```cpp
// Non-throwing version
auto rel = try_parse_path_view(p.path);
if(!rel)
    return rel.error();  // Invalid path from client
capy::path filepath = impl_->root / *rel;
```

The entire `path_cat()` function is eliminated. Separator handling is automatic.
Invalid or malicious paths (containing `../` traversal, illegal characters, etc.)
are rejected at the validation point rather than silently accepted.

## 2. Extension Extraction for MIME Type

The handler extracts file extensions to determine Content-Type. Currently this
uses manual string searching.

**Current code** (serve_static.cpp):
```cpp
static core::string_view
get_extension(core::string_view path) noexcept
{
    auto const pos = path.rfind(".");
    if(pos == core::string_view::npos)
        return core::string_view();
    return path.substr(pos);
}

// Usage:
auto mt = mime_type(get_extension(path));
```

**With std::filesystem::path**:
```cpp
auto ext = filepath.extension();  // Returns std::filesystem::path, allocates

// WRONG on Windows: string() returns ANSI encoding, not UTF-8
auto mt = mime_type(ext.string());

// Correct but verbose (C++20):
auto u8ext = ext.u8string();  // Returns std::u8string
auto mt = mime_type(std::string(u8ext.begin(), u8ext.end()));
```

**With capy::path**:
```cpp
auto mt = mime_type(filepath.extension().string_view());
```

The `get_extension()` function is eliminated. Edge cases like `.gitignore`
(no extension) and `file.tar.gz` (extension is `.gz`) are handled correctly.

## 3. Document Root Storage with Validation

The handler stores the document root path. Currently any string is accepted
without validation.

**Current code** (serve_static.cpp):
```cpp
struct serve_static::impl
{
    impl(
        core::string_view path_,
        options const& opt_)
        : path(path_)  // No validation
        , opt(opt_)
    {
    }

    std::string path;  // Could be invalid
    options opt;
};
```

**With std::filesystem::path**:
```cpp
struct serve_static::impl
{
    impl(
        std::filesystem::path root_,
        options const& opt_)
        : root(std::move(root_))  // No validation
        , opt(opt_)
    {
    }

    std::filesystem::path root;  // Could contain invalid characters
    options opt;
};

// Construction from UTF-8 requires:
serve_static handler(std::filesystem::path(
    std::u8string(doc_root.begin(), doc_root.end())),
    opts);

// Note: std::filesystem::u8path(doc_root) would be cleaner, but it was
// deprecated in C++20 and removed in C++26.
```

**With capy::path**:
```cpp
struct serve_static::impl
{
    impl(
        capy::path root_,
        options const& opt_)
        : root(std::move(root_))  // Already validated
        , opt(opt_)
    {
    }

    capy::path root;  // Guaranteed valid UTF-8
    options opt;
};
```

Invalid paths are rejected at construction time with a clear error.

## 4. Opening Files

The handler opens files using string paths. On Windows this requires conversion
to wide strings for proper Unicode support.

**Current code** (serve_static.cpp):
```cpp
system::error_code ec;
capy::file f;
f.open(path.c_str(), capy::file_mode::scan, ec);
```

**With std::filesystem::path**:
```cpp
system::error_code ec;
capy::file f;

// std::filesystem::path works with standard streams:
std::ifstream ifs(filepath);  // OK

// But capy::file takes a string, so you need:
f.open(filepath.c_str(), capy::file_mode::scan, ec);
```

**With capy::path**:
```cpp
system::error_code ec;
capy::file f;
f.open(filepath, capy::file_mode::scan, ec);
```

The `capy::file::open()` would accept `capy::path` directly, using
`native_wstring()` on Windows internally.

## 5. Appending Index File

When serving directories, the handler appends "index.html" to the path.

**Current code** (serve_static.cpp):
```cpp
if(p.parser.get().target().back() == '/')
{
    path.push_back('/');
    path.append("index.html");
}
```

**With std::filesystem::path**:
```cpp
if(p.parser.get().target().back() == '/')
    filepath /= "index.html";  // OK - same syntax
```

**With capy::path**:
```cpp
if(p.parser.get().target().back() == '/')
    filepath /= "index.html";
```

This operation is equivalent in both. The difference is in how the path was
constructed and how it will be used.

## 6. Complete Refactored Handler

Combining all the above, the handler code is significantly simplified.

**Current code** (~40 lines):
```cpp
std::string path;
path_cat(path, impl_->path, p.path);
if(p.parser.get().target().back() == '/')
{
    path.push_back('/');
    path.append("index.html");
}

system::error_code ec;
capy::file f;
std::uint64_t size = 0;
f.open(path.c_str(), capy::file_mode::scan, ec);
// ...
auto mt = mime_type(get_extension(path));
```

**With std::filesystem::path** (correct cross-platform):
```cpp
// Convert UTF-8 input to std::filesystem::path (C++20)
std::string utf8_input = p.path;
std::filesystem::path rel_path(std::u8string(
    utf8_input.begin(), utf8_input.end()));

// Note: std::filesystem::u8path(utf8_input) would be cleaner, but it was
// deprecated in C++20 and removed in C++26.

// No validation - "../../../etc/passwd" is accepted

std::filesystem::path filepath = impl_->root / rel_path;
if(p.parser.get().target().back() == '/')
    filepath /= "index.html";

system::error_code ec;
capy::file f;

f.open(filepath.c_str(), capy::file_mode::scan, ec);

// Get extension as UTF-8 (verbose)
auto u8ext = filepath.extension().u8string();
auto mt = mime_type(std::string(u8ext.begin(), u8ext.end()));
```

**With capy::path**:
```cpp
// Validate untrusted input from HTTP request
auto rel = try_parse_path_view(p.path);
if(!rel)
    return rel.error();

capy::path filepath = impl_->root / *rel;
if(p.parser.get().target().back() == '/')
    filepath /= "index.html";

system::error_code ec;
capy::file f;
f.open(filepath, capy::file_mode::scan, ec);
// ...
auto mt = mime_type(filepath.extension().string_view());
```

## Summary

| Aspect | std::filesystem::path | capy::path |
|--------|----------------------|------------|
| UTF-8 construction | Iterator conversion to `std::u8string` | Direct from `std::string` |
| UTF-8 extraction | `u8string()` + iterator conversion | `string()` or `string_view()` |
| Decomposition | Allocates new `path` objects | Returns views (zero allocation) |
| Validation | None | At construction |
| File opening | `c_str()` returns native type | `native_wstring()` internally |
| Separator handling | Automatic | Automatic |

### Key Benefits of capy::path

| Benefit | Description |
|---------|-------------|
| Eliminates manual separator handling | No more `#ifdef BOOST_MSVC` blocks |
| Eliminates encoding conversion | UTF-8 is the native format |
| Type-safe path operations | Can't accidentally pass arbitrary strings |
| Validation at construction | Bad paths fail early with clear errors |
| Zero-allocation decomposition | `extension()` returns a view, not a copy |
| Windows Unicode support | `native_wstring()` handles UTF-8 to UTF-16 |
| Simpler API | No encoding conversion dance |
