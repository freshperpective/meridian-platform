\# Incidents



Format: \*\*What I broke → Symptom → Diagnosis → Fix → Lesson\*\*



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

between branches. Detached HEAD means HEAD points at a commit, not a branch; get out with

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



\*\*What I broke:\*\* Set the memory limit to `32Mi` (request `16Mi`) and added

`--stress-memory=64`, so podinfo tries to allocate 64 MB at startup.



\*\*Symptom:\*\* `kubectl get pods -w` showed STATUS `OOMKilled`, then `CrashLoopBackOff`,

with RESTARTS climbing (3 within 40 seconds).



\*\*Diagnosis:\*\*

\- `kubectl describe pod` → `State: Waiting, Reason: CrashLoopBackOff` and

&#x20; `Last State: Terminated, Reason: OOMKilled, Exit Code: 137`.

\- `Started` and `Finished` were the same second: the container died the moment it

&#x20; allocated more than 32Mi.

\- Exit code 137 = 128 + 9: the process was killed by SIGKILL, sent by the kernel's

&#x20; OOM killer when the container's cgroup went over its memory limit.

\- OOMKilled vs CrashLoopBackOff: OOMKilled is the \*\*cause\*\* (why the container died, in

&#x20; Last State). CrashLoopBackOff is the \*\*symptom\*\* (the kubelet waiting longer and longer

&#x20; between restarts). Any repeated crash shows CrashLoopBackOff; Last State tells you why.

\- Why killed instead of slowed down: CPU is compressible, so a container over its CPU

&#x20; limit is throttled and just runs slower. Memory isn't compressible: the kernel can't

&#x20; take back memory a process is using, so it kills the process instead.



\*\*Fix:\*\* Restored the original manifest (`git restore --source f003818 podinfo.yaml`),

which removed the stress flag and set limits back to `512Mi` / requests `64Mi`, then

re-applied. Pods returned to `1/1 Running`.



\*\*Lesson:\*\* Memory limits are hard limits: exceed one and the kernel kills the process

(exit 137). Set limits from observed usage (`kubectl top pod`), and when a pod is

crash-looping, read \*\*Last State\*\*, not the current status, to find out why.

