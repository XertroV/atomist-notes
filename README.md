# Notes on Atomist SDM development

## Local Mode Dev

### Failing builds w/ a crash in LocalGraphClient (missing versioning goal)

If attempting to build things in local mode (e.g. using `DockerBuild()`) then you need a versioning goal (or manually set things like the docker tags).

- If you don't you'll most likely get a cryptic error `TypeError: Cannot read property 'value' of undefined`, `at LocalGraphClient.<anonymous> (.../node_modules/@atomist/sdm-local/lib/sdm/binding/graph/LocalGraphClient.ts:92)`
- The line in question: `eventStore().messages().find(m => !m.value.goalSetId && m.value.sha === sha && m.value.version && m.value.branch === branch).value`
- The issue is the SDM is looking for a version and local mode doesn't provide that automatically; resulting in the crash
- An interesting note: it seems - in the case of docker - that as long as dockerImageNameCreator function is defined in the DockerBuild config the issue with docker doens't happen (even if it just returns `[]`). So the issue in this case is likely in the default equivelant.

To solve this (properly) we need to use a `Version()` goal, particularly whilst providing the `versioner` function:

```typescript
const versionGoal = new Version().with({versioner: async (v: SdmGoalEvent) => {
    // can be pretty much anything in here provided it's a string - git sha hash is always unique, reliable, and deterministic.
    return v.sha
}})
```

### DockerDeploy Goal: oddities

DockerDeploy seems not to respect `displayName`

make sure you have something sensible defined for `successPatterns` - otherwise DockerDeploy will hang waiting for stdout to match or the container to crash. (Maybe a timeout would be a good improvement)

this goal is not really meant for production use, so it does not synchronise well with the docker daemon; as far as I can tell:
* the "state" of docker expected by the goal is reset when the SDM is (re)started. This means:
  * A container named `<project>_<branch>` is expected to _not_ exist when the SDM is first started (soln: `docker rm -f <project>_<branch>` before starting SDM)
  * A _failed_ deployment for whatever reason will cause issues on subsequent runs because the code expects the container to exist (soln: restart SDM)
  * Removing the container manually while the SDM is running will cause future deployments to fail (soln: restart SDM)

### DockerBuild Goal

If the Dockerfile fails to build (via `docker build .` or w/e) the DockerBuild goal will still return `0` (success)

## Team Mode

(nothing yet)

## General

### No fulfillment for X (seen in `atomist feed` in local mode; suspected general)

Example error: `No fulfillment for docker-build#goalCreator.ts:39`

I had this error with the "new" data-driven SDM config (instead of the "old" more imperitive/side-effects-y style).

My DockerBuildGoals interface: 

```typescript
// ./lib/goals/goals.ts (based on empty-sdm seed)
export interface DockerBuildGoals extends AllGoals {
    dockerBuild: DockerBuild;
    dockerVersioning: Version;
}

// ./lib/goals/goalCreator.ts
export const DockerBuildGoalCreator: GoalCreator<DockerBuildGoals> = async sdm => {
    return {
        dockerVersioning: new Version(),
        dockerBuild: new DockerBuild()
    }
}

// ./lib/goals/goalConfigurer.ts
export const DockerBuildGoalConfigurer: GoalConfigurer<DockerBuildGoals> = async (sdm, goals) => {
    goals.dockerVersioning.with({versioner: async (v) => { return v.sha }});
}
```

During my `DockerBuildGoalConfigurer` function I only call `.this()` on `goals.dockerVersioning`.
As it happens, I needed to call `goals.dockerBuild.with(...)`; even using `{}` as the parameter fixed this issue.
Fixed version:

```typescript
// ./lib/goals/goalConfigurer.ts
export const DockerBuildGoalConfigurer: GoalConfigurer<DockerBuildGoals> = async (sdm, goals) => {
    goals.dockerVersioning.with({versioner: async (v) => { return v.sha }});
    goals.dockerBuild.with({ push: false })
}
```

