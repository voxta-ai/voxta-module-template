# Voxta module

## What this is

[Voxta](https://voxta.ai) is an app for talking with AI characters, assistants, integrate AI with games and run
creative tasks like image, video and music generation, using locally hosted or online models. Voxta Server runs on
the user's own machine. Users pick the services it runs on (text generation, text to speech, speech to text, vision and
others), and characters can use tools and react to games and apps. Voxta Cloud is the default service that provides
text generation, text to speech, speech to text, image generation and video generation models out of the box.

A module extends Voxta Server. It is a .NET class library that the server loads into its own process. A module can
be a service (a model provider, a voice), a chat augmentation (tools the AI calls, text in the prompt, reactions to
the chat), or a bridge to a game or another app. It has its own settings page in Voxta.

This repository is one module, `Voxta.Modules.MyModule`, made from the
[voxta-module-template](https://github.com/voxta-ai/voxta-module-template).

## Where things are

| Path | What it holds |
|---|---|
| `Voxta.Modules.MyModule/VoxtaModule.cs` | The entry point: registers the module and its services |
| `Voxta.Modules.MyModule/module.json` | Id, version, the Voxta versions it supports, label, author, category |
| `Voxta.Modules.MyModule/Configuration/` | The settings page |
| `Voxta.Modules.MyModule/Assets/` | The logo |
| `global.json` | The .NET SDK it builds with |
| `packages.lock.json` | The exact NuGet packages. Commit it: CI restores only what it names |
| `.github/workflows/` | CI build, release on a tag, raising the supported Voxta version |
| `.voxta/workspace.json` | What a Voxta chat working in this repository turns on |
| `README.md` | For people: setup, publishing, releases |

The API comes from the [`Voxta.Sdk.Modules`](https://www.nuget.org/packages/Voxta.Sdk.Modules) NuGet package. The
running Voxta provides it, so it is referenced with `ExcludeAssets="runtime"`. It pulls in `Voxta.Abstractions`,
`Voxta.Model` and `Voxta.Common`.

## Build

Install the .NET SDK that `global.json` names (https://dotnet.microsoft.com/download), then:

```bash
dotnet build
```

The build output is what Voxta loads. Keep `CopyLocalLockFileAssemblies` in the csproj: the output folder must hold
the module's own dependencies.

## Develop with Voxta's MCP server

Voxta serves an MCP server that builds this module, loads it into the running Voxta, tests it and reads its errors.
It also serves the module development skills. **Read the skills before you write code**: they describe the API of
the Voxta that is running, with code that compiles against it.

### Connect

The user needs:

1. Voxta Server on this machine. The MCP server answers only requests from that machine.
2. The **Voxta Utilities: Module Development** module, installed by an administrator from **Add Modules →
   Augmentations**. If `http://localhost:5384/api/extensions/module-dev/mcp` answers 403 or 404, ask the user to
   install it.
3. An API key with the **Module development** scope, on an administrator account. The user can create one on the
   API keys page in Voxta. Or ask Voxta for one, and the user approves it there:

   ```bash
   curl -s -X POST http://localhost:5384/api/device/code -H "Content-Type: application/json" \
     -d '{"client_id":"voxta","scope":"modules.dev","label":"Coding agent","prompt":true}'
   # Tell the user to approve it in Voxta, picking the number in "binding_message".
   curl -s -X POST http://localhost:5384/api/device/poll -H "Content-Type: application/json" \
     -d '{"device_code":"<device_code>"}'
   # 204 while waiting, then {"token": "..."}.
   ```

Add the server to the agent. Claude Code:

```bash
claude mcp add --transport http voxta-dev http://localhost:5384/api/extensions/module-dev/mcp \
  --header "Authorization: Bearer <token>"
```

Other agents take the same as JSON: `{"type": "http", "url": "http://localhost:5384/api/extensions/module-dev/mcp",
"headers": {"Authorization": "Bearer <token>"}}`. 5384 is Voxta's default port: use the port Voxta runs on. Do not
commit the token. A first build restores packages and can take minutes: give the server a long tool timeout if the
agent has one.

### Skills

`read_skill` reads one by name. Without a name, it lists them all with what each covers. They are also MCP
resources (`skill://voxta/<name>`). Start with `voxta-module-development` (the loop below, versions, load errors) and
`voxta-module-anatomy` (how a module is laid out, and which skill covers each kind of module).

The other skills explain Voxta itself: contexts and flags, actions and tools, scenarios, scripts, services. Read the
skill before you write code that relies on how Voxta behaves.

### The loop

1. Edit the code.
2. `load_dev_module` with this repository's absolute path. It runs `dotnet build`, then loads the build into Voxta
   or reloads it. It returns the build errors, or the load error with its stack trace: fix them and call it again
   until the module loads. It adds the module to the user's modules when it has chat augmentations.
3. `test_dev_module` runs the module's self-test with the user's settings, as the **Test** button in Voxta does.
   When a URL, a key, a server or a model can be wrong, give the module a self-test (see the skills).
4. The user tries the module in a Voxta chat, with the augmentation keys that `load_dev_module` returned. Each
   reload ends the chats that use the module.
5. If something fails at runtime, `read_server_log` with the module id or `ERR`. `module_status` lists what loaded
   and what failed, with the full errors.

## Without the MCP server

1. `dotnet build`.
2. In Voxta, **Settings → AI Setup → Module Sources → Add source**, then the project folder.
3. After each build, **Apply** on **Local Development Modules**. Load errors show there.

Code examples that compile: [voxta-sdk-examples](https://github.com/voxta-ai/voxta-sdk-examples), folder `modules/`.
Docs: https://doc.voxta.ai/llms.txt (index) and https://doc.voxta.ai/llms-full.txt (all pages). The docs follow the
newest Voxta, which can differ from the one the user runs: prefer the skills when the MCP server is available.

## Rules

- Do not write the user's personal details (name, email, user folder paths) into the repository. `author` in
  `module.json` is public: ask the user which name to show. You can offer their git `user.name`, but do not use it
  without asking.
- `ServiceName` in `VoxtaModule.cs` is what users' settings are saved under. Never change it once released.
- The `Voxta.Sdk.Modules` version is the oldest Voxta the module supports, and `voxta` in `module.json` says the
  same. Raise both together. A module built against a newer package than the running Voxta fails to load.
- Release: bump `version` in `module.json`, commit, tag `v<version>`, push the tag. The release workflow builds and
  publishes it. The zip must stay under 32 MB: download models and large dependencies at runtime.
