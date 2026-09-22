# CPP Refactor Suite Documentation

Rename, move and delete C++ classes in Unreal Engine, with the reference sweep, CoreRedirects, project file regeneration and incremental rebuild handled for you.

**Requirements:** Windows, Unreal Engine 5.6–5.8, a C++ project, and a working Visual Studio toolchain. This is the same setup Unreal already needs to compile C++. Rider and other IDEs are untested and unsupported.

## Overview

The plugin adds a **CPP Refactor Suite** entry under the editor's **Tools** menu. The panel lists every class in your project's `Source` folder and offers four actions: find references, rename, move, delete.

Every operation runs in two halves.

**In the editor**, the plugin edits your source on disk: renaming files, sweeping references, rewriting include paths, and writing CoreRedirects where they are needed. All of this is finished and saved before anything else happens.

**After the editor closes**, a cleanup commandlet finishes the job: it removes the stale Unreal Header Tool output for the affected class, regenerates your project files, and starts an incremental rebuild in a console window.

The editor has to close because Unreal holds your compiled module DLLs open while it is running, and those DLLs cannot be relinked until it lets go. **The editor does not reopen on its own.** When the rebuild finishes, start it again yourself.

### What gets cleaned up

Only the affected class's generated files are removed:

```
<Project>\Intermediate\**\<ClassName>.generated.h
<Project>\Intermediate\**\<ClassName>.gen.cpp
```

Nothing else in `Intermediate` is touched.

Generated file cleanup runs for **rename** and **delete**. A move doesn't change any class name, so there is no stale Unreal Header Tool output to remove.

### The commands it runs

Nothing here is hidden. These are the exact command lines, and you can verify every one of them in the log files after an operation.

**1. The editor launches the cleanup commandlet as it exits:**

```
"<Engine>\Binaries\Win64\UnrealEditor-Cmd.exe" "<Project>.uproject"
    -run=PostOpCleanupCommandlet
    -WaitForPID=<editor process id>
    -OldClassName=<OldName>
```

`-OldClassName` is present for rename and delete, and absent for move. The commandlet host is derived from the editor executable you were actually running, so a DebugGame Editor session launches the matching DebugGame host rather than the Development one.

**2. The commandlet regenerates your project files:**

```
"<Engine>\Binaries\ThirdParty\DotNet\<version>\win-x64\dotnet.exe"
    "<Engine>\Binaries\DotNET\UnrealBuildTool\UnrealBuildTool.dll"
    "<Project>.uproject" <ProjectName> Win64 Development -ProjectFiles
```

The .NET runtime is the one bundled inside your engine install, not whatever is on your system PATH. UnrealBuildTool is a framework-dependent .NET application and the machine-wide runtime is often older than the engine requires.

**3. The commandlet starts the rebuild in a new console window:**

```
cmd.exe /k powershell -Command "Wait-Process -Id <commandlet pid> -ErrorAction SilentlyContinue"
    & "<Engine>\Binaries\ThirdParty\DotNet\<version>\win-x64\dotnet.exe"
      "<Engine>\Binaries\DotNET\UnrealBuildTool\UnrealBuildTool.dll"
      <EditorTarget> Win64 <Configuration>
      -Project="<Project>.uproject"
      -TargetType=Editor -WaitMutex -FromMsBuild
```

The `Wait-Process` step exists because UnrealBuildTool has to relink the plugin's own DLL, which stays locked until the commandlet process has fully exited.

`<EditorTarget>` is read from your `Source\*Editor.Target.cs` rather than assumed, so projects that don't follow the `<ProjectName>Editor` convention still build. `<Configuration>` matches the editor you were running, so a DebugGame user doesn't end up with stale binaries.

The build is **incremental**. The stale `.generated.h` and `.gen.cpp` are already gone, so UnrealBuildTool rebuilds the affected module and its dependents, not your whole project.

The window is left open deliberately so you can read the build output.

### Log files

Every operation writes two logs, both in your project root:

| File | Written by |
|---|---|
| `CPP_Refactor_Suite_Log.txt` | The editor half. File renames, reference updates, and redirects |
| `CPP_Refactor_Suite_UBT_Log.txt` | The cleanup half. Generated file deletion, regeneration, and rebuild |

Both are appended to rather than overwritten, because a broken Blueprint is often noticed several renames later. Each session banner records the date, engine version, build configuration and a **session id,** which is the editor's process id. The same id appears in both files, so you can match the two halves of one operation.

If anything fails, the log says what happened and what to run by hand. It is the first thing to attach to a support request.

### Supported types

Rename resolves the selected header to a reflected type and writes the correct redirect for it:

| Declaration | Redirect written |
|---|---|
| `UCLASS` | `+ClassRedirects` |
| `UINTERFACE` | `+ClassRedirects` |
| `USTRUCT` | `+StructRedirects` |
| `UENUM` | `+EnumRedirects` |

Plain C++ types with no reflection macro are renamed on disk and swept through your source, but no redirect is written because nothing serialized can reference them by name.

Interfaces declared in an `I`-prefixed header (`IPickupable.h`) are handled, and the plugin will refuse a rename that drops the `I`.

Redirects are rewritten from scratch on every rename, not appended. If you rename `A` to `B` and later `B` to `C`, the entry pointing at `A` is retargeted to `C` instead of leaving a dead `A → B` hop behind.

The plugin's own source is permanently excluded. It will not rename, move or delete any of its own files, and those classes do not appear in the class list.

## Before you run anything: version control

**READ THIS SECTION. IT IS THE ONE THAT MATTERS MOST.**

These operations rewrite source files across your entire project and **there is no undo**. The plugin has no backup mechanism and no rollback. Each operation warns you before it starts, and that warning is not boilerplate.

**Commit or check in your work before every operation.**

If an operation produces a result you didn't want, reverting your source control is the recovery path.

A few specifics worth knowing:

- **Perforce and other locking workflows are untested.** If your files are read-only until checked out, the plugin's writes will fail. Check out the affected files first, or check out the whole source tree before running an operation. Git and other non-locking workflows are what this is developed against.
- **`DefaultEngine.ini` is modified** by a rename. Make sure it is under source control and not locked.
- **Generated files are deleted** from `Intermediate`. These are build output and shouldn't be in source control, but if your setup tracks them, expect changes.
- **Run one operation at a time.** Let the rebuild finish and reopen the editor before starting another. Chaining operations without rebuilding in between means the second one runs against a project state that no longer matches its binaries.

## Rename flow

1. Open **Tools → CPP Refactor Suite**.
2. Select a class in the **Project Class List**.
3. *(Recommended)* Click **Find References** first to see what depends on it.
4. Click **Rename**.
5. Enter the new class name. The warning about source control is displayed here.
6. Click **OK**.

What happens next:

- The header and source file are renamed on disk.
- Every reference to the old symbol is updated across `Source` and `Plugins`.
- The matching CoreRedirect is written to `DefaultEngine.ini`, and any existing redirect chain is collapsed.
- The editor closes.
- The stale `.generated.h` and `.gen.cpp` for the old name are deleted.
- Project files are regenerated.
- An incremental rebuild starts in a console window.
- When it finishes, reopen the editor.

**Your Blueprints and assets resolve to the new name on their next load, because of the redirect. You do not need to fix them by hand.**

**The rename is refused if:**

- the class is part of the plugin's own source;
- the target is an `I`-prefixed interface header and the new name drops the `I`;
- the new name is empty.

## Move flow

1. Open **Tools → CPP Refactor Suite**.
2. Select a class in the **Project Class List**.
3. Click **Move**.
4. Choose the destination folder for the header, the source file, or both. **They move independently!** You can send the `.h` one way and the `.cpp` another.
5. The warning about source control is displayed here. Confirm.

What happens next:

- The selected files are moved.
- If the header moved, every file that includes it is updated to the new path.
- Bare includes *inside* the moved files are rewritten to be module-relative. A bare `#include "Sibling.h"` only resolves next to its siblings, so it breaks the moment the file moves and it breaks during the rebuild, after the editor has closed.
- The editor closes.
- Project files are regenerated.
- An incremental rebuild starts in a console window.

**No CoreRedirect is written for a move, and this is correct.** Asset references are based on the class name, not the file path. Moving a file doesn't change any name, so nothing serialized needs redirecting. If you were expecting a redirect entry to appear, its absence is not a bug.

Generated files are not cleaned up for a move either, because the class name is unchanged, so its Unreal Header Tool output is still valid.

**If a bare include is ambiguous:** if the same filename exists in more than one place in your project the log records a warning naming the file it chose. Check that include after the rebuild.

## Delete flow

Delete is deliberately the most conservative of the three operations.

1. Open **Tools → CPP Refactor Suite**.
2. Select a class in the **Project Class List**.
3. **Click Find References first.** This is not optional advice. Delete does not sweep or repair references, so this list is your only warning about what will break.
4. Click **Delete**.
5. Confirm. The dialog states plainly that remaining references will fail to compile.

What happens next:

- Only the `.h` and `.cpp` whose filename exactly matches the class name are removed. Nothing else is touched.
- The editor closes.
- The stale `.generated.h` and `.gen.cpp` are deleted.
- Project files are regenerated.
- An incremental rebuild starts in a console window.

**Delete does not fix references to the deleted class, and does not write any redirect.** Anything that still depends on that class will fail to compile, and any Blueprint that derived from it will lose its parent class.

That is intentional. Silently rewriting code that depends on a class you just deleted would be guessing at what you meant. Instead the rebuild tells you immediately and precisely what depended on it, in minutes rather than the next time someone opens the project.

The plugin's own classes cannot be deleted.

## Getting support

**Before you make a support request, collect the two logs** from your project root:

```
CPP_Refactor_Suite_Log.txt
CPP_Refactor_Suite_UBT_Log.txt
```

Both are needed. The first shows what happened to your source, the second shows what happened during cleanup and rebuild, and the session id ties them together. A report without them usually cannot be diagnosed.

**Include:**

- Unreal Engine version
- Which operation you ran (rename / move / delete) and on which class
- What you expected, and what actually happened
- Both log files, or the relevant session from each

**Where to go:**

| | |
|---|---|
| Bug reports | [ISSUES URL] |
| Questions and discussion | [DISCORD INVITE URL] |
| Documentation | [DOCS URL] |

Support questions are answered in the Discord's support forum. Please open a post there rather than a direct message, so the answer is searchable for the next person with the same problem.

## Known limitations

- **Windows only.** There is no macOS or Linux support.
- **Visual Studio only.** Rider and other IDEs are untested.
- **No undo.** Source control is the recovery path.
- **The editor closes and does not reopen automatically.**
- **Classes are selected from your project's own `Source` folder.** You cannot target classes inside plugins or engine source, though references to a project class are still updated inside your project's plugins.
- **Delete does not repair references.** See the delete flow above.
- **Source control integration is not built in.** Files are written directly to disk.
