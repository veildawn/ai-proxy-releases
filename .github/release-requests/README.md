# Release requests

Push a JSON file named after the version (e.g. `v0.11.9.json`) to trigger a
release via `.github/workflows/release-request.yml`. Schema:

```json
{
  "version": "v0.11.9",
  "source_ref": "main",
  "since_sha": "<previous released source SHA, optional>",
  "changelog": "更新内容 plain text (optional; auto-filled from commits since since_sha)"
}
```

If `since_sha` is set and `source_ref` resolves to the same commit, the release
is skipped. Prefer running CI against the same `source_ref` first.
