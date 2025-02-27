---
id: computation-caching
title: Computation Caching
---

# Computation Caching

> When it comes to running tasks, caching etc., Lerna and Nx can be used interchangeably. When we say "Lerna can cache
> builds", we mean that Lerna uses Nx which can cache builds.

It's costly to rebuild and retest the same code over and over again. Lerna uses a computation cache to never rebuild the
same code twice. This is how it does it.

Before running any task, Lerna computes its computation hash. As long as the computation hash is the same, the output of
running the task is the same.

By default, the computation hash for - say - `lerna run test --scope=remixapp` includes:

- All the source files of `remixapp` and its dependencies
- Relevant global configuration
- Versions of external dependencies
- Runtime values provisioned by the user such as the version of Node
- CLI Command flags

<!-- ![computation-hashing](../images/caching/computation-hashing.png) -->

![computation-hashing](../images/caching/lerna-hashing.png)

> This behavior is customizable. For instance, lint checks may only depend on the source code of the project and global
> configs. Builds can depend on the dts files of the compiled libs instead of their source.

After Lerna computes the hash for a task, it then checks if it ran this exact computation before. First, it checks
locally, and then if it is missing, and if a remote cache is configured, it checks remotely.

If Lerna finds the computation, Lerna retrieves it and replays it. Lerna places the right files in the right folders and
prints the terminal output. From the user’s point of view, the command ran the same, just a lot faster.

![cache](../images/caching/cache.png)

If Lerna doesn’t find a corresponding computation hash, Lerna runs the task, and after it completes, it takes the
outputs and the terminal logs and stores them locally (and if configured remotely as well). All of this happens
transparently, so you don’t have to worry about it.

Although conceptually this is fairly straightforward, Lerna optimizes this to make this experience good for you. For
instance, Lerna:

- Captures stdout and stderr to make sure the replayed output looks the same, including on Windows.
- Minimizes the IO by remembering what files are replayed where.
- Only shows relevant output when processing a large task graph.
- Provides affordances for troubleshooting cache misses. And many other optimizations.

As your workspace grows, the task graph looks more like this:

![cache](../images/caching/task-graph-big.png)

All of these optimizations are crucial for making Lerna usable for any non-trivial workspace. Only the minimum amount of
work happens. The rest is either left as is or restored from the cache.

## Source Code Hash Inputs

The result of building/testing an application or a library depends on the source code of that project and all the source
codes of all the libraries it depends on (directly or indirectly). It also depends on the configuration files
like `package.json`, `nx.json`, and `package-lock.json`. The list of these files isn't arbitrary.

Lerna can deduce most of them by analyzing the codebase. If there are exceptions that cannot be inferred automatically,
they can be manually listed in the `implicitDependencies` property of `nx.json`.

```json
{
  "implicitDependencies": {
    "global-config-file.json": "*"
  },
  ...
}
```

## Runtime Hash Inputs

All commands listed in `runtimeCacheInputs` are invoked by Lerna, and the results are included in the computation hash
of each task. You can customize them in `nx.json`:

```json
{
  "tasksRunnerOptions": {
    "default": {
      "options": {
        "cacheableOperations": [
          "build",
          "test"
        ],
        "runtimeCacheInputs": [
          "node -v",
          "echo $IMPORTANT_ENV_VAR"
        ]
      }
    }
  }
}
```

Sometimes the amount of _runtimeCacheInputs_ can be too overwhelming and difficult to read or parse. In this case, we
recommend creating a `SHA` from those inputs. It can be done as follows:

```json
{
  "tasksRunnerOptions": {
    "default": {
      "options": {
        "cacheableOperations": [
          "build",
          "test"
        ],
        "runtimeCacheInputs": [
          "node -v",
          "echo $IMPORTANT_ENV_VAR",
          "echo $LONG_IMPORTANT_ENV_VAR | sha256sum",
          "cat path/to/my/big-list-of-checksums.txt | sha256sum"
        ]
      }
    }
  }
}
```

## Args Hash Inputs

Finally, in addition to Source Code Hash Inputs and Runtime Hash Inputs, Lerna needs to consider the arguments: For
example, `lerna run build --scope=remixapp` and `lerna run build --scope=remixapp -- --flag=true` produce different
results.

Note, only the flags passed to the npm scripts itself affect results of the computation. For instance, the following
commands are identical from the caching perspective.

```bash
> npx lerna run build --scope=remixapp
> npx lerna run build --ignore=header,footer
```

In other words, Lerna does not cache what the developer types into the terminal.

If you build/test/lint… multiple projects, each individual build has its own hash value and will either be retrieved from
cache or run. This means that from the caching point of view, the following command:

```bash
> npx lerna run build --scope=header,footer
```

is identical to the following two commands:

```bash
> npx lerna run build --scope=header
> npx lerna run build --scope=footer
```

## What is Cached

Lerna works on the process level. Regardless of the tools used to build/test/lint/etc.. your project, the results are
cached.

Lerna sets up hooks to collect stdout/stderr before running the command. All the output is cached and then replayed
during a cache hit.

Lerna also caches the files generated by a command. The list of files/folders is listed in the `outputs` property of the
project's `package.json`:

```json
{
  "nx": {
    "targets": {
      "build": {
        "outputs": [
          "./build",
          "./public/build"
        ]
      }
    }
  }
}
```

If the `outputs` property for a given target isn't defined in the project'
s `package.json` file, Lerna will look at the `targetDefaults` section of `nx.json`:

```json
{
  ...
  "targetDefaults": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [
        "./dist",
        "./build",
        "./public/build"
      ]
    }
  }
}
```

If neither is defined, Lerna defaults to caching `dist` and `build` at the root of the repository.

## Skipping Cache

Sometimes you want to skip the cache. If, for example, you are measuring the performance of a command, you can use
the `--skip-nx-cache` flag to skip checking the computation cache.

```bash
> npx lerna run build --skip-nx-cache
> npx lerna run test --skip-nx-cache
```

## Customizing the Cache Location

The cache is stored in `node_modules/.cache/nx` by default. To change the cache location, update the `cacheDirectory`
option for the task runner in `nx.json`:

```json
{
  "tasksRunnerOptions": {
    "default": {
      "options": {
        "cacheableOperations": [
          "build",
          "test"
        ],
        "cacheDirectory": "/tmp/mycache"
      }
    }
  }
}
```

## Local Computation Caching

By default, Lerna (via Nx) uses a local computation cache. Nx stores the cached values only for a week, after which they
are deleted. To clear the cache run `nx reset`, and Nx will create a new one the next time it tries to access it.

## Distributed Computation Caching

The computation cache provided by Nx can be distributed across multiple machines. You can either build an implementation
of the cache or use Nx Cloud. Nx Cloud is an app that provides a fast and zero-config implementation of distributed
caching. It's completely free for OSS projects and for most closed-sourced
projects ([read more here](https://dev.to/nrwl/more-time-saved-for-free-with-nx-cloud-4a2j)).

You can connect your workspace to Nx Cloud by running:

```bash
> npx nx connect-to-nx-cloud
```

Learn more about Nx Cloud at [https://nx.app](https://nx.app).
