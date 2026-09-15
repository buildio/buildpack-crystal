> [!NOTE]
> **This is Build.io's fork.** Upstream `crystal-lang/heroku-buildpack-crystal` was
> archived on 2025-01-07; its last functional commit was in June 2018. We maintain
> this copy because we still run Crystal apps and no maintained Cloud Native
> Buildpack for Crystal exists — the community options are all classic Heroku
> buildpacks, and all of them are unmaintained.
>
> ### Why the default version is pinned
>
> Upstream resolved the Crystal version to **the latest GitHub release at build
> time** whenever an app had no `.crystal-version` file. That is not a default, it
> is a moving target: an app that had not changed in months could stop building
> because someone else shipped a release.
>
> That is not hypothetical. Crystal **1.21.0** aborts during `bin/compile` with:
>
> ```
> E: Error executing crystal:
> Unhandled exception: Arithmetic overflow (OverflowError)
>   from /crystal/src/slice.cr:271:34 in 'default_workers_count'
> ```
>
> It crashes computing its own worker count, before compiling a line of app code,
> so every app on this buildpack broke at once and none of them had changed.
>
> ### The actual bug, and the actual fix
>
> Crystal picks its worker count like this (`src/fiber/execution_context.cr`):
>
> ```crystal
> count = ENV["CRYSTAL_WORKERS"]?.try(&.to_i?) || -1
> count = Crystal::System.effective_cpu_count.to_i if count == -1
> count = System.cpu_count.to_i if count == -1
> ```
>
> Those two `.to_i` calls are unchecked. A host whose cgroup reports an absurd
> effective CPU count raises `OverflowError` before any user code runs. From 1.21.0
> execution contexts became the default, so this fires during `crystal build`
> itself and takes the compiler down with it.
>
> `CRYSTAL_WORKERS` is read **first**, and the other two lines are guarded on
> `count == -1`. Setting it skips the overflowing probe entirely. So `bin/compile`
> exports `CRYSTAL_WORKERS` (default `4`) for the build, and writes a
> `.profile.d/crystal-workers.sh` so the compiled binary gets it at runtime too —
> the binary makes the same call on startup.
>
> That fix works on every Crystal version, including ones released after this
> commit. It is the reason this buildpack now just works with no `.crystal-version`
> file at all.
>
> ### Belt and braces
>
> `DEFAULT_CRYSTAL_VERSION` is also pinned, to `1.20.3` — the last release before
> 1.21.0 put that probe on the compiler's startup path. So the default works even
> if the reasoning above is wrong somewhere.
>
> Overrides, in precedence order:
>
> | Setting | Effect |
> |---|---|
> | `.crystal-version` | that exact version |
> | `.crystal-version` containing `latest` | the old float-to-newest behaviour |
> | `DEFAULT_CRYSTAL_VERSION` env var | change the pin without editing the script |
> | `CRYSTAL_WORKERS` env var | override the worker count (default `4`) |
>
> Bumping the default is a deliberate commit here, which is the point.

> [!IMPORTANT]
> This library is no longer supported or updated by the Crystal Team,
> therefore we have archived the repository.
> 
> The contents are still available readonly.
>
> If you wish to continue development yourself, we recommend you fork it.
> We can also arrange to transfer ownership.
>
> If you have further questions, please reach out on on https://forum.crystal-lang.org
> or crystal@manas.tech

# Crystal Heroku Buildpack

You can create an app in Heroku with Crystal's buildpack by running the
following command:

```bash
$ heroku create myapp --buildpack https://github.com/crystal-lang/heroku-buildpack-crystal.git
```

The default behaviour is to use the [latest crystal release](https://github.com/crystal-lang/crystal/releases/latest).
If you need to use a specific version create a `.crystal-version` file in your
application root directory with the version that should be used (e.g. `0.17.1`).

## Requirements

In order for the buildpack to work properly you should have a `shard.yml` file,
as it is how it will detect that your app is a Crystal app.

Your application has to listen on a port defined by Heroku. It is given to you
through the command line option `--port` and the environment variable `PORT`
(accessible through `ENV["PORT"]` in Crystal). However, most web frameworks
should handle this for you.

## Testing

To test a change to this buildpack, write a unit test in `tests/run` that asserts your change and
run `make test` to ensure the change works as intended and does not break backwards compatibility.

## More info

To learn more about how to deploy a Crystal application to Heroku, read
[our blog post](http://crystal-lang.org/2016/05/26/heroku-buildpack.html).

## Older versions of Crystal

If you have and older version of Crystal (`<= 0.9`), that uses the old
`Projectfile` way of handling dependencies, please upgrade :-).
