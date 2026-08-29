# CI pipeline smoke test

This file exists only to exercise the `Build & Push` workflow end to end:
a change inside `spring-petclinic-vets-service/` should make `dorny/paths-filter`
select `vets-service` for the build matrix, and nothing else.

Expected pipeline effect:
1. `changes` job → `services = ["vets-service"]`
2. `build (vets-service)` → arm64 image, Trivy CRITICAL gate, push
   `…/petclinic-dev/vets-service:<7-char-sha>` to ECR
3. `dispatch` → `repository_dispatch app-image-built` to the platform repo
   with `{ sha: <7-char-sha>, services: "vets-service" }`
4. platform repo `update-image-tags.yml` → `yq` sets `image.tag: <sha>` in all 8
   `helm-values/*.yaml`, commits `ci: update image tags to <sha> (vets-service)`

Safe to delete once the pipeline is verified working.

<!-- trigger: 2026-08-30 (run 3 — after dropping the AWS_ACCOUNT_ID/AWS_REGION secret dependency) -->
