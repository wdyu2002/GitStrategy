# Git Guide

This document is meant to serve as a repository of best practices for large, multi-environment source control management.

### Feature Development

Let's begin at the start of a sprint. `master` is observed to be the code that is currently in production.

```
A---B---C---D   [master]
```

At the beginning of each sprint, we generate a `release` branch:

```bash
git checkout -B release
```

```
A---B---C---D   [master] / [release]
```

The team agrees to take on 2 epics: (a) BL-1, and (b) BL-2.


```bash
git checkout master
git checkout -B BL-1
git checkout -B BL-2
```

```
A---B---C---D   [master] / [release] / [BL-1] / [BL-2]
```

Somewhere mid-sprint:

```
              H---I---J   [BL-2]
             /
            | E---F---G   [BL-1]
            |/
A---B---C---D             [master] / [release]
```

BL-1 is ready for release. The developer uses rebase to create a merge request on the `release` branch:

```bash
git checkout BL-1
git checkout -B BL-1-MR
git rebase release
```

```
              H---I---J   [BL-2]
             /
            | E'--F'--G'  [BL-1-MR]
            |/
            | E---F---G   [BL-1]
            |/
A---B---C---D             [master] / [release]
```

When the merge request is approved, `release` should be a simple fast-forward and `BL-1-MR` can be deleted:

```
              H---I---J   [BL-2]
             /
            | E'--F'--G'  [release]
            |/
            | E---F---G   [BL-1]
            |/
A---B---C---D             [master]
```

BL-2 is ready for release. The developer uses rebase to create a merge request on the `release` branch:

```bash
git checkout BL-2
git checkout -B BL-2-MR
git rebase release
```

```
                        H'--I'--J'   [BL-2-MR]
                       /
              E'--F'--G'             [release]
             /
            | H---I---J              [BL-2]
            |/
            | E---F---G              [BL-1]
            |/
A---B---C---D                        [master]
```

When the merge request is approved, `release` should be a simple fast-forward and `BL-2-MR` can be deleted:

```
              E'--F'--G'--H'--I'--J'   [release]
             /
            | H---I---J                [BL-2]
            |/
            | E---F---G                [BL-1]
            |/
A---B---C---D                          [master]
```

The team tests the `release` branch & decides it's good enough for release. It's time to merge the `release` branch back into `master`:

```bash
git checkout master
git merge release
```

```
                H---I---J                [BL-2]
              /
             / 
            |   E---F---G                [BL-1]
            | / 
            |/
A---B---C---D---E'--F'--G'--H'--I'--J'   [master] / [release] / [TAG: release-2018-08-01]
```

CICD deploys `release-2018-08-01` to the QA environment at the time of merge (day 0). Team performs PDV.

CICD deploys `release-2018-08-01` to the STAGING environment 2 weeks after (day 14). Team performs PDV.

CICD deploys `release-2018-08-01` to the PROD environment 4 weeks after (day 28). Team performs PDV.



At day 0, the following sprint begins, and several things need to happen: (a) feature branches should be retired as needed, else (b) feature branches should be rebased onto `master` Here we assume that in the following sprint, (a) we can retire BL-1, BL-2 still has more work, and we have picked up BL-3 for the current sprint.

```bash
git checkout BL-2
git rebase master (noop)
```

```
G'--H'--I'--J'   [master] / [release] / [BL-2]
```

We also create a new branch for BL-3 off `master`. Work ensues:

```
              N---O---P   [BL-3]
             /
            | K---L---M   [BL-2]
            |/
G'--H'--I'--J'            [master] / [release]
```

Eventually, both features make it back into the `release` branch:

```
              K'--L'--M'--N'--O'--P'   [release]
             /
            | N---O---P                [BL-3]
            |/
            | K---L---M                [BL-2]
            |/
G'--H'--I'--J'                         [master]
```

Then, `release` into `master`, tagged:

```
                N---O---P                [BL-3]
              /
             / 
            |   K---L---M                [BL-2]
            | / 
            |/
G'--H'--I'--J'--K'--L'--M'--N'--O'--P'   [master] / [release] / [TAG: release-2018-08-15]
```

[release-2018-08-15] is deployed onto the QA server.
[release-2018-08-01] is deployed onto the STAGING server.

2 Weeks later:

[release-2018-08-29] is deployed onto the QA server.
[release-2018-08-15] is deployed onto the STAGING server.
[release-2018-08-01] is deployed onto the PROD server.


Our deployment strategy ends up looking like this:

```
Day 0:
  develop BL-1 feature
  develop BL-2 feature
Day 14:
  deploy BL-1 to QA
  deploy BL-2 to QA
  develop BL-3 feature
  develop BL-4 feature
Day 28:
  deploy BL-1 to STAGING
  deploy BL-2 to STAGING
  deploy BL-3 to QA
  deploy BL-4 to QA
  develop BL-5 feature
Day 42:
  deploy BL-1 to PROD
  deploy BL-2 to PROD
  deploy BL-3 to STAGING
  deploy BL-4 to STAGING
  deploy BL-5 to QA
  develop BL-6 feature
  develop BL-7 feature
```

In tabular form:

```
                              |<----------------- managed by CI/CD pipeline ----------------->|

|---------+-------------------+------------------+-----------------------+--------------------|
|   DAY   |        DEV        |        QA        |        STAGING        |        PROD        |
|---------+-------------------+------------------+-----------------------+--------------------|
|    0    |    BL-1, BL-2     |                  |                       |                    |
|---------+-------------------+------------------+-----------------------+--------------------|
|   14    |    BL-3, BL-4     |    BL-1, BL-2    |                       |                    |
|---------+-------------------+------------------+-----------------------+--------------------|
|   28    |       BL-5        |    BL-3, BL-4    |       BL-1, BL-2      |                    |
|---------+-------------------+------------------+-----------------------+--------------------|
|   42    |    BL-6, BL-7     |       BL-5       |       BL-3, BL-4      |     BL-1, BL-2     |
|---------+-------------------+------------------+-----------------------+--------------------|
|   56    |    BL-8, BL-9     |    BL-6, BL-7    |          BL-5         |     BL-3, BL-4     |
|---------+-------------------+------------------+-----------------------+--------------------|

```
