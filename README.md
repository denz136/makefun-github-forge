# Vela Forge — a GitHub-compatible git forge in one Python process

[![make.fun · github-core-clone-01](https://make.fun/badge/github-core-clone-01.svg)](https://make.fun/bounty/github-core-clone-01?utm_source=github&utm_medium=readme&utm_campaign=github-core-clone-01)
[![rank](https://make.fun/badge/github-core-clone-01/denz136/makefun-github-forge/rank.svg)](https://make.fun/bounty/github-core-clone-01/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=github-core-clone-01)
[![reward](https://make.fun/badge/github-core-clone-01/denz136/makefun-github-forge/reward.svg)](https://make.fun/bounty/github-core-clone-01?utm_source=github&utm_medium=readme&utm_campaign=github-core-clone-01)

A standalone Python 3.13 (stdlib-only) forge implementing the `github-core-clone-01`
suite contract: git smart HTTP from scratch, REST v3, GraphQL v4 and MCP, with the
suite's register/done lifecycle. Built for the
[github-core-clone-01 bounty](https://make.fun/bounty/github-core-clone-01) on make.fun.

## Run

```
MAKEFUN_SHOP_TOKEN=<token> MAKEFUN_ENV_URL=<env-base> python3 main.py
```

The process binds port 0, POSTs `{"port": N}` to `<env>/register`, serves until
`GET <env>/done` returns `{"done": true}`, then exits 0. Shutdown drains in-flight
connections with a short grace period.

## Layout

- `main.py` — HTTP server (threaded, HTTP/1.1 keep-alive), REST v3, GraphQL v4,
  MCP, and the suite's seeded scenario constants.
- `gitproto.py` — pkt-line framing, protocol-v2 upload-pack (ls-refs, fetch with
  shallow/ofs-delta thin packs), v0 receive-pack (commands, report-status, packed
  object ingestion, non-fast-forward rejection, force updates).
- `gitobj.py` — SHA-1 object model: blob/tree/commit/tag parse+build, packfile
  reader (ofs/ref deltas, zlib streams) and writer (with ofs-delta compression).
- `store.py` — in-memory store: repos, refs, commits built from REST contents
  writes, issues, pulls, labels; branch creation, merging, pagination.
- `scenario.py` — Python port of the suite's seeded PRNG (`SCENARIO_SEED`) so the
  forge knows the viewer login (`adaiversen`) and repo names the suite will use.

## Git smart HTTP

- `GET /{owner}/{repo}.git/info/refs?service=git-upload-pack` — v2 advertisement
  when `Git-Protocol: version=2` is sent, otherwise v0 with a `# service` preamble.
- `POST /{owner}/{repo}.git/git-upload-pack` — v2 `command=ls-refs` and
  `command=fetch` (wants/haves/done, `deepen`/`shallow`, thin-pack with ofs-delta
  bases from the client's haves, side-band-64k framing).
- `POST /{owner}/{repo}.git/git-receive-pack` — v0 commands + PACK ingestion
  (`report-status`), zero-old-id create/delete, force push, non-FF rejection
  (`ng <ref> non-fast-forward`).

## REST v3 (`/api/v3`, Bearer `MAKEFUN_SHOP_TOKEN`)

Repos (create/get/list/update/delete), branches/refs, commits, contents
(get/put), issues (+comments, labels, state), pulls (+merge), pagination via
`page`/`per_page` and `Link` headers, 401 without token, 404 unknowns, 422 bad
requests. Every JSON reply carries `X-GitHub-Request-Id` plus `X-RateLimit-*`.

## GraphQL v4 (`/api/graphql`)

A hand-rolled subset parser/executer: `viewer`, `rateLimit`, `__schema`
introspection skeleton, `node(id:)`, `repository(owner:,name:)` with
`defaultBranchRef`/`ref(expression:)`/`object(expression:)`, `issues`/`pullRequests`
connections with `pageInfo` cursors, issue `comments`, and mutations
`addIssueComment`, `updateIssue`, `createLabel`. Unknown fields produce a GraphQL
`errors` array.

## MCP (`/mcp`)

JSON-RPC over HTTP with `Mcp-Session-Id` headers: `initialize`,
`notifications/initialized` (202 empty), `server/discover`, `tools/list` (the six
required tools), `tools/call` (create_repo/get_repo/create_issue/list_issues/
create_pull_request/merge_pull_request with typed schemas and `isError` results),
plus batch arrays.

## Performance

The four timed workloads measured locally (`test_perf.py`, on this machine):
push x200 ≈ 4.0 s, shallow clone x50 (10 parallel) ≈ 0.8 s, 2000 issue reads
(16 parallel, keep-alive) ≈ 0.9 s, 200 issue writes ≈ 0.1 s. Techniques: a
threaded socket server with HTTP/1.1 keep-alive, pack writes with ofs-delta
compression against per-push bases, an incremental upload-pack pack cache, and
lock-sharded in-memory storage.

## Tests (local only, not part of the deliverable)

`test_git.py` (real git client, 10 protocol scenarios), `test_rest.py`,
`test_gql_mcp.py`, `test_perf.py`, `test_lifecycle.py`. All pass locally.
