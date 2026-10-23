\# Incidents



\## #0 — Detached HEAD while trying to discard a change

\*\*Date:\*\* 2026-10-23



\*\*Symptom:\*\* Tried to undo my edit to `podinfo.yaml`. `git checkout apps/podinfo.yaml`

failed with "pathspec did not match", and `git status` showed "HEAD detached at origin/main".



\*\*Diagnosis:\*\*

\- I ran the command from inside `apps/`, so git looked for `apps/apps/podinfo.yaml`.

&#x20; Paths in git commands are relative to the current folder.

\- `git checkout origin` moved HEAD onto the remote-tracking commit instead of my `main`

&#x20; branch → detached HEAD. Commits made there belong to no branch.

\- `git pull` didn't help: it fetches and merges remote changes, it never discards local edits.



\*\*Fix:\*\*

&#x20;   git switch main

&#x20;   git restore podinfo.yaml

&#x20;   kubectl apply -f podinfo.yaml



\*\*Lesson:\*\* `git restore <file>` discards local changes; `git switch <branch>` moves

between branches. Detached HEAD = HEAD points at a commit, not a branch; get out with

`git switch <branch>`.



\## #1 — Readiness probe pointed at the gRPC port

\*\*Date:\*\* 2026-10-23



\*\*What I broke:\*\* Changed the readiness probe from `localhost:9898/readyz` to

`localhost:9999/readyz` and applied.



\*\*Symptom:\*\* The new pod was `Running` but never Ready (`0/1`). `kubectl rollout status`

eventually failed: `deployment "podinfo" exceeded its progress deadline`.

The app kept answering on `curl localhost:9898`.



\*\*Diagnosis:\*\*

\- `kubectl describe pod` → `Readiness: exec \[podcli check http localhost:9999/readyz]`

&#x20; and Events: `Readiness probe failed: ... malformed HTTP response "\\x00\\x00\\x06\\x04..."`

\- `Ports:` showed 9999 is podinfo's \*\*gRPC\*\* port. The probe reached a server, but it

&#x20; speaks HTTP/2 (gRPC), so the HTTP/1.1 check got binary data back: "malformed response",

&#x20; not "connection refused".

\- Why curl kept working: a Service only routes to Ready pods, and a rolling update keeps

&#x20; old pods until new ones are Ready. Old pods kept serving; the rollout stalled until it

&#x20; hit the progress deadline.



\*\*Fix:\*\* Restored the probe to 9898 and re-applied. New pod became Ready, rollout completed.



\*\*Lesson:\*\* A broken readiness probe doesn't cause an outage. It stalls the rollout while

old pods keep serving. "Malformed HTTP response" means you hit the wrong protocol;

"connection refused" means nothing is listening. The exact error tells you which.





\## #2 — Container OOMKilled by a memory limit

\*\*Date:\*\* 2026-10-23



\*\*What I broke:\*\* Set memory limit to 32Mi and added `--stress-memory=64Mi`.



\*\*Symptom:\*\* RESTARTS column went CrashLoopBackOff; status showed OOMKilled.



\*\*Diagnosis:\*\*

\- `kubectl describe pod` → Last State: Terminated, Reason: OOMKilled, Exit Code: \_\_\_\_

\- OOMKilled vs CrashLoopBackOff: OOMKilled is cause; CrashLoopBackOff is symptom.

\- Why the container was killed instead of slowed down: \_\_\_\_



\*\*Fix:\*\* Fixed the memory by restoring the podinfo file and then applied it



\*\*Lesson:\*\* The container should have sufficient memory for it to run



