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



\## #1 — Readiness probe on the wrong port

\*\*Date:\*\* 2026-10-23



\*\*Symptom:\*\* Kubectl get pods showed the the pod is not ready



\*\*Diagnosis:\*\* Kubectl Describe Pod showed the issue as the readiness probe is not working



\*\*Fix:\*\* Fixed the port an then applied the file again. It started to run



\*\*Lesson:\*\* Readiness Probe should mactch the container





\## #2 — OOMKilled probe on the wrong port

\*\*Date:\*\* 2026-10-23



\*\*Symptom:\*\* The pod was restarting on intervals



\*\*Diagnosis:\*\* Ran kubectl Describe it showed the Reason as Crashloopbackoff or OOMKIlled



\*\*Fix:\*\* Updated the memory for the pod and then ran the apply command.



\*\*Lesson:\*\* Sufficient memory should be there for the pods.







