# GoServiceSkills

Claude Code plugin with skills for Go backend microservice development. The repository is also a plugin marketplace, so services install it straight from GitHub.

## Using it in a service

Add this to the service's `.claude/settings.json` and commit it:

```json
{
  "extraKnownMarketplaces": {
    "go-service-skills": {
      "source": { "source": "github", "repo": "dehuk/GoServiceSkills" }
    }
  },
  "enabledPlugins": {
    "go-service-skills@go-service-skills": true
  }
}
```

When the folder is trusted, Claude Code offers to install the plugin. Skills are invoked as `/go-service-skills:<skill-name>`; Claude also picks them up automatically when a task matches a skill's description.

Access to this repository (SSH key or GitHub auth) is required.

To pull a new version:

```
/plugin marketplace update go-service-skills
```


## Developing skills

Load the plugin from a local checkout without installing it:

```
claude --plugin-dir /path/to/GoServiceSkills
```

Validate manifests before pushing:

```
claude plugin validate .
```

Bump `version` in `.claude-plugin/plugin.json` on every release so installed copies pick up the change.
