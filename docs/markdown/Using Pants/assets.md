---
title: "Assets and archives"
slug: "assets"
excerpt: "How to include assets such as images and config files in your project."
hidden: false
createdAt: "2020-09-28T23:07:26.956Z"
updatedAt: "2022-01-29T16:32:14.551Z"
---
There are two ways to include asset files in your project: `resource` and `file` targets.

`resources`
-----------

A [`resource`](doc:reference-resource) target is for files that are members of code packages, and are loaded via language-specific mechanisms, such as Python's `importlib.resources.read_text()` or Java's `getResource()`.  

Pants will make resources available on the appropriate runtime path, such as Python's `PYTHONPATH` or the JVM classpath. Resources can be loaded directly from a binary in which they are embedded, such as a Pex file, without first unpacking it.

To reduce boilerplate, the [`resources`](doc:reference-resources) target generates a `resource` target per file in the `sources` field.

For example, to load resources in Python:

```python src/python/project/app.py
import importlib.resources

if __name__ == "__main__":
    config = importlib.resources.read_text("project", "config.json")
    print(f"Config: {config}")
```
```python src/python/project/BUILD
python_source(
  name="app",
  source="app.py",
  # Pants cannot infer this dependency, so we explicitly add it.
  dependencies=[":config"],
)

resource(
  name="config",
  source="config.json",
)
```
```json src/python/project/config.json
{"k1": "v", "k2": "v"}
```

[Source root](doc:source-roots) stripping applies to resources, just as it does for code. In the example above, Python loads the resource named `project/config`, rather than `src/python/project/config.json`. 

`files`
-------

A `file` target is for loose files that are copied into the chroot where Pants runs your code. You can then load these files through direct mechanisms like Python's `open()` or Java's `FileInputStream`. The files are not associated with a code package, and must be extracted out of a deployed archive file before they can be loaded.

To reduce boilerplate, the [`files`](doc:reference-files) target generates a `file` target per file in the `sources` field.

For example, to load loose files in Python:

```python src/python/project/app_test.py
def test_open_file():
    with open("src/python/project/config.json") as f:
        content = f.read().decode()
    assert content == '{"k1": "v", "k2": "v"}'
```
```python src/python/project/BUILD
python_test(
    name="app_test",
    source="app_test.py",
    # Pants cannot infer this dependency, so we explicitly add it.
    dependencies=[":config"],
)

file(
    name="config",
    source="config.json",
)
```
```json src/python/project/config.json
{"k1": "v", "k2": "v"}
```

Note that we open the file with its full path, including the `src/python` prefix.

> 🚧 `file` targets are not included with binaries like `pex_binary`
> 
> Pants will not include dependencies on `file` / `files` targets when creating binaries like `pex_binary` and `python_awslambda` via `pants package`. Filesystem APIs like Python's `open()` are relative to the current working directory, and they would try to read the files from where the binary is executed, rather than reading from the binary itself.
> 
> Instead, use `resource` / `resources` targets or an `archive` target.

When to use each asset target type
----------------------------------

### When to use `resource`

Use `resource` / `resources`  for files that are associated with (and typically live alongside) the code that loads them. That code's target (e.g. `python_source`) should depend on the `resource` target, ensuring that code and data together are embedded directly in a binary package, such as a wheel, Pex file or AWS Lambda.

### When to use `file`

Use `file` / `files` for files that aren't tightly coupled to any specific code, but need to be deployed alongside a binary, such as images served by a web server.

When writing tests, it is also often more convenient to open a file than to load a resource.

|                       | `resource`                                                                                      | `file`                                                |
| :-------------------- | :---------------------------------------------------------------------------------------------- | :---------------------------------------------------- |
| **Runtime path**      | Relative to source root                                                                         | Relative to repo root                                 |
| **Loading mechanism** | Language's package loader, relative to package                                                  | Language's file loading idioms, relative to repo root |
| **Use with**          | Targets that produce binaries, such as `pex_binary`, `python_distribution`, `python_awslambda`. | `archive` targets, tests                              |

`relocated_files`
-----------------

When you use a `file` target, Pants will preserve the path to the files, relative to your build root. For example, the file `src/assets/logo.png` in your repo would be under this same path in the runtime chroot.

However, you may want to change the path to something else. For example, when creating an `archive` target and setting the `files` field, you might want those files to be placed at a different path in the archive; rather than `src/assets/logo.png`, for example, you might want the file to be at `imgs/logo.png`.

You can use the `relocated_files` target to change the path used at runtime for the files. Your other targets can then add this target to their `dependencies` field, rather than using the original `files` target:

```python src/assets/BUILD
# Original file target.
file(
    name="logo",
    source="logo.png",
)

# At runtime, the file will be `imgs/logo.png`.
relocated_files(
    name="relocated_logo",
    files_targets=[":logo"],
    src="src/assets",
    dest="imgs",
)
```

You can use an empty string in the `src` to add to an existing prefix and an empty string in the `dest` to strip an existing prefix.

If you want multiple different re-mappings for the same original files, you can define multiple `relocated_files` targets.

The `relocated_files` target only accepts `file` and `files` targets in its `files_targets` field. To relocate where other targets like `resource` and `python_source` show up at runtime, you need to change where that code is located in your repository.

`archive`: create a `zip` or `tar` file
---------------------------------------

Running `pants package` on an `archive` target will create a zip or tar file with built packages and/or loose files included. This is often useful when you want to create a binary and bundle it with some loose config files.

For example:

```python project/BUILD
archive(
    name="app_with_config",
    packages=[":app"],
    files=[":production_config"],
    format="tar.xz",
)
```

The format can be `zip`, `tar`, `tar.xz`, `tar.gz`, or `tar.bz2`.

The `packages` field is a list of targets that can be built using `pants package`, such as `pex_binary`, `python_awslambda`, and even other `archive` targets. Pants will build the packages as if you had run `pants package`. It will include the results in your archive using the same name they would normally have, but without the `dist/` prefix.

The `files` field is a list of `file`, `files`, and `relocated_files` targets. See [resources](doc:resources) for more details.

You can optionally set the field `output_path` to change the generated archive's name.
