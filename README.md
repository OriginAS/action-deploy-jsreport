# action-deploy-jsreport

Imports a jsreport export file into a running jsreport instance, over SSH.

jsreport normally listens on `localhost` only, so this action uploads the export file
to the server and runs the import **on the server**. Nothing beyond the SSH port needs
to be reachable from the runner.

The import is transactional: if it fails, the instance is left with exactly what it had.

## Usage

The caller produces the export file — `jsreport export` boots jsreport in-process
against the repository's `data/` directory, so no server is needed for this step:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v6
        with:
          node-version: '22'
      - run: npm ci
      - run: npx jsreport export jsreport.jsrexport

      - uses: OriginAS/action-deploy-jsreport@v1.0.0
        with:
          export-file: jsreport.jsrexport
          server-host: ${{ vars.DEPLOY_SERVER }}
          server-ssh-key: ${{ secrets.ACTIONS_SSH_PRIVATE_KEY }}
          aws-access-key-id: ${{ secrets.ACTIONS_AWS_ACCESS_KEY }}
          aws-secret-access-key: ${{ secrets.ACTIONS_AWS_ACCESS_SECRET }}
          aws-security-group-id: ${{ vars.AWS_SECURITY_GROUP_ID }}
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `export-file` | yes | | The `.jsrexport` (or `.zip`) file to import |
| `server-host` | yes | | Host name of the server jsreport runs on |
| `server-ssh-key` | yes | | SSH private key that can access the server |
| `server-user` | no | `actions` | User to connect to the server as |
| `jsreport-url` | no | `http://localhost:5488` | Base URL of jsreport, **as seen from the server** |
| `jsreport-user` | no | | Username, if the authentication extension is enabled |
| `jsreport-password` | no | | Password, if the authentication extension is enabled |
| `full-import` | no | `true` | Mirror the instance on the export file (see below) |
| `target-folder` | no | | Shortid of a folder to import into, instead of the root |
| `aws-access-key-id` | no | | Only needed if the server sits behind a security group |
| `aws-secret-access-key` | no | | |
| `aws-security-group-id` | no | | Security group to open SSH on. **Leave empty to skip** |
| `aws-region` | no | `eu-north-1` | Region the security group lives in |

### `full-import`

With `full-import: true` (the default) the instance ends up mirroring the export file:
entities that are no longer in the file are **deleted**. This is what you want when the
repository is the source of truth — otherwise a template you delete in git lives on
forever on the server.

Set it to `false` to merge instead, which only creates and updates. `full-import` cannot
be combined with `target-folder`.

### Servers not behind a security group

Leave `aws-security-group-id` empty and the whitelist/revoke steps are skipped entirely,
along with the AWS credential configuration. Everything else works the same.

### Instances with authentication enabled

Set `jsreport-user` and `jsreport-password`. They are passed to `curl` over stdin rather
than on the command line, so they do not show up in the server's process list.

## Gotcha: duplicate entity ids

The import inserts by `_id` and rejects the whole file if two entities share one:

```
Import failed: Unable to insert an entity (assets) "/Folder/logo.png" during the import:
Error: Entity with _id "kebtD3AUTv1KH1R7" already exists.
```

The filesystem store keys entities by path, so it never notices, and `validate-import`
does not catch it either. It happens when an asset folder is copied **on disk** — the
`config.json` comes along, ids and all. Copying in the studio mints new ids.

Worth guarding in CI on the repository side, before it reaches a deploy.
