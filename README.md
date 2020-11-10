# JuliaGPU Buildkite

This repository contains resources related to the JuliaGPU Buildkite CI infrastructure:

- a custom Ubuntu-based image that can be based off of another image (e.g.
  CUDA's images);
- support for encrypted environment variables in the pipeline, and an
  environment hook to decode them;
- Docker Compose templates and systemd service files to tie everything together,
  and give each job a safe and reproducible execution environment.

Many of these features come from the fact that the JuliaGPU CI is intended to be
run on a variety of repositories (and external PRs to those repositories),
whereas Buildkite is typically used within the (trusted) boundaries of a single
organization.

This README documents how to use the JuliaGPU Buildkite CI. For details on the available
agents, or how to add one, see the [agents README](agents/README.md). **Note that this
Buildkite set-up is fairly new, and the set-up may still change. Keep an eye on this README
if anything breaks.**


## Adding a repository

If your Julia package can use a GPU, you may use the JuliaGPU CI infrastructure to run tests
on a system with a GPU. This is **not a general-purpose CI service**, and only intended to
run GPU-related tests of Julia packages. For all other testing, use public infrastructure
(Travis, Github Actions, etc).

To enable the JuliaGPU CI for your repository, follow the steps below. Some of these steps
are to be performed by an admin with access to the JuliaLang Buildkite instance, so you
should ask on the Julia Discourse or Slack about that. Other steps generally need
administrative priviliges on the repository you want to add CI to.

1. (by a Buildkite admin) [Create a new
   pipeline](https://buildkite.com/organizations/julialang/pipelines/new), using the
   package's name.

   a. Under `Git Repository`, select `Any account` and specify the package's HTTPS clone
      URL, e.g. `https://github.com/JuliaGPU/CUDA.jl.git`. Do not use the `git://` URL!

   b. Remove the default step, instead adding the `Read steps from repository` action. Make
      sure to target the JuliaGPU queue under `Agent Targeting Rules` by specifying
      `queue=juliagpu`.

   c. Grant permission to the JuliaGPU team.

2. Apply the proposed GitHub webhook settings to your repository.

3. (by a Buildkite admin) Navigate to the Pipeline Settings.

   a. Under general settings, make the pipeline public by clicking the big green button.

   b. Under GitHub settings, check the box to `Build pull requests from third-party forked
      repositories`.

4. [Install the Buildkite
   app](https://github.com/settings/connections/applications/Iv1.112bf4be3e5ecdeb) so that
   CI statusses can be pushed to GitHub.

5. In your repository, create `.buildkite/pipeline.yml`. Use this as a starting point:

   ```yaml
   env:
     SECRET_CODECOV_TOKEN: "..."

   steps:
     - label: "Julia 1.5"
       plugins:
         - JuliaCI/julia#v0.2:
             version: 1.5
         - JuliaCI/julia-test#v0.2: ~
         - JuliaCI/julia-coverage#v0.2:
             codecov: true
       agents:
         queue: "juliagpu"
         cuda: "*"
   ```

    If you want to perform several tests, or introduce test phases, you may want to add
    additional steps. Note that each step is run in a fresh container (as opposed to how
    Buildkite normally works), so you need to repeat the set-up that provides Julia (here
    done by the `julia` plugin) and/or instantiates your project (done by the `julia-test`
    plugin). If you need to send resources across steps, use
    [artifacts](https://buildkite.com/docs/pipelines/artifacts).

    For coverage submission to Codecov to work, you need to encrypt your `CODECOV_TOKEN` and
    specify it as a global `SECRET_CODECOV_TOKEN` (see below).



## Using secrets

During start-up, agents will scan for `SECRET_` environment variables and decrypt their
contents for use in the rest of the pipeline. If you want to use this mechanism to provide,
say, a secret `CODECOV_TOKEN`, run the `encrypt` script in this repository and follow its
prompts:


```
$ ./tools/encrypt
Variable name: CODECOV_TOKEN
Secret value:

Use the following snippet in your pipeline.yml:

env:
  SECRET_CODECOV_TOKEN: "kaIXEN51HinaQ4JGclQcIgxeMMtXDb5uvnP3E2eKrH4Eruf2pKd5QwUGcIVL8+rcWeo5FWj883rNxRQEH3YeCWs6/i7vzs+ORvG51QeCNYQgNqFzPsWRcq5qJYc+JPFbisS7q9nghqWTwr52cnjarD4Xx3ceGorMyS5NvFpCNxMgqHNyGkLvipxcTTJfKZK61bpnbntoIjiIO1XSZKjcxnXFGFnolV9BHCr5v8f7F42n2tUH7X3nDHmTBr1AbO2lFAU9ra/KezHcIf0wg2HcV8LZD0+mj8q/SBPjQZSH7cxwx4Q2eTjT4Sw7xnrBGuySVm8ZPCAV7nRNEHo+VqR+GQ=="
```
