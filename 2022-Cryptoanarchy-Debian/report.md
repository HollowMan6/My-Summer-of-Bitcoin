# Report of My Summer of Bitcoin 2022 Project - CI for CADR
## Synopsis
Before this [Summer of Bitcoin](https://www.summerofbitcoin.org/project-ideas-details?recordId=recaklAlmMd5cHfdq) project, [Crypto Anarchy Debian Repo (CADR)](github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder) lacks Continuous Integration (CI), which troubles the new coming contributors because setting up the developing environment can be complex. I finally successfully implemented the CI using GitHub Actions default runners. The CI can be triggered manually, or by sending PRs as well as pushing directly to the master branch.

The CI is divided into 2 jobs: 
- The first one is the build job. It builds the running podman environment image, upload the image to Artifacts for reuse. Then with the podman environment, builds the CADR packages, and then uploads the built Debian packages to Artifacts. 
- The second one is the test job. It runs when the build job finishes. the testing jobs are run in parallel for each package. The test job first downloads the built images and packages Artifacts uploaded in the build job, then use make test-here-basic-% and make test-here-upgrade-%(% for package name) to run tests.

## Road of Implementation
### First try
At first, I ignored the fact that there [already exists a dockerfile](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/tree/master/docker) for CADR running (although it was not for building), and setup [my own dockerfile](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/commit/1580e9e025f7ccb597b35753e004a7d5327f10ef) from scratch by adding dependencies when I encounter any errors. 

My own dockerfile turned out to [work fine on GitHub Actions](https://github.com/HollowMan6/build-cryptoanarchy-deb-repo-builder/actions/runs/2042963652) for the building process, but failed for the test process.
Initially I thought it was some more dependency issues, since the test can work on my own computer.

![](https://img-blog.csdnimg.cn/a555b51e82504433bc209712b326a615.png)

When checking [the logs](https://github.com/HollowMan6/build-cryptoanarchy-deb-repo-builder/runs/5700123801?check_suite_focus=true) I found that it’s due to the unshare issue.

![](https://img-blog.csdnimg.cn/1cd9fb66d1e042abbe920aa69458cf61.png)
![](https://img-blog.csdnimg.cn/790d483263614dd98b4e7ac85c485595.png)

Then I noticed that adding a `--privileged` parameter can fix the unshare issue for docker. But then a systemd issue just came after it. Finally I noticed the dockerfile in the codebase that already exists for CADR, and just as how it works, I made the systemd to run as the first programme.

```dockerfile
VOLUME [ "/sys/fs/cgroup" ]
CMD ["/lib/systemd/systemd"]
```

and add `--tmpfs /tmp --tmpfs /run --tmpfs /run/lock -v /sys/fs/cgroup:/sys/fs/cgroup:ro` as additional parameters to share some host machine resources and make it as the daemon container. Operations are done using docker exec to attach to the container. But still it doesn’t fix the systemd issue.

Finally I switched to podman and the systemd issue got fixed (it supports `--systemd=true`) because I occasionally found [this article](https://developers.redhat.com/blog/2019/04/24/how-to-run-systemd-in-a-container) when I Googled the issue.

However, a [new issue occurs](https://github.com/HollowMan6/build-cryptoanarchy-deb-repo-builder/runs/5708913219?check_suite_focus=true) suggesting failed to override dbcache.

![](https://img-blog.csdnimg.cn/a5283aca019d4103b144765555134a6c.png)

Can't see any errors from [the workflow logs](https://github.com/HollowMan6/build-cryptoanarchy-deb-repo-builder/runs/5710646832?check_suite_focus=true), even though the missing bc dependency has been fixed and I can confirm that [the command here](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/blob/master/pkg_specs/bitcoin-@variant.sps#L88) works correctly in the container running on my PC. Moreover when I manually skip the test just mentioned, [the test](https://github.com/HollowMan6/build-cryptoanarchy-deb-repo-builder/runs/5710055525?check_suite_focus=true,) further shows that there is an unneeded reindex

![](https://img-blog.csdnimg.cn/37b06ec8572a4616a499d743b9af4fdd.png)

Which suggests that it's related to [issue 108](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/issues/108) maybe? Have opened [an issue here](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/issues/202). I can skip that test with [this commit](https://github.com/HollowMan6/build-cryptoanarchy-deb-repo-builder/commit/4defba982028724de70b9ac26d27a59cb150c4c6), but [further error](https://github.com/HollowMan6/build-cryptoanarchy-deb-repo-builder/runs/5713299561?check_suite_focus=true) suggesting electrs is not available. It can be fixed with [this PR](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/pull/203)

![](https://img-blog.csdnimg.cn/a6658b680e5446f6871a25319dfbfeeb.png)

Even if those issues are fixed, another platform may still be needed since [the full test requires too much space](https://github.com/HollowMan6/build-cryptoanarchy-deb-repo-builder/runs/5713729925?check_suite_focus=true)

![](https://img-blog.csdnimg.cn/cae7728b10214519bd25cf3748417448.png)

[Only testing the regtest part](https://github.com/HollowMan6/build-cryptoanarchy-deb-repo-builder/runs/5750416035?check_suite_focus=true) will be fine with no space issue.

### Fixing the tests

The first issue is to find out why overriding dbcache fails, This takes me quite a few weeks to find out [the solution](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/issues/199) Originally, when determining bitcoin dbcache size, bc is invoked. But bc is not guaranteed to be installed, thus it can fail. Then instead of doing maths wizardry, I [submitted a PR](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/pull/200) to just match on ranges: `RAM < 1024` -> `default dbcache`, `RAM < 2048` -> `dbcache=512`, ..., `RAM => 8192` -> `dbcache=4096`. However, the failed to override dbcache still exists when running in GitHub Actions workflow with this PR. It's very weird, as when I run the test manually using the same methods from the workflow with the podman container on my PC, such an issue won't appear. So the issue seems to belong to GitHub Actions workflow environment only even though it's running in the container.

Then after a lot of trial and error, I finally found the culprit. By [referring to PR](https://github.com/SinusBot/docker/pull/40), previously the debcrafter dropped all capabilities which seems to cause errors when the host kernel capabilities do not match those known to `setpriv`. We need to only drop supported capabilities by the current kernel.

![](https://img-blog.csdnimg.cn/1b152561e4614738840551c91d92a07b.png)

Then I [submitted a PR](https://github.com/Kixunil/debcrafter/pull/52 ) to fix this, and got merged.

The second issue is to find out why the unneeded reindex error exists. I noticed that although the test failed on test-here-basic-electrs with unneeded reindex when running make test, running make test-here-basic-electrs alone won't fail. So prior to fixing this issue, my guess was that maybe it's still related to issue [issue 108](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/issues/108), which is that after updating existing non-pruned nodes in experimental `-reindex` was used despite pruning not being changed. Or the test environment didn't get cleaned up before `test-here-basic-electrs` when executing make test. My final result proves that later is right, and [submitted PR](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/pull/204) to clean up the chain mode marker, since we want it to be clean with package_clean_install.sh

I also submitted [a PR](https://github.com/Kixunil/debcrafter/pull/53) to force linked-hash-map version to be 0.5.4 for fixing the build issue of debcrafter that just came up during the project period.

[The test can be successful](https://github.com/HollowMan6/build-cryptoanarchy-deb-repo-builder/runs/6537201901?check_suite_focus=true) on GitHub Actions when only test with regtest and after [PR 200](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/pull/200), [201](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/pull/201), [203](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/pull/203), [204](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/pull/204) get merged.

### Reseach on cloud provider
I researched on which cloud service to use. As I checked on cloud service providers AWS, Google Cloud and Azure. I find that GitHub Actions uses Azure as their [default runners](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners#cloud-hosts-used-by-github-hosted-runners), to build a GitHub self-hosted runner, Azure would be a good choice if we use the same provider as the GitHub default hosted runner. Also Azure has a [trial period](https://azure.microsoft.com/en-us/free/) with 200 dollars for one month when starting a new account, so we can start free, while other cloud service providers don’t offer such a great discount.

### Investigation on other possible CI platforms
I find out that unlike GitHub Actions that runs CI in a virtual machine, Travis CI, GitLab CI/CD, Jenkins all run the CI inside a predefined Docker containers that [without systemd support](https://stackoverflow.com/a/61705398/14343335). Then it would be not possible to run a systemd supported podman container inside such container. Azure DevsOps is a valid one for our use case, and I also tried to [use it](https://github.com/HollowMan6/build-cryptoanarchy-deb-repo-builder/blob/master/azure-pipelines.yml). Then I notice that for a single ci job, each step, it only allows 1 hour maximum, otherwise it would [got cancelled](https://dev.azure.com/HollowMan6/cryptoanarchy-deb-repo-builder/_build/results?buildId=35&view=logs&j=12f1170f-54f2-53f3-20dd-22fc7dff55f9), while our build job as well as full net test job last much longer than 1 hour, it's also not suitable for the our. Then finally the GitHub Actions would be the only good choice to have.

### Self-hosted runners for GitHub Actions
I had also tried to use Azure Virtual Machines service by myself to set up Self-hosted runners for GitHub Actions, but the environment is [contaminated from previous runs](https://github.com/HollowMan6/build-cryptoanarchy-deb-repo-builder/runs/7145902738?check_suite_focus=true). Also, [the issues](https://github.com/HollowMan6/build-cryptoanarchy-deb-repo-builder/commit/e75b7a8bd2d9e5370c79f81a6c6e81bee2d473b1) for running tests on both the mainnet and regtest still exists on my physical machine, although the regtest one has already been solved. 

### Divide and Conquer
Finally I come up with a new way for testing. We can use the divide and conquer to bypass the short of disk space issue as well as the contaminated running environment issue when running tests on both the mainnet and regtest. I can see that the make test is composed of make test-here-basic-% and make test-here-upgrade-%(% for package name), we can just use [the matrix](https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs#using-a-matrix-strategy) in GitHub Actions to test each package in separate environments, and then the disk space would be enough, and [PR 204](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/pull/204) can be closed because we now always have a clean environment.

Then I successfully implemented and tested that method, and [it works](https://github.com/HollowMan6/build-cryptoanarchy-deb-repo-builder/actions/runs/2715251380)!  So right now it's the full test and I have reached the project goal with even a solution that has no cost for ci building and testing!

The ci can be successful with the previous tests fixing PRs get merged first.

## Final Deliverables
After my mentor [@Kixunil](https://github.com/Kixunil)'s review, I began to use `--locked` parameter to make cargo use `cargo.lock` which contains correct versions. Also I fixed a security issue, make user account `user` and build/test with `user`, and upload the sha256sum result for the built deb packages afterwards so people can check hashes.

In addition, I fixed tests for bitcoin-regtest after setting bitcoind nosettings enabled in [PR 205](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/pull/205).

The CI runs successfully with the above changes, and everything works fine, nothing left to do.

## Conclusion
The Summer of Bitcoin project this time is a great experience for me as I learned a lot about Bitcoin and related DevOps knowledge.

If you are interested in Bitcoin and would like to start contributing to Cryptoanarchy Debian, with my work, you can easily fork the repo and add more test or new packages by committing the code to your fork's master branch on GitHub, the CI will help you build the deb packages and locate possible errors, no need to setup the developing environment on your computer again.

## Summary of PRs submitted
- [Set bitcoind log file under /var/log](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/pull/196)
- [Set bitcoind nosettings enabled by default](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/pull/197)
- [[SoB] Add GitHub workflow for CI](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/pull/198)
- [Remove bc dependency for overriding dbcache](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/pull/200)
- [Change all apt into apt-get in test](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/pull/201)
- [(Issue) Bug: Test failed for bitcoin-regtest with error unneeded reindex for test-here-basic-electrs](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/issues/202)
- [Try more times to test connecting to electrs-regtest](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/pull/203)
- [Fix unneeded reindex error running tests](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/pull/204)
- [Fix tests for bitcoin-regtest after setting bitcoind nosettings enabled](https://github.com/debian-cryptoanarchy/cryptoanarchy-deb-repo-builder/pull/205)
- [Fix libcap-ng is too old for 'all' caps](https://github.com/Kixunil/debcrafter/pull/52)
- [Force linked-hash-map version to be 0.5.4](https://github.com/Kixunil/debcrafter/pull/53)
- [rustyline downgrade to 6.3.0 to fix compiling issue](https://github.com/debian-cryptoanarchy/cadr-guide/pull/14)
