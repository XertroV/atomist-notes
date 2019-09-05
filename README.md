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
